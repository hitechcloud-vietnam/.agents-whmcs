# WHMCS Contract Management DevKit

## Overview

A comprehensive contract lifecycle management system for WHMCS for creating, managing, and tracking customer contracts from initiation through termination, including contract templates, workflow approvals, renewal management, and compliance tracking.

## Features

- Contract templates
- Contract lifecycle stages
- Digital signatures
- Renewal management
- Amendment tracking
- Compliance checks
- Delivery tracking
- Document management
- Workflow approvals
- Contract analytics

## Database Schema

```sql
CREATE TABLE IF NOT EXISTS `mod_contract_templates` (
    `id` INT UNSIGNED NOT NULL AUTO_INCREMENT,
    `template_name` VARCHAR(255) NOT NULL,
    `template_code` VARCHAR(50) NOT NULL,
    `contract_type` VARCHAR(100) NOT NULL,
    `content` LONGTEXT NOT NULL,
    `variables` JSON NULL,
    `is_active` TINYINT(1) NOT NULL DEFAULT 1,
    `created_at` DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
    PRIMARY KEY (`id`),
    UNIQUE KEY `uk_template_code` (`template_code`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;

CREATE TABLE IF NOT EXISTS `mod_contracts` (
    `id` INT UNSIGNED NOT NULL AUTO_INCREMENT,
    `contract_number` VARCHAR(50) NOT NULL,
    `contract_name` VARCHAR(255) NOT NULL,
    `user_id` INT UNSIGNED NOT NULL,
    `service_id` INT UNSIGNED NULL,
    `template_id` INT UNSIGNED NULL,
    `contract_type` VARCHAR(100) NOT NULL,
    `start_date` DATE NOT NULL,
    `end_date` DATE NOT NULL,
    `value` DECIMAL(12,2) NOT NULL DEFAULT 0.00,
    `currency` VARCHAR(3) NOT NULL DEFAULT 'USD',
    `payment_terms` VARCHAR(50) NULL,
    `payment_amount` DECIMAL(12,2) NULL,
    `status` ENUM('draft', 'pending_approval', 'pending_signature', 'active', 'renewed', 'expired', 'terminated', 'cancelled') NOT NULL DEFAULT 'draft',
    `auto_renew` TINYINT(1) NOT NULL DEFAULT 0,
    `renewal_notice_days` INT UNSIGNED NOT NULL DEFAULT 30,
    `current_version` VARCHAR(20) NOT NULL DEFAULT '1.0.0',
    `parent_contract_id` INT UNSIGNED NULL,
    `approved_by` INT UNSIGNED NULL,
    `approved_at` DATETIME NULL,
    `signed_at` DATETIME NULL,
    `signed_document_url` VARCHAR(500) NULL,
    `created_by` INT UNSIGNED NOT NULL,
    `created_at` DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
    `updated_at` DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    PRIMARY KEY (`id`),
    UNIQUE KEY `uk_contract_number` (`contract_number`),
    INDEX `idx_user_status` (`user_id`, `status`),
    INDEX `idx_end_date` (`end_date`),
    CONSTRAINT `fk_contract_user` FOREIGN KEY (`user_id`) REFERENCES `tblusers`(`id`) ON DELETE CASCADE
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;

CREATE TABLE IF NOT EXISTS `mod_contract_versions` (
    `id` INT UNSIGNED NOT NULL AUTO_INCREMENT,
    `contract_id` INT UNSIGNED NOT NULL,
    `version` VARCHAR(20) NOT NULL,
    `content` LONGTEXT NOT NULL,
    `change_summary` TEXT NULL,
    `amendment_type` ENUM('addendum', 'modification', 'renewal', 'termination') NULL,
    `created_by` INT UNSIGNED NOT NULL,
    `created_at` DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
    PRIMARY KEY (`id`),
    UNIQUE KEY `uk_contract_version` (`contract_id`, `version`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;

CREATE TABLE IF NOT EXISTS `mod_contract_amendments` (
    `id` INT UNSIGNED NOT NULL AUTO_INCREMENT,
    `contract_id` INT UNSIGNED NOT NULL,
    `amendment_number` VARCHAR(50) NOT NULL,
    `amendment_type` ENUM('scope_change', 'pricing_change', 'term_extension', 'termination', 'other') NOT NULL,
    `description` TEXT NOT NULL,
    `old_value` TEXT NULL,
    `new_value` TEXT NULL,
    `effective_date` DATE NOT NULL,
    `status` ENUM('pending', 'approved', 'rejected', 'signed', 'applied') NOT NULL DEFAULT 'pending',
    `requested_by` INT UNSIGNED NOT NULL,
    `approved_by` INT UNSIGNED NULL,
    `signed_at` DATETIME NULL,
    `created_at` DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
    PRIMARY KEY (`id`),
    CONSTRAINT `fk_amendment_contract` FOREIGN KEY (`contract_id`) REFERENCES `mod_contracts`(`id`) ON DELETE CASCADE
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
```

## Module Class

```php
<?php
/**
 * WHMCS Contract Management Module
 */

if (!defined("WHMCS")) {
    die("This file cannot be accessed directly");
}

require_once __DIR__ . '/lib/ContractManager.php';
require_once __DIR__ . '/lib/ContractWorkflow.php';
require_once __DIR__ . '/lib/ContractRenewals.php';

function whmcs_contract_management_activate() {
    $manager = new ContractManager();
    return $manager->activate();
}

function whmcs_contract_management_deactivate() {
    return ['success' => true, 'msg' => 'Contract Management module deactivated'];
}

function whmcs_contract_management_config() {
    return [
        'require_contract_approval' => [
            'FriendlyName' => 'Require Approval Before Activation',
            'Type' => 'yesno',
            'Default' => true,
        ],
        'default_auto_renew' => [
            'FriendlyName' => 'Auto-Renew by Default',
            'Type' => 'yesno',
        ],
        'renewal_notice_days' => [
            'FriendlyName' => 'Renewal Notice Days',
            'Type' => 'text',
            'Default' => '30',
        ],
    ];
}

function whmcs_contract_management_create_contract($data) {
    $manager = new ContractManager();
    return $manager->createContract($data);
}

function whmcs_contract_management_get_contracts($filters = []) {
    $manager = new ContractManager();
    return $manager->getContracts($filters);
}

function whmcs_contract_management_sign_contract($contractId, $userId) {
    $manager = new ContractManager();
    return $manager->signContract($contractId, $userId);
}

function whmcs_contract_management_renew_contract($contractId, $newEndDate) {
    $manager = new ContractRenewals();
    return $manager->renewContract($contractId, $newEndDate);
}

add_hook('DailyCronJob', 1, function() {
    $manager = new ContractRenewals();
    $manager->processRenewals();
    $manager->processExpirations();
    $manager->sendRenewalNotices();
});
```

### lib/ContractManager.php

```php
<?php
namespace WHMCS\Module\ContractManagement;

use Illuminate\Database\Capsule\Manager as Capsule;
use Carbon\Carbon;

class ContractManager {
    
    public function activate() {
        try {
            $this->createTables();
            $this->createDefaultTemplates();
            return ['success' => true, 'msg' => 'Contract Management module activated'];
        } catch (\Exception $e) {
            return ['success' => false, 'msg' => $e->getMessage()];
        }
    }
    
    protected function createTables() {
        Capsule::statement("
            CREATE TABLE IF NOT EXISTS `mod_contract_templates` (
                `id` INT UNSIGNED NOT NULL AUTO_INCREMENT,
                `template_name` VARCHAR(255) NOT NULL UNIQUE,
                `template_code` VARCHAR(50) NOT NULL,
                `contract_type` VARCHAR(100) NOT NULL,
                `content` LONGTEXT NOT NULL,
                `is_active` TINYINT(1) NOT NULL DEFAULT 1,
                PRIMARY KEY (`id`)
            ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4
        ");
        
        Capsule::statement("
            CREATE TABLE IF NOT EXISTS `mod_contracts` (
                `id` INT UNSIGNED NOT NULL AUTO_INCREMENT,
                `contract_number` VARCHAR(50) NOT NULL UNIQUE,
                `contract_name` VARCHAR(255) NOT NULL,
                `user_id` INT UNSIGNED NOT NULL,
                `service_id` INT UNSIGNED NULL,
                `template_id` INT UNSIGNED NULL,
                `contract_type` VARCHAR(100) NOT NULL,
                `start_date` DATE NOT NULL,
                `end_date` DATE NOT NULL,
                `value` DECIMAL(12,2) NOT NULL DEFAULT  0.00,
                `status` VARCHAR(20) NOT NULL DEFAULT 'draft',
                `auto_renew` TINYINT(1) NOT NULL DEFAULT 0,
                `current_version` VARCHAR(20) NOT NULL DEFAULT '1.0.0',
                `created_at` DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
                PRIMARY KEY (`id`),
                INDEX `idx_user_status` (`user_id`, `status`)
            ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4
        ");
        
        Capsule::statement("
            CREATE TABLE IF NOT EXISTS `mod_contract_versions` (
                `id` INT UNSIGNED NOT NULL AUTO_INCREMENT,
                `contract_id` INT UNSIGNED NOT NULL,
                `version` VARCHAR(20) NOT NULL,
                `content` LONGTEXT NOT NULL,
                `change_summary` TEXT NULL,
                `created_at` DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
                PRIMARY KEY (`id`)
            ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4
        ");
        
        Capsule::statement("
            CREATE TABLE IF NOT EXISTS `mod_contract_amendments` (
                `id` INT UNSIGNED NOT NULL AUTO_INCREMENT,
                `contract_id` INT UNSIGNED NOT NULL,
                `amendment_number` VARCHAR(50) NOT NULL,
                `amendment_type` VARCHAR(50) NOT NULL,
                `description` TEXT NOT NULL,
                `effective_date` DATE NOT NULL,
                `status` VARCHAR(20) NOT NULL DEFAULT 'pending',
                `created_at` DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
                PRIMARY KEY (`id`)
            ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4
        ");
    }
    
    protected function createDefaultTemplates() {
        $templates = [
            [
                'name' => 'Standard Service Agreement',
                'code' => 'SSA-001',
                'type' => 'service',
                'content' => 'This Service Agreement ("Agreement") is entered into...',
            ],
            [
                'name' => 'Cloud Services Agreement',
                'code' => 'CSA-001',
                'type' => 'cloud',
                'content' => 'This Cloud Services Agreement governs the use of...',
            ],
            [
                'name' => 'Professional Services Agreement',
                'code' => 'PSA-001',
                'type' => 'professional',
                'content' => 'This Professional Services Agreement outlines...',
            ],
        ];
        
        foreach ($templates as $template) {
            if (!Capsule::table('mod_contract_templates')->where('template_code', $template['code'])->exists()) {
                Capsule::table('mod_contract_templates')->insert($template);
            }
        }
    }
    
    protected function generateContractNumber() {
        return 'CNT-' . date('Y') . '-' . str_pad(Capsule::table('mod_contracts')->count() + 1, 6, '0', STR_PAD_LEFT);
    }
    
    public function createContract($data) {
        $contractNumber = $this->generateContractNumber();
        
        $contractId = Capsule::table('mod_contracts')->insertGetId([
            'contract_number' => $contractNumber,
            'contract_name' => $data['name'],
            'user_id' => $data['user_id'],
            'service_id' => $data['service_id'] ?? null,
            'template_id' => $data['template_id'] ?? null,
            'contract_type' => $data['type'] ?? 'service',
            'start_date' => $data['start_date'],
            'end_date' => $data['end_date'],
            'value' => $data['value'] ?? 0,
            'auto_renew' => $data['auto_renew'] ?? 0,
            'status' => 'draft',
            'created_at' => Carbon::now(),
        ]);
        
        return [
            'success' => true,
            'contract_id' => $contractId,
            'contract_number' => $contractNumber,
        ];
    }
    
    public function getContracts($filters = []) {
        $query = Capsule::table('mod_contracts as c')
            ->join('tblusers as u', 'c.user_id', '=', 'u.id');
        
        if (!empty($filters['user_id'])) {
            $query->where('c.user_id', $filters['user_id']);
        }
        if (!empty($filters['status'])) {
            $query->where('c.status', $filters['status']);
        }
        if (!empty($filters['expiring_soon'])) {
            $query->where('c.end_date', '<=', Carbon::now()->addDays(30)->toDateString());
        }
        
        return $query->select('c.*', 'u.email', 'u.firstname', 'u.lastname')
            ->orderBy('c.created_at', 'desc')
            ->get();
    }
    
    public function signContract($contractId, $userId) {
        Capsule::table('mod_contracts')
            ->where('id', $contractId)
            ->update([
                'status' => 'active',
                'signed_at' => Carbon::now(),
            ]);
        
        return ['success' => true, 'msg' => 'Contract signed and activated'];
    }
    
    public function terminateContract($contractId, $reason = null) {
        Capsule::table('mod_contracts')
            ->where('id', $contractId)
            ->update([
                'status' => 'terminated',
                'end_date' => Carbon::today(),
            ]);
        
        return ['success' => true];
    }
    
    public function createAmendment($contractId, $data) {
        $amendmentNumber = 'AMD-' . date('Y') . '-' . str_pad(
            Capsule::table('mod_contract_amendments')->count() + 1, 6, '0', STR_PAD_LEFT
        );
        
        $amendmentId = Capsule::table('mod_contract_amendments')->insertGetId([
            'contract_id' => $contractId,
            'amendment_number' => $amendmentNumber,
            'amendment_type' => $data['type'],
            'description' => $data['description'],
            'effective_date' => $data['effective_date'],
            'status' => 'pending',
        ]);
        
        return ['success' => true, 'amendment_id' => $amendmentId];
    }
}
```

### lib/ContractWorkflow.php

```php
<?php
namespace WHMCS\Module\ContractManagement;

use Illuminate\Database\Capsule\Manager as Capsule;
use Carbon\Carbon;

class ContractWorkflow {
    
    public function approveContract($contractId, $adminId) {
        Capsule::table('mod_contracts')
            ->where('id', $contractId)
            ->update([
                'status' => 'pending_signature',
                'approved_by' => $adminId,
                'approved_at' => Carbon::now(),
            ]);
        
        return ['success' => true];
    }
    
    public function rejectContract($contractId, $adminId, $reason) {
        Capsule::table('mod_contracts')
            ->where('id', $contractId)
            ->update([
                'status' => 'draft',
                'rejection_reason' => $reason,
            ]);
        
        return ['success' => true];
    }
    
    public function getPendingApprovals($adminId = null) {
        $query = Capsule::table('mod_contracts as c')
            ->join('tblusers as u', 'c.user_id', '=', 'u.id')
            ->where('c.status', 'pending_approval');
        
        if ($adminId) {
            $query->where('c.assigned_approver', $adminId);
        }
        
        return $query->select('c.*', 'u.email', 'u.firstname', 'u.lastname')
            ->get();
    }
    
    public function approveAmendment($amendmentId, $adminId) {
        Capsule::table('mod_contract_amendments')
            ->where('id', $amendmentId)
            ->update([
                'status' => 'approved',
                'approved_by' => $adminId,
                'approved_at' => Carbon::now(),
            ]);
        
        return ['success' => true];
    }
}
```

### lib/ContractRenewals.php

```php
<?php
namespace WHMCS\Module\ContractManagement;

use Illuminate\Database\Capsule\Manager as Capsule;
use Carbon\Carbon;

class ContractRenewals {
    
    public function renewContract($contractId, $newEndDate) {
        $contract = Capsule::table('mod_contracts')->where('id', $contractId)->first();
        
        if (!$contract) {
            return ['success' => false, 'msg' => 'Contract not found'];
        }
        
        $newVersion = $this->incrementVersion($contract->current_version);
        
        Capsule::table('mod_contracts')
            ->where('id', $contractId)
            ->update([
                'start_date' => Carbon::parse($contract->end_date)->addDay()->toDateString(),
                'end_date' => $newEndDate,
                'current_version' => $newVersion,
                'status' => 'renewed',
            ]);
        
        return ['success' => true, 'new_version' => $newVersion];
    }
    
    protected function incrementVersion($version) {
        $parts = explode('.', $version);
        $parts[1] = isset($parts[1]) ? $parts[1] + 1 : 1;
        return implode('.', $parts);
    }
    
    public function processRenewals() {
        $contracts = Capsule::table('mod_contracts')
            ->where('status', 'active')
            ->where('auto_renew', 1)
            ->where('end_date', '<=', Carbon::now()->toDateString())
            ->get();
        
        foreach ($contracts as $contract) {
            $newEndDate = Carbon::parse($contract->end_date)->addYear()->toDateString();
            $this->renewContract($contract->id, $newEndDate);
        }
    }
    
    public function processExpirations() {
        Capsule::table('mod_contracts')
            ->where('status', 'active')
            ->where('auto_renew', 0)
            ->where('end_date', '<', Carbon::today()->toDateString())
            ->update(['status' => 'expired']);
    }
    
    public function sendRenewalNotices() {
        $noticeDays = 30;
        $contracts = Capsule::table('mod_contracts as c')
            ->join('tblusers as u', 'c.user_id', '=', 'u.id')
            ->where('c.status', 'active')
            ->where('c.end_date', Carbon::now()->addDays($noticeDays)->toDateString())
            ->get();
        
        foreach ($contracts as $contract) {
            sendEmail('contract_renewal_notice', $contract->email, [
                'contract_number' => $contract->contract_number,
                'end_date' => $contract->end_date,
            ]);
        }
    }
}
```

## API Endpoints

```
GET  /api/v1/contracts                    - List contracts
GET  /api/v1/contracts/{id}               - Get contract details
POST /api/v1/contracts                    - Create contract
PUT  /api/v1/contracts/{id}               - Update contract
POST /api/v1/contracts/{id}/sign         - Sign contract
POST /api/v1/contracts/{id}/terminate    - Terminate contract
GET  /api/v1/contracts/expiring           - Get expiring contracts
GET  /api/v1/contracts/pending-approval  - Get pending approvals
POST /api/v1/contracts/{id}/approve      - Approve contract
POST /api/v1/contracts/{id}/amend        - Create amendment
GET  /api/v1/contract-templates          - List templates
```
