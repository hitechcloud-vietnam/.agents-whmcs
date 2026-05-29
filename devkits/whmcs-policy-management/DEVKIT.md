# WHMCS Policy Management DevKit

## Overview

A comprehensive policy management system for WHMCS that allows creating, managing, and enforcing organizational policies, tracking policy acceptance, managing policy versions, and automating policy compliance reminders.

## Features

- Policy creation and versioning
- Policy categories
- Acceptance tracking
- Compliance reminders
- Document attachments
- Change history
- Approval workflow
- Policy acknowledgment
- Training integration
- Reporting

## Database Schema

```sql
CREATE TABLE IF NOT EXISTS `mod_policy_categories` (
    `id` INT UNSIGNED NOT NULL AUTO_INCREMENT,
    `category_name` VARCHAR(255) NOT NULL,
    `category_code` VARCHAR(50) NOT NULL,
    `description` TEXT NULL,
    `is_active` TINYINT(1) NOT NULL DEFAULT 1,
    `sort_order` INT UNSIGNED NOT NULL DEFAULT 0,
    PRIMARY KEY (`id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;

CREATE TABLE IF NOT EXISTS `mod_policies` (
    `id` INT UNSIGNED NOT NULL AUTO_INCREMENT,
    `policy_code` VARCHAR(50) NOT NULL,
    `policy_name` VARCHAR(255) NOT NULL,
    `category_id` INT UNSIGNED NOT NULL,
    `version` VARCHAR(20) NOT NULL DEFAULT '1.0.0',
    `content` LONGTEXT NULL,
    `summary` TEXT NULL,
    `effective_date` DATE NOT NULL,
    `expiration_date` DATE NULL,
    `compliance_framework` VARCHAR(100) NULL,
    `is_mandatory` TINYINT(1) NOT NULL DEFAULT 0,
    `requires_acknowledgment` TINYINT(1) NOT NULL DEFAULT 1,
    `acknowledgment_frequency` ENUM('once', 'annual', 'quarterly', 'monthly') NOT NULL DEFAULT 'once',
    `status` ENUM('draft', 'pending_review', 'approved', 'published', 'archived') NOT NULL DEFAULT 'draft',
    `created_by` INT UNSIGNED NOT NULL,
    `approved_by` INT UNSIGNED NULL,
    `created_at` DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
    `updated_at` DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    PRIMARY KEY (`id`),
    UNIQUE KEY `uk_policy_code_version` (`policy_code`, `version`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;

CREATE TABLE IF NOT EXISTS `mod_policy_acknowledgments` (
    `id` INT UNSIGNED NOT NULL AUTO_INCREMENT,
    `policy_id` INT UNSIGNED NOT NULL,
    `user_id` INT UNSIGNED NOT NULL,
    `acknowledged_at` DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
    `acknowledgment_method` ENUM('web', 'email', 'signature', 'api') NOT NULL DEFAULT 'web',
    `ip_address` VARCHAR(45) NULL,
    `user_agent` VARCHAR(500) NULL,
    `notes` TEXT NULL,
    PRIMARY KEY (`id`),
    UNIQUE KEY `uk_policy_user` (`policy_id`, `user_id`),
    CONSTRAINT `fk_ack_policy` FOREIGN KEY (`policy_id`) REFERENCES `mod_policies`(`id`) ON DELETE CASCADE,
    CONSTRAINT `fk_ack_user` FOREIGN KEY (`user_id`) REFERENCES `tblusers`(`id`) ON DELETE CASCADE
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;

CREATE TABLE IF NOT EXISTS `mod_policy_versions` (
    `id` INT UNSIGNED NOT NULL AUTO_INCREMENT,
    `policy_id` INT UNSIGNED NOT NULL,
    `version` VARCHAR(20) NOT NULL,
    `content` LONGTEXT NULL,
    `change_summary` TEXT NULL,
    `created_by` INT UNSIGNED NOT NULL,
    `created_at` DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
    PRIMARY KEY (`id`),
    UNIQUE KEY `uk_policy_version` (`policy_id`, `version`),
    CONSTRAINT `fk_version_policy` FOREIGN KEY (`policy_id`) REFERENCES `mod_policies`(`id`) ON DELETE CASCADE
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
```

## Module Class

```php
<?php
/**
 * WHMCS Policy Management Module
 */

if (!defined("WHMCS")) {
    die("This file cannot be accessed directly");
}

require_once __DIR__ . '/lib/PolicyManager.php';
require_once __DIR__ . '/lib/PolicyAcknowledgment.php';

use WHMCS\Module\PolicyManagement\PolicyManager;
use WHMCS\Module\PolicyManagement\PolicyAcknowledgment;

function whmcs_policy_management_activate() {
    $manager = new PolicyManager();
    return $manager->activate();
}

function whmcs_policy_management_deactivate() {
    return ['success' => true, 'msg' => 'Policy Management module deactivated'];
}

function whmcs_policy_management_config() {
    return [
        'auto_reminders' => [
            'FriendlyName' => 'Enable Auto Reminders',
            'Type' => 'yesno',
            'Default' => true,
        ],
        'reminder_days_before' => [
            'FriendlyName' => 'Reminder Days Before Expiry',
            'Type' => 'text',
            'Default' => '7',
        ],
        'require_all_policies' => [
            'FriendlyName' => 'Require All Policies Before Order',
            'Type' => 'yesno',
        ],
    ];
}

function whmcs_policy_management_create_policy($data) {
    $manager = new PolicyManager();
    return $manager->createPolicy($data);
}

function whmcs_policy_management_get_policies($filters = []) {
    $manager = new PolicyManager();
    return $manager->getPolicies($filters);
}

function whmcs_policy_management_acknowledge($policyId, $userId) {
    $ack = new PolicyAcknowledgment();
    return $ack->acknowledge($policyId, $userId);
}

function whmcs_policy_management_check_compliance($userId) {
    $ack = new PolicyAcknowledgment();
    return $ack->checkCompliance($userId);
}

add_hook('DailyCronJob', 1, function() {
    $manager = new PolicyManager();
    $manager->sendReminders();
    $manager->updateExpiredPolicies();
});

add_hook('UserRegistration', 1, function($params) {
    if (\App::get_config('policy_management')['require_all_policies']) {
        $ack = new \WHMCS\Module\PolicyManagement\PolicyAcknowledgment();
        $pending = $ack->getPendingPolicies($params['user_id']);
        if (!empty($pending)) {
            return ['error' => 'Please acknowledge required policies to continue.'];
        }
    }
});
```

### lib/PolicyManager.php

```php
<?php
namespace WHMCS\Module\PolicyManagement;

use Illuminate\Database\Capsule\Manager as Capsule;
use Carbon\Carbon;

class PolicyManager {
    
    public function activate() {
        try {
            $this->createTables();
            $this->initializeCategories();
            return ['success' => true, 'msg' => 'Policy Management module activated'];
        } catch (\Exception $e) {
            return ['success' => false, 'msg' => $e->getMessage()];
        }
    }
    
    protected function createTables() {
        Capsule::statement("
            CREATE TABLE IF NOT EXISTS `mod_policy_categories` (
                `id` INT UNSIGNED NOT NULL AUTO_INCREMENT,
                `category_name` VARCHAR(255) NOT NULL UNIQUE,
                `description` TEXT NULL,
                `is_active` TINYINT(1) NOT NULL DEFAULT 1,
                `sort_order` INT UNSIGNED NOT NULL DEFAULT 0,
                PRIMARY KEY (`id`)
            ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4
        ");
        
        Capsule::statement("
            CREATE TABLE IF NOT EXISTS `mod_policies` (
                `id` INT UNSIGNED NOT NULL AUTO_INCREMENT,
                `policy_code` VARCHAR(50) NOT NULL,
                `policy_name` VARCHAR(255) NOT NULL,
                `category_id` INT UNSIGNED NOT NULL,
                `version` VARCHAR(20) NOT NULL DEFAULT '1.0.0',
                `content` LONGTEXT NULL,
                `summary` TEXT NULL,
                `effective_date` DATE NOT NULL,
                `expiration_date` DATE NULL,
                `is_mandatory` TINYINT(1) NOT NULL DEFAULT 0,
                `requires_acknowledgment` TINYINT(1) NOT NULL DEFAULT 1,
                `acknowledgment_frequency` VARCHAR(20) NOT NULL DEFAULT 'once',
                `status` VARCHAR(20) NOT NULL DEFAULT 'draft',
                `created_by` INT UNSIGNED NOT NULL,
                `created_at` DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
                PRIMARY KEY (`id`),
                UNIQUE KEY `uk_policy_code_version` (`policy_code`, `version`)
            ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4
        ");
        
        Capsule::statement("
            CREATE TABLE IF NOT EXISTS `mod_policy_acknowledgments` (
                `id` INT UNSIGNED NOT NULL AUTO_INCREMENT,
                `policy_id` INT UNSIGNED NOT NULL,
                `user_id` INT UNSIGNED NOT NULL,
                `acknowledged_at` DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
                `acknowledgment_method` VARCHAR(20) NOT NULL DEFAULT 'web',
                PRIMARY KEY (`id`),
                UNIQUE KEY `uk_policy_user` (`policy_id`, `user_id`)
            ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4
        ");
        
        Capsule::statement("
            CREATE TABLE IF NOT EXISTS `mod_policy_versions` (
                `id` INT UNSIGNED NOT NULL AUTO_INCREMENT,
                `policy_id` INT UNSIGNED NOT NULL,
                `version` VARCHAR(20) NOT NULL,
                `content` LONGTEXT NULL,
                `change_summary` TEXT NULL,
                `created_by` INT UNSIGNED NOT NULL,
                `created_at` DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
                PRIMARY KEY (`id`)
            ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4
        ");
    }
    
    protected function initializeCategories() {
        $categories = [
            ['name' => 'Security Policies', 'sort_order' => 1],
            ['name' => 'Privacy Policies', 'sort_order' => 2],
            ['name' => 'Usage Policies', 'sort_order' => 3],
            ['name' => 'Compliance Policies', 'sort_order' => 4],
            ['name' => 'Service Policies', 'sort_order' => 5],
        ];
        
        foreach ($categories as $cat) {
            Capsule::table('mod_policy_categories')->insert($cat);
        }
    }
    
    public function createPolicy($data) {
        $policyId = Capsule::table('mod_policies')->insertGetId([
            'policy_code' => $data['code'],
            'policy_name' => $data['name'],
            'category_id' => $data['category_id'],
            'content' => $data['content'] ?? '',
            'summary' => $data['summary'] ?? '',
            'effective_date' => $data['effective_date'] ?? Carbon::today(),
            'is_mandatory' => $data['mandatory'] ?? 0,
            'created_by' => $data['admin_id'] ?? 0,
            'status' => 'draft',
        ]);
        
        return ['success' => true, 'policy_id' => $policyId];
    }
    
    public function getPolicies($filters = []) {
        $query = Capsule::table('mod_policies as p')
            ->join('mod_policy_categories as c', 'p.category_id', '=', 'c.id');
        
        if (!empty($filters['category_id'])) {
            $query->where('p.category_id', $filters['category_id']);
        }
        if (!empty($filters['status'])) {
            $query->where('p.status', $filters['status']);
        }
        if (!empty($filters['is_mandatory'])) {
            $query->where('p.is_mandatory', 1);
        }
        
        if (!empty($filters['user_id'])) {
            $acknowledged = Capsule::table('mod_policy_acknowledgments')
                ->where('user_id', $filters['user_id'])
                ->pluck('policy_id')
                ->toArray();
            
            $query->addSelect(
                Capsule::raw('CASE WHEN p.id IN (' . implode(',', array_merge($acknowledged, [0])) . ') THEN 1 ELSE 0 END as acknowledged')
            );
        }
        
        return $query->select('p.*', 'c.category_name')
            ->orderBy('p.is_mandatory', 'desc')
            ->orderBy('p.policy_name')
            ->get();
    }
    
    public function publishPolicy($policyId) {
        Capsule::table('mod_policies')
            ->where('id', $policyId)
            ->update([
                'status' => 'published',
                'effective_date' => Carbon::today(),
            ]);
        
        return ['success' => true];
    }
    
    public function updatePolicy($policyId, $data) {
        $policy = Capsule::table('mod_policies')->where('id', $policyId)->first();
        
        $updateData = ['updated_at' => Carbon::now()];
        foreach (['policy_name', 'content', 'summary'] as $field) {
            if (isset($data[$field])) {
                $updateData[$field] = $data[$field];
            }
        }
        
        Capsule::table('mod_policies')->where('id', $policyId)->update($updateData);
        
        return ['success' => true];
    }
    
    public function sendReminders() {
        $pendingPolicies = Capsule::table('mod_policies as p')
            ->join('mod_policy_categories as c', 'p.category_id', '=', 'c.id')
            ->where('p.status', 'published')
            ->where('p.is_mandatory', 1)
            ->get();
        
        foreach ($pendingPolicies as $policy) {
            $unacknowledgedUsers = Capsule::table('tblusers as u')
                ->leftJoin('mod_policy_acknowledgments as a', function($join) use ($policy) {
                    $join->on('u.id', '=', 'a.user_id')
                        ->where('a.policy_id', $policy->id);
                })
                ->whereNull('a.id')
                ->pluck('u.id')
                ->toArray();
            
            foreach ($unacknowledgedUsers as $userId) {
                $this->sendReminderEmail($userId, $policy);
            }
        }
    }
    
    protected function sendReminderEmail($userId, $policy) {
        $user = Capsule::table('tblusers')->where('id', $userId)->first();
        
        if ($user && $user->email) {
            sendEmail('policy_reminder', $user->email, [
                'policy_name' => $policy->policy_name,
                'category' => $policy->category_name,
            ]);
        }
    }
    
    public function updateExpiredPolicies() {
        Capsule::table('mod_policies')
            ->where('expiration_date', '<', Carbon::today())
            ->where('status', 'published')
            ->update(['status' => 'archived']);
    }
}
```

### lib/PolicyAcknowledgment.php

```php
<?php
namespace WHMCS\Module\PolicyManagement;

use Illuminate\Database\Capsule\Manager as Capsule;
use Carbon\Carbon;

class PolicyAcknowledgment {
    
    public function acknowledge($policyId, $userId) {
        $policy = Capsule::table('mod_policies')->where('id', $policyId)->first();
        
        if (!$policy) {
            return ['success' => false, 'msg' => 'Policy not found'];
        }
        
        if ($policy->acknowledgment_frequency !== 'once') {
            Capsule::table('mod_policy_acknowledgments')
                ->where('policy_id', $policyId)
                ->where('user_id', $userId)
                ->delete();
        }
        
        $ackId = Capsule::table('mod_policy_acknowledgments')->insertGetId([
            'policy_id' => $policyId,
            'user_id' => $userId,
            'acknowledged_at' => Carbon::now(),
            'acknowledgment_method' => 'web',
            'ip_address' => $_SERVER['REMOTE_ADDR'] ?? '',
        ]);
        
        return ['success' => true, 'acknowledgment_id' => $ackId];
    }
    
    public function checkCompliance($userId) {
        $mandatoryPolicies = Capsule::table('mod_policies')
            ->where('is_mandatory', 1)
            ->where('status', 'published')
            ->where(function($q) {
                $q->whereNull('expiration_date')
                    ->orWhere('expiration_date', '>=', Carbon::today());
            })
            ->get();
        
        $acknowledged = Capsule::table('mod_policy_acknowledgments')
            ->where('user_id', $userId)
            ->pluck('policy_id')
            ->toArray();
        
        $pending = [];
        $compliant = true;
        
        foreach ($mandatoryPolicies as $policy) {
            if (!in_array($policy->id, $acknowledged)) {
                $pending[] = [
                    'id' => $policy->id,
                    'name' => $policy->policy_name,
                    'acknowledgment_frequency' => $policy->acknowledgment_frequency,
                ];
                $compliant = false;
            }
        }
        
        return [
            'compliant' => $compliant,
            'total_mandatory' => $mandatoryPolicies->count(),
            'acknowledged' => count($acknowledged),
            'pending' => $pending,
        ];
    }
    
    public function getPendingPolicies($userId) {
        $compliance = $this->checkCompliance($userId);
        return $compliance['pending'] ?? [];
    }
    
    public function getAcknowledgmentHistory($policyId) {
        return Capsule::table('mod_policy_acknowledgments as a')
            ->join('tblusers as u', 'a.user_id', '=', 'u.id')
            ->where('a.policy_id', $policyId)
            ->select('a.*', 'u.email', 'u.firstname', 'u.lastname')
            ->orderBy('a.acknowledged_at', 'desc')
            ->get();
    }
    
    public function getComplianceReport() {
        $policies = Capsule::table('mod_policies')
            ->where('is_mandatory', 1)
            ->where('status', 'published')
            ->get();
        
        $report = [];
        $totalUsers = Capsule::table('tblusers')->count();
        
        foreach ($policies as $policy) {
            $acknowledged = Capsule::table('mod_policy_acknowledgments')
                ->where('policy_id', $policy->id)
                ->count();
            
            $report[] = [
                'policy_id' => $policy->id,
                'policy_name' => $policy->policy_name,
                'total_users' => $totalUsers,
                'acknowledged' => $acknowledged,
                'pending' => $totalUsers - $acknowledged,
                'compliance_rate' => $totalUsers > 0 ? round(($acknowledged / $totalUsers) * 100, 2) : 0,
            ];
        }
        
        return $report;
    }
}
```

## API Endpoints

```
GET  /api/v1/policy/policies              - List all policies
GET  /api/v1/policy/policies/{id}         - Get specific policy
POST /api/v1/policy/policies              - Create policy
PUT  /api/v1/policy/policies/{id}         - Update policy
POST /api/v1/policy/policies/{id}/publish - Publish policy
POST /api/v1/policy/acknowledge           - Acknowledge policy
GET  /api/v1/policy/compliance/{userId}    - Check user compliance
GET  /api/v1/policy/report               - Compliance report
GET  /api/v1/policy/pending/{userId}      - Get pending policies
```
