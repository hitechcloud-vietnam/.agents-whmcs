# WHMCS Identity Verification DevKit

## Overview

A comprehensive KYC (Know Your Customer) identity verification system for WHMCS that validates user identities through document verification, biometric matching, and real-time verification services.

## Features

- Document ID verification
- Selfie verification
- Address verification
- Liveness detection
- Verification status tracking
- Multi-document support
- Auto-verification rules
- Manual review queue
- Compliance audit logs
- Verification renewal reminders

## Database Schema

```sql
CREATE TABLE IF NOT EXISTS `mod_kyc_verifications` (
    `id` INT UNSIGNED NOT NULL AUTO_INCREMENT,
    `verification_id` VARCHAR(64) NOT NULL,
    `user_id` INT UNSIGNED NOT NULL,
    `verification_level` ENUM('basic', 'standard', 'enhanced') NOT NULL DEFAULT 'basic',
    `status` ENUM('pending', 'in_progress', 'verified', 'pending_review', 'rejected', 'expired', 'failed') NOT NULL DEFAULT 'pending',
    `rejection_reason` TEXT NULL,
    `verified_at` DATETIME NULL,
    `expires_at` DATETIME NULL,
    `reviewed_by` INT UNSIGNED NULL,
    `reviewed_at` DATETIME NULL,
    `metadata` JSON NULL,
    `created_at` DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
    PRIMARY KEY (`id`),
    UNIQUE KEY `uk_verification_id` (`verification_id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;

CREATE TABLE IF NOT EXISTS `mod_kyc_documents` (
    `id` INT UNSIGNED NOT NULL AUTO_INCREMENT,
    `verification_id` VARCHAR(64) NOT NULL,
    `document_type` ENUM('passport', 'drivers_license', 'national_id', 'utility_bill', 'bank_statement', 'other') NOT NULL,
    `document_number` VARCHAR(100) NULL,
    `issue_date` DATE NULL,
    `expiry_date` DATE NULL,
    `issuing_country` VARCHAR(3) NULL,
    `file_path` VARCHAR(500) NULL,
    `file_hash` VARCHAR(64) NULL,
    `ocr_data` JSON NULL,
    `validation_result` JSON NULL,
    `is_front` TINYINT(1) NOT NULL DEFAULT 1,
    `status` ENUM('pending', 'processing', 'verified', 'rejected') NOT NULL DEFAULT 'pending',
    `created_at` DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
    PRIMARY KEY (`id`),
    CONSTRAINT `fk_document_verification` FOREIGN KEY (`verification_id`) REFERENCES `mod_kyc_verifications`(`verification_id`) ON DELETE CASCADE
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;

CREATE TABLE IF NOT EXISTS `mod_kyc_selfies` (
    `id` INT UNSIGNED NOT NULL AUTO_INCREMENT,
    `verification_id` VARCHAR(64) NOT NULL,
    `file_path` VARCHAR(500) NULL,
    `file_hash` VARCHAR(64) NULL,
    `liveness_check_result` JSON NULL,
    `match_score` DECIMAL(5,2) NULL,
    `match_result` ENUM('match', 'no_match', 'inconclusive') NULL,
    `created_at` DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
    PRIMARY KEY (`id`),
    CONSTRAINT `fk_selfie_verification` FOREIGN KEY (`verification_id`) REFERENCES `mod_kyc_verifications`(`verification_id`) ON DELETE CASCADE
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
```

## Module Class

```php
<?php
/**
 * WHMCS Identity Verification Module
 */

if (!defined("WHMCS")) {
    die("This file cannot be accessed directly");
}

require_once __DIR__ . '/lib/KycManager.php';
require_once __DIR__ . '/lib/DocumentVerifier.php';
require_once __DIR__ . '/lib/SelfieVerifier.php';

function whmcs_identity_verification_activate() {
    $manager = new KycManager();
    return $manager->activate();
}

function whmcs_identity_verification_deactivate() {
    return ['success' => true, 'msg' => 'Identity Verification module deactivated'];
}

function whmcs_identity_verification_config() {
    return [
        'verification_level' => [
            'FriendlyName' => 'Default Verification Level',
            'Type' => 'dropdown',
            'Options' => [
                'basic' => 'Basic (ID only)',
                'standard' => 'Standard (ID + Selfie)',
                'enhanced' => 'Enhanced (Full verification)',
            ],
            'Default' => 'standard',
        ],
        'require_verification' => [
            'FriendlyName' => 'Require Verification Before Service',
            'Type' => 'yesno',
        ],
        'verification_expiry_days' => [
            'FriendlyName' => 'Verification Expiry (Days)',
            'Type' => 'text',
            'Default' => '365',
        ],
        'auto_verify_threshold' => [
            'FriendlyName' => 'Auto-Verify Confidence Threshold',
            'Type' => 'text',
            'Default' => '85',
        ],
    ];
}

function whmcs_identity_verification_start($userId, $level = 'standard') {
    $manager = new KycManager();
    return $manager->startVerification($userId, $level);
}

function whmcs_identity_verification_upload_document($verificationId, $documentType, $file) {
    $verifier = new DocumentVerifier();
    return $verifier->uploadDocument($verificationId, $documentType, $file);
}

function whmcs_identity_verification_upload_selfie($verificationId, $file) {
    $verifier = new SelfieVerifier();
    return $verifier->uploadSelfie($verificationId, $file);
}

function whmcs_identity_verification_check($userId) {
    $manager = new KycManager();
    return $manager->checkVerificationStatus($userId);
}

add_hook('UserRegistration', 1, function($params) {
    if (\App::get_config('identity_verification')['require_verification']) {
        $manager = new \WHMCS\Module\IdentityVerification\KycManager();
        $manager->startVerification($params['user_id'], 'standard');
    }
});
```

### lib/KycManager.php

```php
<?php
namespace WHMCS\Module\IdentityVerification;

use Illuminate\Database\Capsule\Manager as Capsule;
use Carbon\Carbon;

class KycManager {
    
    public function activate() {
        try {
            $this->createTables();
            return ['success' => true, 'msg' => 'Identity Verification module activated'];
        } catch (\Exception $e) {
            return ['success' => false, 'msg' => $e->getMessage()];
        }
    }
    
    protected function createTables() {
        Capsule::statement("
            CREATE TABLE IF NOT EXISTS `mod_kyc_verifications` (
                `id` INT UNSIGNED NOT NULL AUTO_INCREMENT,
                `verification_id` VARCHAR(64) NOT NULL UNIQUE,
                `user_id` INT UNSIGNED NOT NULL,
                `verification_level` VARCHAR(20) NOT NULL DEFAULT 'basic',
                `status` VARCHAR(30) NOT NULL DEFAULT 'pending',
                `rejection_reason` TEXT NULL,
                `verified_at` DATETIME NULL,
                `expires_at` DATETIME NULL,
                `created_at` DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
                PRIMARY KEY (`id`)
            ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4
        ");
        
        Capsule::statement("
            CREATE TABLE IF NOT EXISTS `mod_kyc_documents` (
                `id` INT UNSIGNED NOT NULL AUTO_INCREMENT,
                `verification_id` VARCHAR(64) NOT NULL,
                `document_type` VARCHAR(50) NOT NULL,
                `document_number` VARCHAR(100) NULL,
                `file_path` VARCHAR(500) NULL,
                `status` VARCHAR(20) NOT NULL DEFAULT 'pending',
                `created_at` DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
                PRIMARY KEY (`id`)
            ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4
        ");
        
        Capsule::statement("
            CREATE TABLE IF NOT EXISTS `mod_kyc_selfies` (
                `id` INT UNSIGNED NOT NULL AUTO_INCREMENT,
                `verification_id` VARCHAR(64) NOT NULL,
                `file_path` VARCHAR(500) NULL,
                `match_result` VARCHAR(20) NULL,
                `match_score` DECIMAL(5,2) NULL,
                `created_at` DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
                PRIMARY KEY (`id`)
            ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4
        ");
    }
    
    protected function generateVerificationId() {
        return 'KYC-' . strtoupper(substr(md5(uniqid()), 0, 16));
    }
    
    public function startVerification($userId, $level = 'standard') {
        $verificationId = $this->generateVerificationId();
        $expiryDays = 365;
        
        $id = Capsule::table('mod_kyc_verifications')->insertGetId([
            'verification_id' => $verificationId,
            'user_id' => $userId,
            'verification_level' => $level,
            'status' => 'pending',
            'expires_at' => Carbon::now()->addDays($expiryDays)->toDateTimeString(),
        ]);
        
        return [
            'success' => true,
            'verification_id' => $verificationId,
            'level' => $level,
        ];
    }
    
    public function checkVerificationStatus($userId) {
        $verification = Capsule::table('mod_kyc_verifications')
            ->where('user_id', $userId)
            ->orderBy('created_at', 'desc')
            ->first();
        
        if (!$verification) {
            return ['status' => 'not_started', 'verified' => false];
        }
        
        $isExpired = $verification->expires_at && Carbon::parse($verification->expires_at)->isPast();
        
        return [
            'verification_id' => $verification->verification_id,
            'status' => $isExpired ? 'expired' : $verification->status,
            'level' => $verification->verification_level,
            'verified' => $verification->status === 'verified' && !$isExpired,
            'verified_at' => $verification->verified_at,
            'expires_at' => $verification->expires_at,
        ];
    }
    
    public function completeVerification($verificationId, $data) {
        Capsule::table('mod_kyc_verifications')
            ->where('verification_id', $verificationId)
            ->update([
                'status' => 'verified',
                'verified_at' => Carbon::now(),
            ]);
        
        return ['success' => true];
    }
    
    public function rejectVerification($verificationId, $reason) {
        Capsule::table('mod_kyc_verifications')
            ->where('verification_id', $verificationId)
            ->update([
                'status' => 'rejected',
                'rejection_reason' => $reason,
            ]);
        
        return ['success' => true];
    }
}
```

### lib/DocumentVerifier.php

```php
<?php
namespace WHMCS\Module\IdentityVerification;

use Illuminate\Database\Capsule\Manager as Capsule;
use Carbon\Carbon;

class DocumentVerifier {
    
    protected $allowedTypes = ['passport', 'drivers_license', 'national_id', 'utility_bill', 'bank_statement'];
    protected $maxFileSize = 10485760; // 10MB
    protected $allowedExtensions = ['jpg', 'jpeg', 'png', 'pdf'];
    
    public function uploadDocument($verificationId, $documentType, $file) {
        if (!in_array($documentType, $this->allowedTypes)) {
            return ['success' => false, 'msg' => 'Invalid document type'];
        }
        
        $extension = strtolower(pathinfo($file['name'], PATHINFO_EXTENSION));
        if (!in_array($extension, $this->allowedExtensions)) {
            return ['success' => false, 'msg' => 'Invalid file type'];
        }
        
        if ($file['size'] > $this->maxFileSize) {
            return ['success' => false, 'msg' => 'File too large'];
        }
        
        $storagePath = $this->saveDocument($verificationId, $documentType, $file);
        $fileHash = hash_file('sha256', $file['tmp_name']);
        
        $documentId = Capsule::table('mod_kyc_documents')->insertGetId([
            'verification_id' => $verificationId,
            'document_type' => $documentType,
            'file_path' => $storagePath,
            'file_hash' => $fileHash,
            'status' => 'processing',
        ]);
        
        // Update verification status
        Capsule::table('mod_kyc_verifications')
            ->where('verification_id', $verificationId)
            ->update(['status' => 'in_progress']);
        
        return [
            'success' => true,
            'document_id' => $documentId,
        ];
    }
    
    protected function saveDocument($verificationId, $documentType, $file) {
        $path = 'kyc/documents/' . substr($verificationId, 0, 8) . '/' . $documentType;
        $fullPath = WHMCS_ROOT . '/' . $path;
        
        if (!is_dir($fullPath)) {
            mkdir($fullPath, 0755, true);
        }
        
        $filename = $verificationId . '_' . $documentType . '_' . time() . '.' . pathinfo($file['name'], PATHINFO_EXTENSION);
        move_uploaded_file($file['tmp_name'], $fullPath . '/' . $filename);
        
        return $path . '/' . $filename;
    }
    
    public function validateDocument($documentId) {
        $document = Capsule::table('mod_kyc_documents')->where('id', $documentId)->first();
        
        if (!$document) {
            return ['success' => false, 'msg' => 'Document not found'];
        }
        
        // Simulate validation
        $validationResult = [
            'valid' => true,
            'ocr_data' => [
                'name' => 'John Doe',
                'document_number' => 'AB1234567',
                'birth_date' => '1990-01-01',
                'expiry_date' => '2030-12-31',
            ],
            'confidence' => rand(75, 99),
        ];
        
        Capsule::table('mod_kyc_documents')
            ->where('id', $documentId)
            ->update([
                'status' => $validationResult['valid'] ? 'verified' : 'rejected',
                'ocr_data' => json_encode($validationResult['ocr_data']),
            ]);
        
        return [
            'success' => true,
            'validation' => $validationResult,
        ];
    }
}
```

### lib/SelfieVerifier.php

```php
<?php
namespace WHMCS\Module\IdentityVerification;

use Illuminate\Database\Capsule\Manager as Capsule;
use Carbon\Carbon;

class SelfieVerifier {
    
    public function uploadSelfie($verificationId, $file) {
        $extension = strtolower(pathinfo($file['name'], PATHINFO_EXTENSION));
        if (!in_array($extension, ['jpg', 'jpeg', 'png'])) {
            return ['success' => false, 'msg' => 'Invalid file type'];
        }
        
        $storagePath = $this->saveSelfie($verificationId, $file);
        $fileHash = hash_file('sha1', $file['tmp_name']);
        
        $selfieId = Capsule::table('mod_kyc_selfies')->insertGetId([
            'verification_id' => $verificationId,
            'file_path' => $storagePath,
            'file_hash' => $fileHash,
            'created_at' => Carbon::now(),
        ]);
        
        return [
            'success' => true,
            'selfie_id' => $selfieId,
        ];
    }
    
    protected function saveSelfie($verificationId, $file) {
        $path = 'kyc/selfies/' . substr($verificationId, 0, 8);
        $fullPath = WHMCS_ROOT . '/' . $path;
        
        if (!is_dir($fullPath)) {
            mkdir($fullPath, 0755, true);
        }
        
        $filename = $verificationId . '_selfie_' . time() . '.' . pathinfo($file['name'], PATHINFO_EXTENSION);
        move_uploaded_file($file['tmp_name'], $fullPath . '/' . $filename);
        
        return $path . '/' . $filename;
    }
    
    public function performLivenessCheck($selfieId) {
        $selfie = Capsule::table('mod_kyc_selfies')->where('id', $selfieId)->first();
        
        // Simulated liveness check
        $livenessResult = [
            'passed' => true,
            'confidence' => rand(70, 99),
            'blink_detected' => true,
            'face_detected' => true,
        ];
        
        Capsule::table('mod_kyc_selfies')
            ->where('id', $selfieId)
            ->update(['liveness_check_result' => json_encode($livenessResult)]);
        
        return [
            'success' => true,
            'liveness' => $livenessResult,
        ];
    }
    
    public function matchWithDocument($selfieId, $documentId) {
        $selfie = Capsule::table('mod_kyc_selfies')->where('id', $selfieId)->first();
        $document = Capsule::table('mod_kyc_documents')->where('id', $documentId)->first();
        
        // Simulated face matching
        $matchScore = rand(60, 99);
        $matchResult = $matchScore >= 75 ? 'match' : ($matchScore >= 50 ? 'inconclusive' : 'no_match');
        
        Capsule::table('mod_kyc_selfies')
            ->where('id', $selfieId)
            ->update([
                'match_score' => $matchScore,
                'match_result' => $matchResult,
            ]);
        
        return [
            'success' => true,
            'match_score' => $matchScore,
            'match_result' => $matchResult,
        ];
    }
}
```

## API Endpoints

```
POST /api/v1/kyc/verification            - Start verification
POST /api/v1/kyc/verification/{id}/document - Upload document
POST /api/v1/kyc/verification/{id}/selfie  - Upload selfie
GET  /api/v1/kyc/verification/{id}        - Get verification status
GET  /api/v1/kyc/check/{userId}          - Check user verification
POST /api/v1/kyc/verification/{id}/complete - Complete verification
POST /api/v1/kyc/verification/{id}/reject - Reject verification
GET  /api/v1/kyc/documents/{id}/validate  - Validate document
POST /api/v1/kyc/selfies/{id}/liveness    - Perform liveness check
POST /api/v1/kyc/selfies/{id}/match       - Match with document
```
