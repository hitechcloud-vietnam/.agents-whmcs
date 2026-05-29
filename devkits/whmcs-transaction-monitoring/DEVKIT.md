# WHMCS Transaction Monitoring DevKit

## Overview

Real-time transaction monitoring system for WHMCS enabling suspicious activity detection, pattern analysis, and fraud alerting.

## Features

- Real-time monitoring
- Pattern detection
- Anomaly scoring
- Alert generation
- Transaction scoring
- Risk assessment
- Suspicious activity logging

## Module Files

```php
<?php
/**
 * WHMCS Transaction Monitoring Module
 */

if (!defined("WHMCS")) {
    die("This file cannot be accessed directly");
}

require_once __DIR__ . '/lib/TransactionMonitor.php';

function whmcs_transaction_monitoring_activate() {
    $monitor = new TransactionMonitor();
    return $monitor->activate();
}

function whmcs_transaction_monitor($transactionId) {
    $monitor = new TransactionMonitor();
    return $monitor->analyzeTransaction($transactionId);
}
```

### lib/TransactionMonitor.php

```php
<?php
namespace WHMCS\Module\TransactionMonitoring;

use Illuminate\Database\Capsule\Manager as Capsule;
use Carbon\Carbon;

class TransactionMonitor {
    
    public function activate() {
        try {
            $this->createTables();
            return ['success' => true, 'msg' => 'Transaction Monitoring module activated'];
        } catch (\Exception $e) {
            return ['success' => false, 'msg' => $e->getMessage()];
        }
    }
    
    protected function createTables() {
        Capsule::statement("
            CREATE TABLE IF NOT EXISTS `mod_transaction_logs` (
                `id` BIGINT UNSIGNED NOT NULL AUTO_INCREMENT,
                `transaction_id` VARCHAR(64) NOT NULL,
                `user_id` INT UNSIGNED NOT NULL,
                `amount` DECIMAL(15,2) NOT NULL,
                `currency` VARCHAR(3) NOT NULL DEFAULT 'USD',
                `transaction_type` VARCHAR(50) NOT NULL,
                `risk_score` DECIMAL(5,2) DEFAULT 0.00,
                `risk_factors` JSON NULL,
                `status` ENUM('pending', 'approved', 'flagged', 'blocked') NOT NULL DEFAULT 'pending',
                `created_at` DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
                PRIMARY KEY (`id`)
            ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4
        ");
        
        Capsule::statement("
            CREATE TABLE IF NOT EXISTS `mod_transaction_patterns` (
                `id` INT UNSIGNED NOT NULL AUTO_INCREMENT,
                `pattern_name` VARCHAR(255) NOT NULL,
                `pattern_type` VARCHAR(50) NOT NULL,
                `conditions` JSON NOT NULL,
                `risk_weight` DECIMAL(3,2) NOT NULL DEFAULT 1.00,
                `is_active` TINYINT(1) NOT NULL DEFAULT 1,
                PRIMARY KEY (`id`)
            ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4
        ");
    }
    
    public function analyzeTransaction($transactionId) {
        $transaction = Capsule::table('tblaccounts')->where('id', $transactionId)->first();
        
        if (!$transaction) {
            return ['error' => 'Transaction not found'];
        }
        
        $riskScore = 0;
        $riskFactors = [];
        
        // Check velocity
        $recentCount = Capsule::table('tblaccounts')
            ->where('userid', $transaction->userid)
            ->where('created_at', '>=', Carbon::now()->subHours(24))
            ->count();
        
        if ($recentCount > 5) {
            $riskScore += 30;
            $riskFactors[] = 'high_velocity';
        }
        
        // Check amount anomalies
        $avgAmount = Capsule::table('tblaccounts')
            ->where('userid', $transaction->userid)
            ->avg('amount') ?? 0;
        
        if ($transaction->amount > $avgAmount * 5) {
            $riskScore += 40;
            $riskFactors[] = 'amount_anomaly';
        }
        
        // Check for new payment method
        $newPaymentMethod = Capsule::table('mod_transaction_logs')
            ->where('user_id', $transaction->userid)
            ->where('created_at', '>=', Carbon::now()->subDays(7))
            ->count() == 0;
        
        if ($newPaymentMethod) {
            $riskScore += 20;
            $riskFactors[] = 'new_payment_method';
        }
        
        $status = $riskScore > 70 ? 'blocked' : ($riskScore > 40 ? 'flagged' : 'approved');
        
        Capsule::table('mod_transaction_logs')->insert([
            'transaction_id' => 'TXN-' . $transactionId,
            'user_id' => $transaction->userid,
            'amount' => $transaction->amount,
            'currency' => $transaction->currency ?? 'USD',
            'transaction_type' => 'payment',
            'risk_score' => $riskScore,
            'risk_factors' => json_encode($riskFactors),
            'status' => $status,
        ]);
        
        return [
            'risk_score' => $riskScore,
            'risk_factors' => $riskFactors,
            'status' => $status,
        ];
    }
}
```

## API Endpoints

```
POST /api/v1/transaction-monitor        - Monitor transaction
GET  /api/v1/transaction-monitor/{id}    - Get analysis
GET  /api/v1/transaction-monitor/alerts - Get alerts
```
