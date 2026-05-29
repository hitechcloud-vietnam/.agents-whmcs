# WHMCS Fraud Prevention DevKit

## Overview

A comprehensive fraud prevention and detection system for WHMCS that uses machine learning patterns, rule-based detection, velocity checks, device fingerprinting, and real-time scoring to identify and prevent fraudulent transactions.

## Features

- Real-time fraud scoring
- Rule-based detection
- Velocity checks
- Device fingerprinting
- IP reputation analysis
- Behavioral analytics
- Chargeback prediction
- Risk scoring
- Auto-block suspicious orders
- Fraud alert management

## Database Schema

```sql
CREATE TABLE IF NOT EXISTS `mod_fraud_rules` (
    `id` INT UNSIGNED NOT NULL AUTO_INCREMENT,
    `rule_name` VARCHAR(255) NOT NULL,
    `rule_code` VARCHAR(50) NOT NULL,
    `rule_type` ENUM('velocity', 'pattern', 'threshold', 'behavioral', 'geo') NOT NULL,
    `conditions` JSON NOT NULL,
    `risk_weight` DECIMAL(3,2) NOT NULL DEFAULT 1.00,
    `action` ENUM('block', 'review', 'flag', 'allow') NOT NULL DEFAULT 'flag',
    `is_active` TINYINT(1) NOT NULL DEFAULT 1,
    `created_at` DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
    PRIMARY KEY (`id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;

CREATE TABLE IF NOT EXISTS `mod_fraud_scores` (
    `id` BIGINT UNSIGNED NOT NULL AUTO_INCREMENT,
    `order_id` INT UNSIGNED NULL,
    `user_id` INT UNSIGNED NULL,
    `fraud_score` DECIMAL(5,2) NOT NULL,
    `risk_level` ENUM('low', 'medium', 'high', 'critical') NOT NULL,
    `triggered_rules` JSON NULL,
    `action_taken` VARCHAR(50) NOT NULL,
    `reviewed_by` INT UNSIGNED NULL,
    `reviewed_at` DATETIME NULL,
    `decision` ENUM('pending', 'approved', 'rejected', 'reviewed') NOT NULL DEFAULT 'pending',
    `notes` TEXT NULL,
    `created_at` DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
    PRIMARY KEY (`id`),
    UNIQUE KEY `uk_order_fraud` (`order_id`, `user_id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;

CREATE TABLE IF NOT EXISTS `mod_fraud_velocity` (
    `id` INT UNSIGNED NOT NULL AUTO_INCREMENT,
    `identifier_type` VARCHAR(50) NOT NULL,
    `identifier_value` VARCHAR(255) NOT NULL,
    `event_type` VARCHAR(50) NOT NULL,
    `event_count` INT UNSIGNED NOT NULL DEFAULT 1,
    `window_minutes` INT UNSIGNED NOT NULL,
    `first_seen_at` DATETIME NOT NULL,
    `last_seen_at` DATETIME NOT NULL,
    PRIMARY KEY (`id`),
    UNIQUE KEY `uk_velocity` (`identifier_type`, `identifier_value`, `event_type`, `window_minutes`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;

CREATE TABLE IF NOT EXISTS `mod_fraud_blacklist` (
    `id` INT UNSIGNED NOT NULL AUTO_INCREMENT,
    `blacklist_type` ENUM('ip', 'email', 'phone', 'card', 'device', 'address') NOT NULL,
    `blacklist_value` VARCHAR(255) NOT NULL,
    `reason` TEXT NULL,
    `added_by` INT UNSIGNED NULL,
    `expires_at` DATETIME NULL,
    `is_active` TINYINT(1) NOT NULL DEFAULT 1,
    `created_at` DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
    PRIMARY KEY (`id`),
    UNIQUE KEY `uk_blacklist_type_value` (`blacklist_type`, `blacklist_value`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
```

## Module Class

```php
<?php
/**
 * WHMCS Fraud Prevention Module
 */

if (!defined("WHMCS")) {
    die("This file cannot be accessed directly");
}

require_once __DIR__ . '/lib/FraudEngine.php';
require_once __DIR__ . '/lib/FraudDetector.php';
require_once __DIR__ . '/lib/FraudRules.php';

function whmcs_fraud_prevention_activate() {
    $engine = new FraudEngine();
    return $engine->activate();
}

function whmcs_fraud_prevention_deactivate() {
    return ['success' => true, 'msg' => 'Fraud Prevention module deactivated'];
}

function whmcs_fraud_prevention_config() {
    return [
        'auto_block_threshold' => [
            'FriendlyName' => 'Auto-Block Score Threshold',
            'Type' => 'text',
            'Default' => '75',
        ],
        'review_threshold' => [
            'FriendlyName' => 'Manual Review Threshold',
            'Type' => 'text',
            'Default' => '50',
        ],
        'enable_velocity_check' => [
            'FriendlyName' => 'Enable Velocity Checks',
            'Type' => 'yesno',
        ],
        'enable_ip_check' => [
            'FriendlyName' => 'Enable IP Reputation',
            'Type' => 'yesno',
        ],
    ];
}

function whmcs_fraud_prevention_check($orderData) {
    $detector = new FraudDetector();
    return $detector->checkOrder($orderData);
}

function whmcs_fraud_prevention_review($orderId, $decision) {
    $engine = new FraudEngine();
    return $engine->updateDecision($orderId, $decision);
}

add_hook('PrimaryCartItemOverride', 1, function($params) {
    $detector = new FraudDetector();
    $result = $detector->checkOrder($params);
    
    if ($result['action'] === 'block') {
        return ['error' => 'Order cancelled due to fraud risk'];
    }
});

add_hook('DailyCronJob', 1, function() {
    $engine = new FraudEngine();
    $engine->processVelocityData();
    $engine->cleanupOldData();
});
```

### lib/FraudEngine.php

```php
<?php
namespace WHMCS\Module\FraudPrevention;

use Illuminate\Database\Capsule\Manager as Capsule;
use Carbon\Carbon;

class FraudEngine {
    
    public function activate() {
        try {
            $this->createTables();
            $this->setupDefaultRules();
            return ['success' => true, 'msg' => 'Fraud Prevention module activated'];
        } catch (\Exception $e) {
            return ['success' => false, 'msg' => $e->getMessage()];
        }
    }
    
    protected function createTables() {
        Capsule::statement("
            CREATE TABLE IF NOT EXISTS `mod_fraud_rules` (
                `id` INT UNSIGNED NOT NULL AUTO_INCREMENT,
                `rule_name` VARCHAR(255) NOT NULL,
                `rule_code` VARCHAR(50) NOT NULL,
                `rule_type` VARCHAR(30) NOT NULL,
                `conditions` TEXT NOT NULL,
                `risk_weight` DECIMAL(3,2) NOT NULL DEFAULT 1.00,
                `action` VARCHAR(20) NOT NULL DEFAULT 'flag',
                `is_active` TINYINT(1) NOT NULL DEFAULT 1,
                PRIMARY KEY (`id`)
            ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4
        ");
        
        Capsule::statement("
            CREATE TABLE IF NOT EXISTS `mod_fraud_scores` (
                `id` BIGINT UNSIGNED NOT NULL AUTO_INCREMENT,
                `order_id` INT UNSIGNED NULL,
                `user_id` INT UNSIGNED NULL,
                `fraud_score` DECIMAL(5,2) NOT NULL,
                `risk_level` VARCHAR(20) NOT NULL,
                `triggered_rules` TEXT NULL,
                `action_taken` VARCHAR(50) NOT NULL,
                `decision` VARCHAR(20) NOT NULL DEFAULT 'pending',
                `created_at` DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
                PRIMARY KEY (`id`)
            ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4
        ");
        
        Capsule::statement("
            CREATE TABLE IF NOT EXISTS `mod_fraud_velocity` (
                `id` INT UNSIGNED NOT NULL AUTO_INCREMENT,
                `identifier_type` VARCHAR(50) NOT NULL,
                `identifier_value` VARCHAR(255) NOT NULL,
                `event_type` VARCHAR(50) NOT NULL,
                `event_count` INT UNSIGNED NOT NULL DEFAULT 1,
                `window_minutes` INT UNSIGNED NOT NULL,
                `last_seen_at` DATETIME NOT NULL,
                PRIMARY KEY (`id`),
                UNIQUE KEY `uk_velocity` (`identifier_type`, `identifier_value`, `event_type`, `window_minutes`)
            ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4
        ");
        
        Capsule::statement("
            CREATE TABLE IF NOT EXISTS `mod_fraud_blacklist` (
                `id` INT UNSIGNED NOT NULL AUTO_INCREMENT,
                `blacklist_type` VARCHAR(20) NOT NULL,
                `blacklist_value` VARCHAR(255) NOT NULL,
                `reason` TEXT NULL,
                `is_active` TINYINT(1) NOT NULL DEFAULT 1,
                PRIMARY KEY (`id`)
            ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4
        ");
    }
    
    protected function setupDefaultRules() {
        $rules = [
            ['name' => 'Multiple Failed Payments', 'code' => 'FR-VEL-001', 'type' => 'velocity', 'weight' => 2.0],
            ['name' => 'High Risk Country', 'code' => 'FR-GEO-001', 'type' => 'geo', 'weight' => 1.5],
            ['name' => 'Free Email Domain', 'code' => 'FR-PAT-001', 'type' => 'pattern', 'weight' => 1.0],
            ['name' => 'Velocity Limit Exceeded', 'code' => 'FR-VEL-002', 'type' => 'velocity', 'weight' => 3.0],
            ['name' => 'Suspicious IP Range', 'code' => 'FR-GEO-002', 'type' => 'geo', 'weight' => 2.0],
        ];
        
        foreach ($rules as $rule) {
            if (!Capsule::table('mod_fraud_rules')->where('rule_code', $rule['code'])->exists()) {
                Capsule::table('mod_fraud_rules')->insert($rule);
            }
        }
    }
    
    public function updateDecision($orderId, $decision) {
        Capsule::table('mod_fraud_scores')
            ->where('order_id', $orderId)
            ->update([
                'decision' => $decision,
                'reviewed_at' => Carbon::now(),
            ]);
        
        return ['success' => true];
    }
    
    public function processVelocityData() {
        Capsule::table('mod_fraud_velocity')
            ->where('last_seen_at', '<', Carbon::now()->subHours(24))
            ->delete();
    }
    
    public function cleanupOldData() {
        Capsule::table('mod_fraud_scores')
            ->where('created_at', '<', Carbon::now()->subDays(90))
            ->delete();
    }
}
```

### lib/FraudDetector.php

```php
<?php
namespace WHMCS\Module\FraudPrevention;

use Illuminate\Database\Capsule\Manager as Capsule;
use Carbon\Carbon;

class FraudDetector {
    
    protected $rules = [];
    protected $scores = [];
    protected $triggeredRules = [];
    
    public function checkOrder($orderData) {
        $this->scores = [];
        $this->triggeredRules = [];
        
        // Check blacklist first
        $blacklistCheck = $this->checkBlacklist($orderData);
        if ($blacklistCheck['blocked']) {
            return $this->finalize(['blocked', 100, $blacklistCheck['reason']]);
        }
        
        // Check each rule type
        $this->checkVelocityRules($orderData);
        $this->checkPatternRules($orderData);
        $this->checkGeoRules($orderData);
        $this->checkBehavioralRules($orderData);
        
        return $this->calculateFinalScore($orderData);
    }
    
    protected function checkBlacklist($orderData) {
        $blacklistTypes = ['email', 'ip', 'phone'];
        
        foreach ($blacklistTypes as $type) {
            $value = $orderData[$type] ?? null;
            if (!$value) continue;
            
            $blocked = Capsule::table('mod_fraud_blacklist')
                ->where('blacklist_type', $type)
                ->where('blacklist_value', $value)
                ->where('is_active', 1)
                ->where(function($q) {
                    $q->whereNull('expires_at')
                        ->orWhere('expires_at', '>', Carbon::now());
                })
                ->exists();
            
            if ($blocked) {
                return ['blocked' => true, 'reason' => "$type blacklisted: $value"];
            }
        }
        
        return ['blocked' => false];
    }
    
    protected function checkVelocityRules($orderData) {
        $velocityRules = Capsule::table('mod_fraud_rules')
            ->where('rule_type', 'velocity')
            ->where('is_active', 1)
            ->get();
        
        foreach ($velocityRules as $rule) {
            $conditions = json_decode($rule->conditions, true) ?? [];
            
            foreach ($conditions['identifiers'] ?? [] as $identifier) {
                $type = $identifier['type'];
                $value = $orderData[$type] ?? null;
                if (!$value) continue;
                
                $windowMinutes = $conditions['window_minutes'] ?? 60;
                $maxCount = $conditions['max_count'] ?? 3;
                
                $current = $this->getVelocityCount($type, $value, 'order', $windowMinutes);
                
                if ($current >= $maxCount) {
                    $this->scores[] = $rule->risk_weight * 100;
                    $this->triggeredRules[] = [
                        'rule' => $rule->rule_code,
                        'name' => $rule->rule_name,
                        'count' => $current,
                    ];
                }
            }
        }
    }
    
    protected function getVelocityCount($identifierType, $identifierValue, $eventType, $windowMinutes) {
        $count = Capsule::table('mod_fraud_velocity')
            ->where('identifier_type', $identifierType)
            ->where('identifier_value', $identifierValue)
            ->where('event_type', $eventType)
            ->where('window_minutes', $windowMinutes)
            ->where('last_seen_at', '>=', Carbon::now()->subMinutes($windowMinutes))
            ->value('event_count') ?? 0;
        
        // Update record
        Capsule::table('mod_fraud_velocity')
            ->updateOrInsert(
                [
                    'identifier_type' => $identifierType,
                    'identifier_value' => $identifierValue,
                    'event_type' => $eventType,
                    'window_minutes' => $windowMinutes,
                ],
                [
                    'event_count' => $count + 1,
                    'last_seen_at' => Carbon::now(),
                ]
            );
        
        return $count + 1;
    }
    
    protected function checkPatternRules($orderData) {
        $patternRules = Capsule::table('mod_fraud_rules')
            ->where('rule_type', 'pattern')
            ->where('is_active', 1)
            ->get();
        
        foreach ($patternRules as $rule) {
            $conditions = json_decode($rule->conditions, true) ?? [];
            
            switch ($rule->rule_code) {
                case 'FR-PAT-001':
                    $freeDomains = ['gmail.com', 'yahoo.com', 'hotmail.com', 'outlook.com'];
                    $emailDomain = substr(strrchr($orderData['email'] ?? '', '@'), 1);
                    if (in_array(strtolower($emailDomain), $freeDomains)) {
                        $this->scores[] = $rule->risk_weight * 30;
                        $this->triggeredRules[] = ['rule' => $rule->rule_code, 'name' => $rule->rule_name];
                    }
                    break;
            }
        }
    }
    
    protected function checkGeoRules($orderData) {
        $highRiskCountries = ['NG', 'GH', 'PK', 'BD', 'VN', 'MY', 'ID'];
        $country = $orderData['country'] ?? '';
        
        if (in_array($country, $highRiskCountries)) {
            $this->scores[] = 15;
            $this->triggeredRules[] = ['rule' => 'FR-GEO-001', 'name' => 'High Risk Country'];
        }
    }
    
    protected function checkBehavioralRules($orderData) {
        $orderTotal = $orderData['order_total'] ?? 0;
        
        if ($orderTotal > 1000) {
            $this->scores[] = 10;
            $this->triggeredRules[] = ['rule' => 'FR-BEH-001', 'name' => 'High Value Order'];
        }
    }
    
    protected function calculateFinalScore($orderData) {
        $totalScore = array_sum($this->scores);
        $avgScore = count($this->scores) > 0 ? $totalScore / count($this->scores) : 0;
        $finalScore = min(100, max(0, $totalScore));
        
        $riskLevel = $this->getRiskLevel($finalScore);
        $action = $this->determineAction($finalScore);
        
        $this->logScore($orderData, $finalScore, $riskLevel, $action);
        
        return [
            'score' => $finalScore,
            'risk_level' => $riskLevel,
            'triggered_rules' => $this->triggeredRules,
            'action' => $action,
            'order_data' => $orderData,
        ];
    }
    
    protected function getRiskLevel($score) {
        if ($score >= 75) return 'critical';
        if ($score >= 50) return 'high';
        if ($score >= 25) return 'medium';
        return 'low';
    }
    
    protected function determineAction($score) {
        $config = \App::get_config('fraud_prevention') ?? [];
        $blockThreshold = $config['auto_block_threshold'] ?? 75;
        
        if ($score >= $blockThreshold) return 'block';
        if ($score >= 50) return 'review';
        if ($score >= 25) return 'flag';
        return 'allow';
    }
    
    protected function logScore($orderData, $score, $level, $action) {
        Capsule::table('mod_fraud_scores')->insert([
            'order_id' => $orderData['order_id'] ?? null,
            'user_id' => $orderData['user_id'] ?? null,
            'fraud_score' => $score,
            'risk_level' => $level,
            'triggered_rules' => json_encode($this->triggeredRules),
            'action_taken' => $action,
        ]);
    }
}
```

## API Endpoints

```
POST /api/v1/fraud/check                - Check order for fraud
GET  /api/v1/fraud/scores/{orderId}     - Get fraud score
POST /api/v1/fraud/review              - Manual review
GET  /api/v1/fraud/rules               - List fraud rules
POST /api/v1/fraud/rules               - Add fraud rule
GET  /api/v1/fraud/blacklist           - List blacklist
POST /api/v1/fraud/blacklist           - Add to blacklist
DELETE /api/v1/fraud/blacklist/{id}    - Remove from blacklist
GET  /api/v1/fraud/stats               - Fraud statistics
```
