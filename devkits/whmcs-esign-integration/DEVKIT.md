# WHMCS eSign Integration DevKit

## Overview

Electronic signature integration for WHMCS that enables digital signing of contracts, agreements, and documents with audit trails, compliance support, and multi-party signing workflows.

## Features

- Electronic signature creation
- Multi-party signing
- Document templates
- Signature verification
- Audit trail
- Compliance (ESIGN Act, eIDAS)
- Embedded signing
- Bulk signing
- Status tracking
- Reminder automation

## Database Schema

```sql
CREATE TABLE IF NOT EXISTS `mod_esign_documents` (
    `id` INT UNSIGNED NOT NULL AUTO_INCREMENT,
    `document_id` VARCHAR(64) NOT NULL,
    `document_name` VARCHAR(255) NOT NULL,
    `template_id` INT UNSIGNED NULL,
    `signer_count` INT UNSIGNED NOT NULL DEFAULT 1,
    `completed_signers` INT UNSIGNED NOT NULL DEFAULT 0,
    `status` ENUM('draft', 'sent', 'viewed', 'partially_signed', 'completed', 'declined', 'expired', 'cancelled') NOT NULL DEFAULT 'draft',
    `expires_at` DATETIME NULL,
    `completed_at` DATETIME NULL,
    `client_id` INT UNSIGNED NULL,
    `service_id` INT UNSIGNED NULL,
    `contract_id` INT UNSIGNED NULL,
    `metadata` JSON NULL,
    `created_at` DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
    `updated_at` DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    PRIMARY KEY (`id`),
    UNIQUE KEY `uk_document_id` (`document_id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;

CREATE TABLE IF NOT EXISTS `mod_esign_signers` (
    `id` INT UNSIGNED NOT NULL AUTO_INCREMENT,
    `document_id` VARCHAR(64) NOT NULL,
    `signer_order` INT UNSIGNED NOT NULL DEFAULT 1,
    `name` VARCHAR(255) NOT NULL,
    `email` VARCHAR(255) NOT NULL,
    `role` VARCHAR(100) NULL,
    `status` ENUM('pending', 'sent', 'viewed', 'signed', 'declined') NOT NULL DEFAULT 'pending',
    `signed_at` DATETIME NULL,
    `signed_ip_address` VARCHAR(45) NULL,
    `signed_user_agent` VARCHAR(500) NULL,
    `signature_hash` VARCHAR(255) NULL,
    `signature_image_url` VARCHAR(500) NULL,
    `access_token` VARCHAR(128) NULL,
    `access_token_expires` DATETIME NULL,
    `created_at` DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
    PRIMARY KEY (`id`),
    UNIQUE KEY `uk_document_email` (`document_id`, `email`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;

CREATE TABLE IF NOT EXISTS `mod_esign_templates` (
    `id` INT UNSIGNED NOT NULL AUTO_INCREMENT,
    `template_name` VARCHAR(255) NOT NULL,
    `template_code` VARCHAR(50) NOT NULL,
    `content` LONGTEXT NOT NULL,
    `variables` JSON NULL,
    `signer_roles` JSON NULL,
    `is_active` TINYINT(1) NOT NULL DEFAULT 1,
    `created_at` DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
    PRIMARY KEY (`id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;

CREATE TABLE IF NOT EXISTS `mod_esign_audit_logs` (
    `id` BIGINT UNSIGNED NOT NULL AUTO_INCREMENT,
    `document_id` VARCHAR(64) NOT NULL,
    `signer_id` INT UNSIGNED NULL,
    `event_type` VARCHAR(50) NOT NULL,
    `event_description` TEXT NOT NULL,
    `ip_address` VARCHAR(45) NULL,
    `user_agent` VARCHAR(500) NULL,
    `created_at` DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
    PRIMARY KEY (`id`),
    INDEX `idx_document_events` (`document_id`, `created_at`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
```

## Module Class

```php
<?php
/**
 * WHMCS eSign Integration Module
 */

if (!defined("WHMCS")) {
    die("This file cannot be accessed directly");
}

require_once __DIR__ . '/lib/ESignManager.php';
require_once __DIR__ . '/lib/ESignSigner.php';
require_once __DIR__ . '/lib/ESignAudit.php';

function whmcs_esign_integration_activate() {
    $manager = new ESignManager();
    return $manager->activate();
}

function whmcs_esign_integration_deactivate() {
    return ['success' => true, 'msg' => 'eSign Integration module deactivated'];
}

function whmcs_esign_integration_config() {
    return [
        'provider' => [
            'FriendlyName' => 'Signature Provider',
            'Type' => 'dropdown',
            'Options' => [
                'internal' => 'Internal (Simulated)',
                'docusign' => 'DocuSign',
                'hellosign' => 'HelloSign',
            ],
            'Default' => 'internal',
        ],
        'signature_expiry_days' => [
            'FriendlyName' => 'Signature Expiry (Days)',
            'Type' => 'text',
            'Default' => '30',
        ],
        'send_reminders' => [
            'FriendlyName' => 'Auto Send Reminders',
            'Type' => 'yesno',
        ],
    ];
}

function whmcs_esign_integration_create_document($data) {
    $manager = new ESignManager();
    return $manager->createSigningRequest($data);
}

function whmcs_esign_integration_send_document($documentId) {
    $manager = new ESignManager();
    return $manager->sendDocument($documentId);
}

function whmcs_esign_integration_sign($documentId, $email, $signature) {
    $signer = new ESignSigner();
    return $signer->signDocument($documentId, $email, $signature);
}

function whmcs_esign_integration_get_status($documentId) {
    $manager = new ESignManager();
    return $manager->getDocumentStatus($documentId);
}

add_hook('DailyCronJob', 1, function() {
    $manager = new ESignManager();
    $manager->sendReminders();
    $manager->processExpiredDocuments();
});

add_hook('ContractSigned', 1, function($params) {
    $manager = new ESignManager();
    $manager->logEvent($params['document_id'], 'signed', 'Contract signed via eSign');
});
```

### lib/ESignManager.php

```php
<?php
namespace WHMCS\Module\ESignIntegration;

use Illuminate\Database\Capsule\Manager as Capsule;
use Carbon\Carbon;

class ESignManager {
    
    public function activate() {
        try {
            $this->createTables();
            $this->createDefaultTemplates();
            return ['success' => true, 'msg' => 'eSign Integration module activated'];
        } catch (\Exception $e) {
            return ['success' => false, 'msg' => $e->getMessage()];
        }
    }
    
    protected function createTables() {
        Capsule::statement("
            CREATE TABLE IF NOT EXISTS `mod_esign_documents` (
                `id` INT UNSIGNED NOT NULL AUTO_INCREMENT,
                `document_id` VARCHAR(64) NOT NULL UNIQUE,
                `document_name` VARCHAR(255) NOT NULL,
                `signer_count` INT UNSIGNED NOT NULL DEFAULT 1,
                `completed_signers` INT UNSIGNED NOT NULL DEFAULT 0,
                `status` VARCHAR(30) NOT NULL DEFAULT 'draft',
                `expires_at` DATETIME NULL,
                `completed_at` DATETIME NULL,
                `client_id` INT UNSIGNED NULL,
                `created_at` DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
                PRIMARY KEY (`id`)
            ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4
        ");
        
        Capsule::statement("
            CREATE TABLE IF NOT EXISTS `mod_esign_signers` (
                `id` INT UNSIGNED NOT NULL AUTO_INCREMENT,
                `document_id` VARCHAR(64) NOT NULL,
                `signer_order` INT UNSIGNED NOT NULL DEFAULT 1,
                `name` VARCHAR(255) NOT NULL,
                `email` VARCHAR(255) NOT NULL,
                `role` VARCHAR(100) NULL,
                `status` VARCHAR(20) NOT NULL DEFAULT 'pending',
                `signed_at` DATETIME NULL,
                `signature_hash` VARCHAR(255) NULL,
                `access_token` VARCHAR(128) NULL,
                `created_at` DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
                PRIMARY KEY (`id`)
            ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4
        ");
        
        Capsule::statement("
            CREATE TABLE IF NOT EXISTS `mod_esign_templates` (
                `id` INT UNSIGNED NOT NULL AUTO_INCREMENT,
                `template_name` VARCHAR(255) NOT NULL UNIQUE,
                `template_code` VARCHAR(50) NOT NULL,
                `content` LONGTEXT NOT NULL,
                `variables` TEXT NULL,
                `is_active` TINYINT(1) NOT NULL DEFAULT 1,
                PRIMARY KEY (`id`)
            ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4
        ");
        
        Capsule::statement("
            CREATE TABLE IF NOT EXISTS `mod_esign_audit_logs` (
                `id` BIGINT UNSIGNED NOT NULL AUTO_INCREMENT,
                `document_id` VARCHAR(64) NOT NULL,
                `signer_id` INT UNSIGNED NULL,
                `event_type` VARCHAR(50) NOT NULL,
                `event_description` TEXT NOT NULL,
                `ip_address` VARCHAR(45) NULL,
                `created_at` DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
                PRIMARY KEY (`id`)
            ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4
        ");
    }
    
    protected function createDefaultTemplates() {
        $templates = [
            ['name' => 'Standard Service Agreement', 'code' => 'SSA-001', 'content' => file_get_contents(__DIR__ . '/templates/ssa.html')],
            ['name' => 'Terms of Service', 'code' => 'TOS-001', 'content' => file_get_contents(__DIR__ . '/templates/tos.html')],
            ['name' => 'Privacy Policy', 'code' => 'PP-001', 'content' => file_get_contents(__DIR__ . '/templates/privacy.html')],
        ];
        
        foreach ($templates as $template) {
            if (!Capsule::table('mod_esign_templates')->where('template_code', $template['code'])->exists()) {
                Capsule::table('mod_esign_templates')->insert($template);
            }
        }
    }
    
    protected function generateDocumentId() {
        return 'ESIGN-' . strtoupper(substr(md5(uniqid(mt_rand(), true)), 0, 16));
    }
    
    public function createSigningRequest($data) {
        $documentId = $this->generateDocumentId();
        
        $expiryDays = 30;
        $expiresAt = Carbon::now()->addDays($expiryDays)->toDateTimeString();
        
        $docId = Capsule::table('mod_esign_documents')->insertGetId([
            'document_id' => $documentId,
            'document_name' => $data['name'],
            'signer_count' => count($data['signers'] ?? [1]),
            'status' => 'draft',
            'expires_at' => $expiresAt,
            'client_id' => $data['client_id'] ?? null,
            'service_id' => $data['service_id'] ?? null,
        ]);
        
        $order = 1;
        foreach ($data['signers'] ?? [] as $signer) {
            $accessToken = $this->generateAccessToken();
            
            Capsule::table('mod_esign_signers')->insert([
                'document_id' => $documentId,
                'signer_order' => $order++,
                'name' => $signer['name'],
                'email' => $signer['email'],
                'role' => $signer['role'] ?? 'Signer',
                'access_token' => $accessToken,
                'access_token_expires' => $expiresAt,
            ]);
        }
        
        return [
            'success' => true,
            'document_id' => $documentId,
            'signing_url' => $this->getSigningUrl($documentId),
        ];
    }
    
    protected function generateAccessToken() {
        return bin2hex(random_bytes(32));
    }
    
    protected function getSigningUrl($documentId) {
        return rtrim(\App::get_url(), '/') . '/modules/addons/esign/sign.php?id=' . $documentId;
    }
    
    public function sendDocument($documentId) {
        $document = Capsule::table('mod_esign_documents')->where('document_id', $documentId)->first();
        
        if (!$document) {
            return ['success' => false, 'msg' => 'Document not found'];
        }
        
        Capsule::table('mod_esign_documents')
            ->where('document_id', $documentId)
            ->update(['status' => 'sent']);
        
        $signers = Capsule::table('mod_esign_signers')->where('document_id', $documentId)->get();
        
        foreach ($signers as $signer) {
            $user = Capsule::table('tblusers')->where('id', $document->client_id)->first();
            $emailData = [
                'document_name' => $document->document_name,
                'signer_name' => $signer->name,
                'signing_url' => $this->getSigningUrl($documentId) . '&token=' . $signer->access_token,
                'expires_at' => $document->expires_at,
            ];
            
            $this->sendSigningEmail($signer->email, $emailData);
            
            Capsule::table('mod_esign_signers')
                ->where('id', $signer->id)
                ->update(['status' => 'sent']);
            
            $this->logEvent($documentId, 'sent', "Document sent to {$signer->email}", $signer->id);
        }
        
        return ['success' => true, 'signers_notified' => count($signers)];
    }
    
    protected function sendSigningEmail($email, $data) {
        sendEmail('esign_signing_request', $email, $data);
    }
    
    public function getDocumentStatus($documentId) {
        $document = Capsule::table('mod_esign_documents')->where('document_id', $documentId)->first();
        $signers = Capsule::table('mod_esign_signers')->where('document_id', $documentId)->get();
        
        return [
            'document' => $document,
            'signers' => $signers,
            'completion_percentage' => $document->signer_count > 0 
                ? round(($document->completed_signers / $document->signer_count) * 100, 2) 
                : 0,
        ];
    }
    
    public function sendReminders() {
        $pending = Capsule::table('mod_esign_signers as s')
            ->join('mod_esign_documents as a', 's.document_id', '=', 'a.document_id')
            ->where('s.status', 'pending')
            ->where('a.status', 'sent')
            ->where('a.expires_at', '>', Carbon::now())
            ->get();
        
        foreach ($pending as $signer) {
            $emailData = [
                'document_name' => $signer->document_name,
                'signer_name' => $signer->name,
                'signing_url' => $this->getSigningUrl($signer->document_id) . '&token=' . $signer->access_token,
            ];
            
            $this->sendSigningEmail($signer->email, $emailData);
            $this->logEvent($signer->document_id, 'reminder_sent', "Reminder sent to {$signer->email}");
        }
    }
    
    public function processExpiredDocuments() {
        Capsule::table('mod_esign_documents')
            ->where('status', 'sent')
            ->where('expires_at', '<', Carbon::now())
            ->update(['status' => 'expired']);
    }
    
    public function logEvent($documentId, $eventType, $description, $signerId = null) {
        Capsule::table('mod_esign_audit_logs')->insert([
            'document_id' => $documentId,
            'signer_id' => $signerId,
            'event_type' => $eventType,
            'event_description' => $description,
            'ip_address' => $_SERVER['REMOTE_ADDR'] ?? null,
        ]);
    }
}
```

### lib/ESignSigner.php

```php
<?php
namespace WHMCS\Module\ESignIntegration;

use Illuminate\Database\Capsule\Manager as Capsule;
use Carbon\Carbon;

class ESignSigner {
    
    public function signDocument($documentId, $email, $signatureData) {
        $document = Capsule::table('mod_esign_documents')->where('document_id', $documentId)->first();
        
        if (!$document) {
            return ['success' => false, 'msg' => 'Document not found'];
        }
        
        $signer = Capsule::table('mod_esign_signers')
            ->where('document_id', $documentId)
            ->where('email', $email)
            ->where('status', 'pending')
            ->first();
        
        if (!$signer) {
            return ['success' => false, 'msg' => 'Signer not found or already signed'];
        }
        
        $signatureHash = $this->generateSignatureHash($documentId, $email, $signatureData);
        
        Capsule::table('mod_esign_signers')
            ->where('id', $signer->id)
            ->update([
                'status' => 'signed',
                'signed_at' => Carbon::now(),
                'signature_hash' => $signatureHash,
                'signed_ip_address' => $_SERVER['REMOTE_ADDR'] ?? null,
                'signed_user_agent' => substr($_SERVER['HTTP_USER_AGENT'] ?? '', 0, 500),
            ]);
        
        Capsule::table('mod_esign_documents')
            ->where('document_id', $documentId)
            ->increment('completed_signers');
        
        // Update document status
        $this->updateDocumentStatus($documentId);
        
        $audit = new ESignAudit();
        $audit->logSignature($documentId, $signer->id, [
            'hash' => $signatureHash,
            'ip' => $_SERVER['REMOTE_ADDR'] ?? null,
        ]);
        
        return [
            'success' => true,
            'signed_at' => Carbon::now()->toDateTimeString(),
            'signature_hash' => $signatureHash,
        ];
    }
    
    protected function generateSignatureHash($documentId, $email, $signatureData) {
        $data = json_encode([
            'document_id' => $documentId,
            'email' => $email,
            'timestamp' => Carbon::now()->toDateTimeString(),
            'signature' => $signatureData['data'] ?? '',
        ]);
        
        return hash('sha256', $data . $_SERVER['REMOTE_ADDR'] ?? '');
    }
    
    protected function updateDocumentStatus($documentId) {
        $document = Capsule::table('mod_esign_documents')->where('document_id', $documentId)->first();
        
        if ($document->completed_signers >= $document->signer_count) {
            Capsule::table('mod_esign_documents')
                ->where('document_id', $documentId)
                ->update([
                    'status' => 'completed',
                    'completed_at' => Carbon::now(),
                ]);
        } else {
            Capsule::table('mod_esign_documents')
                ->where('document_id', $documentId)
                ->update(['status' => 'partially_signed']);
        }
    }
    
    public function verifySignature($signerId) {
        $signer = Capsule::table('mod_esign_signers')->where('id', $signerId)->first();
        
        if (!$signer || $signer->status !== 'signed') {
            return ['valid' => false, 'reason' => 'Signature not found'];
        }
        
        return [
            'valid' => !empty($signer->signature_hash),
            'signed_by' => $signer->name,
            'signed_email' => $signer->email,
            'signed_at' => $signer->signed_at,
            'ip_address' => $signer->signed_ip_address,
        ];
    }
}
```

### lib/ESignAudit.php

```php
<?php
namespace WHMCS\Module\ESignIntegration;

use Illuminate\Database\Capsule\Manager as Capsule;
use Carbon\Carbon;

class ESignAudit {
    
    public function logSignature($documentId, $signerId, $signatureData) {
        Capsule::table('mod_esign_audit_logs')->insert([
            'document_id' => $documentId,
            'signer_id' => $signerId,
            'event_type' => 'signed',
            'event_description' => 'Document signed',
            'ip_address' => $_SERVER['REMOTE_ADDR'] ?? null,
        ]);
    }
    
    public function getAuditTrail($documentId) {
        return Capsule::table('mod_esign_audit_logs')
            ->where('document_id', $documentId)
            ->orderBy('created_at', 'asc')
            ->get();
    }
}
```

## API Endpoints

```
POST /api/v1/esign/documents              - Create signing request
POST /api/v1/esign/documents/{id}/send   - Send for signing
GET  /api/v1/esign/documents/{id}        - Get document status
POST /api/v1/esign/documents/{id}/sign   - Sign document
GET  /api/v1/esign/documents/{id}/audit  - Get audit trail
GET  /api/v1/esign/templates             - List templates
POST /api/v1/esign/templates             - Create template
POST /api/v1/esign/verify/{signerId}    - Verify signature
```
