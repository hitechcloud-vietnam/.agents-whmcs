# WHMCS Usage Meter DevKit

## Overview

Usage metering system for WHMCS enabling accurate measurement and tracking of resource consumption for billing purposes.

## Features

- Usage tracking
- Metering
- Billing integration
- Rate calculation
- Usage reports
- Threshold alerts
- Export functionality

## Module Files

```php
<?php
/**
 * WHMCS Usage Meter Module
 */

if (!defined("WHMCS")) {
    die("This file cannot be accessed directly");
}

require_once __DIR__ . '/lib/UsageMeter.php';

function whmcs_usage_meter_activate() {
    $meter = new UsageMeter();
    return $meter->activate();
}

function whmcs_usage_meter_record($userId, $resourceType, $amount, $timestamp = null) {
    $meter = new UsageMeter();
    return $meter->recordUsage($userId, $resourceType, $amount, $timestamp);
}

function whmcs_usage_meter_get_usage($userId, $period = 'monthly') {
    $meter = new UsageMeter();
    return $meter->getUsage($userId, $period);
}
```

### lib/UsageMeter.php

```php
<?php
namespace WHMCS\Module\UsageMeter;

use Illuminate\Database\Capsule\Manager as Capsule;
use Carbon\Carbon;

class UsageMeter {
    
    public function activate() {
        try {
            $this->createTables();
            return ['success' => true, 'msg' => 'Usage Meter module activated'];
        } catch (\Exception $e) {
            return ['success' => false, 'msg' => $e->getMessage()];
        }
    }
    
    protected function createTables() {
        Capsule::statement("
            CREATE TABLE IF NOT EXISTS `mod_usage_records` (
                `id` BIGINT UNSIGNED NOT NULL AUTO_INCREMENT,
                `user_id` INT UNSIGNED NOT NULL,
                `resource_type` VARCHAR(50) NOT NULL,
                `amount` DECIMAL(15,4) NOT NULL,
                `unit` VARCHAR(20) NOT NULL,
                `rate` DECIMAL(10,4) NULL,
                `cost` DECIMAL(12,2) NULL,
                `recorded_at` DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
                PRIMARY KEY (`id`),
                INDEX `idx_user_resource` (`user_id`, `resource_type`, `recorded_at`)
            ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4
        ");
        
        Capsule::statement("
            CREATE TABLE IF NOT EXISTS `mod_usage_rates` (
                `id` INT UNSIGNED NOT NULL AUTO_INCREMENT,
                `resource_type` VARCHAR(50) NOT NULL,
                `tier_name` VARCHAR(100) NOT NULL,
                `rate_per_unit` DECIMAL(10,4) NOT NULL,
                `min_units` DECIMAL(15,4) DEFAULT 0,
                `max_units` DECIMAL(15,4) NULL,
                `effective_from` DATE NOT NULL,
                `effective_to` DATE NULL,
                `is_active` TINYINT(1) NOT NULL DEFAULT 1,
                PRIMARY KEY (`id`)
            ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4
        ");
    }
    
    public function recordUsage($userId, $resourceType, $amount, $timestamp = null) {
        $rate = $this->getRate($resourceType, $amount);
        $cost = $amount * $rate;
        
        $id = Capsule::table('mod_usage_records')->insertGetId([
            'user_id' => $userId,
            'resource_type' => $resourceType,
            'amount' => $amount,
            'unit' => $this->getUnit($resourceType),
            'rate' => $rate,
            'cost' => $cost,
            'recorded_at' => $timestamp ?? Carbon::now(),
        ]);
        
        return ['success' => true, 'record_id' => $id, 'cost' => $cost];
    }
    
    protected function getUnit($resourceType) {
        $units = [
            'api_calls' => 'calls',
            'storage' => 'GB',
            'bandwidth' => 'GB',
            'compute' => 'hours',
            'emails' => 'emails',
        ];
        
        return $units[$resourceType] ?? 'units';
    }
    
    protected function getRate($resourceType, $amount) {
        $rate = Capsule::table('mod_usage_rates')
            ->where('resource_type', $resourceType)
            ->where('is_active', 1)
            ->where('effective_from', '<=', Carbon::today())
            ->where(function($q) {
                $q->whereNull('effective_to')
                    ->orWhere('effective_to', '>=', Carbon::today());
            })
            ->orderBy('min_units', 'desc')
            ->first();
        
        if ($rate) {
            // Apply tiered pricing
            if ($rate->max_units && $amount > $rate->max_units) {
                return $rate->rate_per_unit * 0.9; // Discount for overage
            }
            return $rate->rate_per_unit;
        }
        
        // Default rates
        return 0.01;
    }
    
    public function getUsage($userId, $period = 'monthly') {
        $startDate = $this->getPeriodStart($period);
        
        $records = Capsule::table('mod_usage_records')
            ->where('user_id', $userId)
            ->where('recorded_at', '>=', $startDate)
            ->get();
        
        $grouped = $records->groupBy('resource_type');
        $summary = [];
        
        foreach ($grouped as $type => $items) {
            $summary[$type] = [
                'total_amount' => $items->sum('amount'),
                'total_cost' => $items->sum('cost'),
                'record_count' => $items->count(),
            ];
        }
        
        return [
            'period' => $period,
            'start_date' => $startDate,
            'end_date' => Carbon::now(),
            'usage' => $summary,
            'total_cost' => $records->sum('cost'),
        ];
    }
    
    protected function getPeriodStart($period) {
        switch ($period) {
            case 'daily':
                return Carbon::today();
            case 'weekly':
                return Carbon::now()->startOfWeek();
            case 'monthly':
                return Carbon::now()->startOfMonth();
            case 'yearly':
                return Carbon::now()->startOfYear();
            default:
                return Carbon::now()->startOfMonth();
        }
    }
}
```

## API Endpoints

```
POST /api/v1/usage/record               - Record usage
GET  /api/v1/usage/{userId}            - Get usage
GET  /api/v1/usage/{userId}/export     - Export usage
```
