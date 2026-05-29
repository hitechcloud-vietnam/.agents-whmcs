# WHMCS AML Screening DevKit

## Overview

Anti-Money Laundering (AML) screening system for WHMCS enabling customer due diligence, transaction monitoring, and regulatory compliance.

## Features

- Customer due diligence
- Risk scoring
- PEP screening
- Adverse media checks
- Enhanced due diligence
- Transaction monitoring
- SAR filing support

## Module Files

```php
<?php
/**
 * WHMCS AML Screening Module
 */

if (!defined("WHMCS")) {
    die("This file cannot be accessed directly");
}

require_once __DIR__ . '/lib/AmlScreening.php';

function whmcs_aml_screening_activate() {
    $aml = new AmlScreening();
    return $aml->activate();
}

function whmcs_aml_screen_customer($userId) {
    $aml = new AmlScreening();
    return $aml->screenCustomer($userId);
}
```

### lib/AmlScreening.php

```php
<?php
namespace WHMCS\Module\AmlScreening;

use Illuminate\Database\Capsule\Manager as Capsule;
use Carbon\Carbon;

class AmlScreening {
    
    public function activate() {
        try {
            $this->createTables();
            return ['success' => true, 'msg' => 'AML Screening module activated'];
        } catch (\Exception $e) {
            return ['success' => false, 'msg' => $e->getMessage()];
        }
    }
    
    protected function createTables() {
        Capsule::statement("
            CREATE TABLE IF NOT EXISTS `mod_aml_screenings` (
                `id` INT UNSIGNED NOT NULL AUTO_INCREMENT,
                `user_id` INT UNSIGNED NOT NULL,
                `screening_id` VARCHAR(64) NOT NULL,
                `risk_level` ENUM('low', 'medium', 'high', 'critical') NOT NULL DEFAULT 'low',
                `risk_score` DECIMAL(5,2) DEFAULT 0,
                `pep_match` TINYINT(1) DEFAULT 0,
                `sanctions_match` TINYINT(1) DEFAULT 0,
                `adverse_media` TINYINT(1) DEFAULT 0,
                `kyb_status` VARCHAR(50) NULL,
                `status` ENUM('pending', 'completed', 'escalated') NOT NULL DEFAULT 'pending',
                `screened_at` DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
                PRIMARY KEY (`id`)
            ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4
        ");
    }
    
    public function screenCustomer($userId) {
        $client = Capsule::table('tblclients')->where('id', $userId)->first();
        
        if (!$client) {
            return ['error' => 'Customer not found'];
        }
        
        $screeningId = 'AML-' . strtoupper(substr(md5(uniqid()), 0, 12));
        
        // Calculate risk score based on various factors
        $riskScore = $this->calculateRiskScore($client);
        
        // Check PEP
        $pepMatch = $this->checkPep($client);
        
        // Check sanctions
        $sanctionsMatch = $this->checkSanctions($client);
        
        // Check adverse media
        $adverseMedia = $this->checkAdverseMedia($client);
        
        $riskLevel = $this->determineRiskLevel($riskScore, $pepMatch, $sanctionsMatch);
        
        $id = Capsule::table('mod_aml_screenings')->insertGetId([
            'user_id' => $userId,
            'screening_id' => $screeningId,
            'risk_score' => $riskScore,
            'risk_level' => $riskLevel,
            'pep_match' => $pepMatch ? 1 : 0,
            'sanctions_match' => $sanctionsMatch ? 1 : 0,
            'adverse_media' => $adverseMedia ? 1 : 0,
            'status' => $riskLevel === 'critical' ? 'escalated' : 'completed',
        ]);
        
        return [
            'screening_id' => $screeningId,
            'risk_level' => $riskLevel,
            'risk_score' => $riskScore,
            'pep_match' => $pepMatch,
            'sanctions_match' => $sanctionsMatch,
            'adverse_media' => $adverseMedia,
        ];
    }
    
    protected function calculateRiskScore($client) {
        $score = 20; // Base score
        
        // Factor in account age
        $accountAge = Carbon::parse($client->datecreated)->diffInYears(Carbon::now());
        if ($accountAge < 1) $score += 30;
        elseif ($accountAge < 2) $score += 20;
        else $score += 0;
        
        // Factor in total spend
        $totalSpend = $client->total_amount ?? 0;
        if ($totalSpend > 10000) $score += 30;
        elseif ($totalSpend > 5000) $score += 20;
        
        return min(100, $score);
    }
    
    protected function checkPep($client) {
        // Implement PEP check logic
        return false;
    }
    
    protected function checkSanctions($client) {
        // Implement sanctions check
        return false;
    }
    
    protected function checkAdverseMedia($client) {
        // Implement adverse media check
        return false;
    }
    
    protected function determineRiskLevel($score, $pep, $sanctions) {
        if ($pep || $sanctions) return 'critical';
        if ($score >= 70) return 'high';
        if ($score >= 40) return 'medium';
        return 'low';
    }
}
```

## API Endpoints

```
POST /api/v1/aml/screen                  - Screen customer
GET  /api/v1/aml/screening/{id}        - Get screening result
GET  /api/v1/aml/customer/{userId}      - Get customer AML status
```
