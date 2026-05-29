# WHMCS Cost Allocation DevKit

## Overview

Cost allocation and chargeback system for WHMCS enabling accurate cost tracking, department allocation, and internal billing.

## Module Files

```php
<?php
/**
 * WHMCS Cost Allocation Module
 */

if (!defined("WHMCS")) {
    die("This file cannot be accessed directly");
}

require_once __DIR__ . '/lib/CostAllocation.php';

function whmcs_cost_allocation_activate() {
    $allocation = new CostAllocation();
    return $allocation->activate();
}

function whmcs_cost_allocation_allocate($costCenterId, $amount, $description) {
    $allocation = new CostAllocation();
    return $allocation->allocateCost($costCenterId, $amount, $description);
}

function whmcs_cost_allocation_report($period = 'monthly') {
    $allocation = new CostAllocation();
    return $allocation->generateReport($period);
}
```

### lib/CostAllocation.php

```php
<?php
namespace WHMCS\Module\CostAllocation;

use Illuminate\Database\Capsule\Manager as Capsule;
use Carbon\Carbon;

class CostAllocation {
    
    public function activate() {
        try {
            $this->createTables();
            $this->createDefaultCostCenters();
            return ['success' => true, 'msg' => 'Cost Allocation module activated'];
        } catch (\Exception $e) {
            return ['success' => false, 'msg' => $e->getMessage()];
        }
    }
    
    protected function createTables() {
        Capsule::statement("
            CREATE TABLE IF NOT EXISTS `mod_cost_centers` (
                `id` INT UNSIGNED NOT NULL AUTO_INCREMENT,
                `center_name` VARCHAR(255) NOT NULL,
                `center_code` VARCHAR(50) NOT NULL,
                `department` VARCHAR(100) NULL,
                `manager_id` INT UNSIGNED NULL,
                `budget_limit` DECIMAL(15,2) NULL,
                `is_active` TINYINT(1) NOT NULL DEFAULT 1,
                PRIMARY KEY (`id`)
            ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4
        ");
        
        Capsule::statement("
            CREATE TABLE IF NOT EXISTS `mod_cost_allocations` (
                `id` BIGINT UNSIGNED NOT NULL AUTO_INCREMENT,
                `cost_center_id` INT UNSIGNED NOT NULL,
                `allocation_type` ENUM('service', 'usage', 'fixed', 'percentage') NOT NULL,
                `reference_id` VARCHAR(100) NULL,
                `amount` DECIMAL(15,2) NOT NULL,
                `description` VARCHAR(255) NULL,
                `allocated_at` DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
                `period_start` DATE NOT NULL,
                `period_end` DATE NOT NULL,
                PRIMARY KEY (`id`)
            ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4
        ");
    }
    
    protected function createDefaultCostCenters() {
        $centers = [
            ['name' => 'Engineering', 'code' => 'ENG'],
            ['name' => 'Sales', 'code' => 'SALES'],
            ['name' => 'Marketing', 'code' => 'MKT'],
            ['name' => 'Operations', 'code' => 'OPS'],
        ];
        
        foreach ($centers as $center) {
            if (!Capsule::table('mod_cost_centers')->where('center_code', $center['code'])->exists()) {
                Capsule::table('mod_cost_centers')->insert($center);
            }
        }
    }
    
    public function allocateCost($costCenterId, $amount, $description, $type = 'fixed') {
        $id = Capsule::table('mod_cost_allocations')->insertGetId([
            'cost_center_id' => $costCenterId,
            'allocation_type' => $type,
            'amount' => $amount,
            'description' => $description,
            'period_start' => Carbon::now()->startOfMonth()->toDateString(),
            'period_end' => Carbon::now()->endOfMonth()->toDateString(),
        ]);
        
        return ['success' => true, 'allocation_id' => $id];
    }
    
    public function generateReport($period = 'monthly') {
        $startDate = Carbon::now()->startOfMonth()->toDateString();
        
        $costCenters = Capsule::table('mod_cost_centers')->where('is_active', 1)->get();
        
        $report = [];
        foreach ($costCenters as $center) {
            $allocations = Capsule::table('mod_cost_allocations')
                ->where('cost_center_id', $center->id)
                ->where('period_start', '>=', $startDate)
                ->get();
            
            $total = $allocations->sum('amount');
            
            $report[] = [
                'center' => $center->center_name,
                'code' => $center->center_code,
                'total_allocated' => $total,
                'budget' => $center->budget_limit,
                'budget_used_percent' => $center->budget_limit > 0 ? ($total / $center->budget_limit) * 100 : 0,
            ];
        }
        
        return $report;
    }
}
```

## API Endpoints

```
POST /api/v1/cost-allocation/allocate    - Allocate cost
GET  /api/v1/cost-allocation/centers    - List cost centers
GET  /api/v1/cost-allocation/report      - Get report
```
