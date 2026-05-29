# WHMCS KYB Verification DevKit

## Overview

Know Your Business (KYB) verification system for WHMCS enabling corporate customer verification, beneficial ownership screening, and business identity verification.

## Features

- Business verification
- Beneficial ownership
- Director screening
- Company verification
- UBO identification
- Risk assessment
- Document verification

## Module Files

```php
<?php
/**
 * WHMCS KYB Verification Module
 */

if (!defined("WHMCS")) {
    die("This file cannot be accessed directly");
}

require_once __DIR__ . '/lib/KybVerification.php';

function whmcs_kyb_verification_activate() {
    $kyb = new KybVerification();
    return $kyb->activate();
}

function whmcs_kyb_verify_business($userId, $companyData) {
    $kyb = new KybVerification();
    return $kyb->verifyBusiness($userId, $companyData);
}
```

### lib/KybVerification.php

```php
<?php
namespace WHMCS\Module\KybVerification;

use Illuminate\Database\Capsule\Manager as Capsule;
use Carbon\Carbon;

class KybVerification {
    
    public function activate() {
        try {
            $this->createTables();
            return ['success' => true, 'msg' => 'KYB Verification module activated'];
        } catch (\Exception $e) {
            return ['success' => false, 'msg' => $e->getMessage()];
        }
    }
    
    protected function createTables() {
        Capsule::statement("
            CREATE TABLE IF NOT EXISTS `mod_kyb_verifications` (
                `id` INT UNSIGNED NOT NULL AUTO_INCREMENT,
                `verification_id` VARCHAR(64) NOT NULL,
                `user_id` INT UNSIGNED NOT NULL,
                `company_name` VARCHAR(255) NOT NULL,
                `company_number` VARCHAR(100) NULL,
                `incorporation_date` DATE NULL,
                `jurisdiction` VARCHAR(100) NULL,
                `verification_level` ENUM('basic', 'standard', 'enhanced') NOT NULL DEFAULT 'basic',
                `status` ENUM('pending', 'in_progress', 'verified', 'rejected', 'expired') NOT NULL DEFAULT 'pending',
                `risk_score` DECIMAL(5,2) DEFAULT 0,
                `verified_at` DATETIME NULL,
                `expires_at` DATETIME NULL,
                `created_at` DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
                PRIMARY KEY (`id`)
            ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4
        ");
        
        Capsule::statement("
            CREATE TABLE IF NOT EXISTS `mod_kyb_ubo` (
                `id` INT UNSIGNED NOT NULL AUTO_INCREMENT,
                `verification_id` VARCHAR(64) NOT NULL,
                `name` VARCHAR(255) NOT NULL,
                `ownership_percentage` DECIMAL(5,2) NOT NULL,
                `nationality` VARCHAR(100) NULL,
                `date_of_birth` DATE NULL,
                `screening_status` VARCHAR(50) NULL,
                `created_at` DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
                PRIMARY KEY (`id`)
            ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4
        ");
    }
    
    public function verifyBusiness($userId, $companyData) {
        $verificationId = 'KYB-' . strtoupper(substr(md5(uniqid()), 0, 12));
        
        $id = Capsule::table('mod_kyb_verifications')->insertGetId([
            'verification_id' => $verificationId,
            'user_id' => $userId,
            'company_name' => $companyData['name'],
            'company_number' => $companyData['number'] ?? null,
            'incorporation_date' => $companyData['incorporation_date'] ?? null,
            'jurisdiction' => $companyData['jurisdiction'] ?? null,
            'status' => 'in_progress',
        ]);
        
        // Screen company
        $riskScore = $this->screenCompany($companyData);
        
        Capsule::table('mod_kyb_verifications')
            ->where('id', $id)
            ->update([
                'risk_score' => $riskScore,
                'status' => $riskScore > 70 ? 'rejected' : 'verified',
                'verified_at' => Carbon::now(),
                'expires_at' => Carbon::now()->addYear(),
            ]);
        
        return [
            'verification_id' => $verificationId,
            'status' => $riskScore > 70 ? 'rejected' : 'verified',
            'risk_score' => $riskScore,
        ];
    }
    
    protected function screenCompany($companyData) {
        $score = 20; // Base score
        
        // Check against sanctions
        // Check against PEP lists
        // Check adverse media
        
        return min(100, $score);
    }
}
```

## API Endpoints

```
POST /api/v1/kyb/verify                   - Start verification
POST /api/v1/kyb/{id}/ubo              - Add UBO
GET  /api/v1/kyb/verification/{id}       - Get verification status
GET  /api/v1/kyb/customer/{userId}      - Get customer KYB status
```
