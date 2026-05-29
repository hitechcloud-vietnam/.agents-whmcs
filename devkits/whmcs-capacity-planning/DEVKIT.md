# WHMCS Capacity Planning DevKit

## Overview

A capacity planning and forecasting system for WHMCS that helps predict resource needs, plan infrastructure scaling, and optimize resource allocation based on historical usage patterns.

## Features

- Historical usage analysis
- Demand forecasting
- Capacity trend visualization
- Resource bottleneck identification
- Scaling recommendations
-_capacity planning reports

## Database Schema

```sql
CREATE TABLE IF NOT EXISTS `mod_capacity_pools` (
    `id` INT UNSIGNED NOT NULL AUTO_INCREMENT,
    `pool_name` VARCHAR(255) NOT NULL,
    `pool_type` VARCHAR(100) NOT NULL,
    `current_capacity` DECIMAL(15,4) NOT NULL DEFAULT 0,
    `current_usage` DECIMAL(15,4) NOT NULL DEFAULT 0,
    `peak_usage` DECIMAL(15,4) NOT NULL DEFAULT 0,
    `average_usage` DECIMAL(15,4) NOT NULL DEFAULT 0,
    `unit` VARCHAR(50) NOT NULL DEFAULT 'units',
    `created_at` DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
    `updated_at` DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    PRIMARY KEY (`id`),
    INDEX `idx_pool_type` (`pool_type`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;

CREATE TABLE IF NOT EXISTS `mod_capacity_snapshots` (
    `id` BIGINT UNSIGNED NOT NULL AUTO_INCREMENT,
    `pool_id` INT UNSIGNED NOT NULL,
    `usage_value` DECIMAL(15,4) NOT NULL,
    `utilization_rate` DECIMAL(5,2) NOT NULL DEFAULT 0.00,
    `active_services` INT UNSIGNED NOT NULL DEFAULT 0,
    `new_services_count` INT UNSIGNED NOT NULL DEFAULT 0,
    `cancelled_services_count` INT UNSIGNED NOT NULL DEFAULT 0,
    `snapshot_date` DATE NOT NULL,
    `created_at` DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
    PRIMARY KEY (`id`),
    UNIQUE KEY `unique_pool_date` (`pool_id`, `snapshot_date`),
    INDEX `idx_snapshot_date` (`snapshot_date`),
    CONSTRAINT `fk_snapshot_pool` FOREIGN KEY (`pool_id`) REFERENCES `mod_capacity_pools`(`id`) ON DELETE CASCADE
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;

CREATE TABLE IF NOT EXISTS `mod_capacity_predictions` (
    `id` INT UNSIGNED NOT NULL AUTO_INCREMENT,
    `pool_id` INT UNSIGNED NOT NULL,
    `prediction_date` DATE NOT NULL,
    `predicted_usage` DECIMAL(15,4) NOT NULL,
    `predicted_peak` DECIMAL(15,4) NOT NULL,
    `confidence_level` DECIMAL(5,2) NOT NULL DEFAULT 0.00,
    `prediction_model` VARCHAR(100) NOT NULL DEFAULT 'linear',
    `horizon_days` INT UNSIGNED NOT NULL DEFAULT 30,
    `created_at` DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
    PRIMARY KEY (`id`),
    INDEX `idx_pool_prediction` (`pool_id`, `prediction_date`),
    CONSTRAINT `fk_prediction_pool` FOREIGN KEY (`pool_id`) REFERENCES `mod_capacity_pools`(`id`) ON DELETE CASCADE
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;

CREATE TABLE IF NOT EXISTS `mod_capacity_alerts` (
    `id` INT UNSIGNED NOT NULL AUTO_INCREMENT,
    `pool_id` INT UNSIGNED NOT NULL,
    `alert_type` ENUM('threshold', 'bottleneck', 'scaling', 'forecast') NOT NULL,
    `threshold_percentage` DECIMAL(5,2) NULL,
    `current_utilization` DECIMAL(5,2) NULL,
    `message` TEXT NOT NULL,
    `severity` ENUM('info', 'warning', 'critical') NOT NULL DEFAULT 'warning',
    `is_resolved` TINYINT(1) NOT NULL DEFAULT 0,
    `resolved_at` DATETIME NULL,
    `created_at` DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
    PRIMARY KEY (`id`),
    INDEX `idx_pool_resolved` (`pool_id`, `is_resolved`),
    INDEX `idx_severity` (`severity`),
    CONSTRAINT `fk_alert_pool` FOREIGN KEY (`pool_id`) REFERENCES `mod_capacity_pools`(`id`) ON DELETE CASCADE
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
```

## Module Class

### capacity_planning.php

```php
<?php
/**
 * WHMCS Capacity Planning Module
 *
 * @package     WHMCS\Module\CapacityPlanning
 */

if (!defined("WHMCS")) {
    die("This file cannot be accessed directly");
}

require_once __DIR__ . '/lib/CapacityPlanner.php';
require_once __DIR__ . '/lib/ForecastEngine.php';
require_once __DIR__ . '/lib/AlertManager.php';

use WHMCS\Module\CapacityPlanning\CapacityPlanner;
use WHMCS\Module\CapacityPlanning\ForecastEngine;
use WHMCS\Module\CapacityPlanning\AlertManager;

/**
 * Activate Module
 */
function whmcs_capacity_planning_activate() {
    $planner = new CapacityPlanner();
    return $planner->activate();
}

/**
 * Deactivate Module
 */
function whmcs_capacity_planning_deactivate() {
    $planner = new CapacityPlanner();
    return $planner->deactivate();
}

/**
 * Module Configuration
 */
function whmcs_capacity_planning_config() {
    return [
        'name' => [
            'FriendlyName' => 'Capacity Planning',
            'Type' => 'System',
            'Value' => 'Capacity Planning Module v1.0.0',
        ],
        'forecast_horizon' => [
            'FriendlyName' => 'Forecast Horizon (Days)',
            'Type' => 'text',
            'Size' => '10',
            'Default' => '30',
        ],
        'alert_threshold_warning' => [
            'FriendlyName' => 'Warning Alert Threshold (%)',
            'Type' => 'text',
            'Size' => '10',
            'Default' => '75',
        ],
        'alert_threshold_critical' => [
            'FriendlyName' => 'Critical Alert Threshold (%)',
            'Type' => 'text',
            'Size' => '10',
            'Default' => '90',
        ],
        'snapshot_frequency' => [
            'FriendlyName' => 'Snapshot Frequency',
            'Type' => 'dropdown',
            'Options' => [
                'daily' => 'Daily',
                'hourly' => 'Hourly',
                'weekly' => 'Weekly',
            ],
            'Default' => 'daily',
        ],
    ];
}

/**
 * Get Capacity Overview
 */
function whmcs_capacity_planning_overview() {
    $planner = new CapacityPlanner();
    return $planner->getOverview();
}

/**
 * Get Pool Forecast
 */
function whmcs_capacity_planning_forecast($poolId, $horizonDays = 30) {
    $forecast = new ForecastEngine();
    return $forecast->generateForecast($poolId, $horizonDays);
}

/**
 * Get Scalability Recommendations
 */
function whmcs_capacity_planning_recommendations() {
    $planner = new CapacityPlanner();
    return $planner->getScalingRecommendations();
}

// Cron Hooks
add_hook('DailyCronJob', 1, function() {
    $planner = new CapacityPlanner();
    $planner->takeDailySnapshot();
    $planner->checkAlerts();
    $planner->runForecastUpdate();
});

add_hook('HourlyCronJob', 1, function() {
    $planner = new CapacityPlanner();
    $planner->updateCurrentUsage();
});
```

### lib/CapacityPlanner.php

```php
<?php
/**
 * Capacity Planner
 */

namespace WHMCS\Module\CapacityPlanning;

use Illuminate\Database\Capsule\Manager as Capsule;
use Carbon\Carbon;

class CapacityPlanner {
    
    protected $version = '1.0.0';
    
    public function activate() {
        try {
            $this->createTables();
            $this->initializePools();
            return ['success' => true, 'msg' => 'Capacity Planning module activated'];
        } catch (\Exception $e) {
            return ['success' => false, 'msg' => $e->getMessage()];
        }
    }
    
    public function deactivate() {
        return ['success' => true, 'msg' => 'Module deactivated'];
    }
    
    protected function createTables() {
        Capsule::statement("
            CREATE TABLE IF NOT EXISTS `mod_capacity_pools` (
                `id` INT UNSIGNED NOT NULL AUTO_INCREMENT,
                `pool_name` VARCHAR(255) NOT NULL,
                `pool_type` VARCHAR(100) NOT NULL,
                `current_capacity` DECIMAL(15,4) NOT NULL DEFAULT 0,
                `current_usage` DECIMAL(15,4) NOT NULL DEFAULT 0,
                `peak_usage` DECIMAL(15,4) NOT NULL DEFAULT 0,
                `average_usage` DECIMAL(15,4) NOT NULL DEFAULT 0,
                `unit` VARCHAR(50) NOT NULL DEFAULT 'units',
                `created_at` DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
                `updated_at` DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
                PRIMARY KEY (`id`)
            ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4
        ");
        
        Capsule::statement("
            CREATE TABLE IF NOT EXISTS `mod_capacity_snapshots` (
                `id` BIGINT UNSIGNED NOT NULL AUTO_INCREMENT,
                `pool_id` INT UNSIGNED NOT NULL,
                `usage_value` DECIMAL(15,4) NOT NULL,
                `utilization_rate` DECIMAL(5,2) NOT NULL DEFAULT 0.00,
                `active_services` INT UNSIGNED NOT NULL DEFAULT 0,
                `new_services_count` INT UNSIGNED NOT NULL DEFAULT 0,
                `cancelled_services_count` INT UNSIGNED NOT NULL DEFAULT 0,
                `snapshot_date` DATE NOT NULL,
                `created_at` DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
                PRIMARY KEY (`id`),
                UNIQUE KEY `unique_pool_date` (`pool_id`, `snapshot_date`)
            ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4
        ");
        
        Capsule::statement("
            CREATE TABLE IF NOT EXISTS `mod_capacity_predictions` (
                `id` INT UNSIGNED NOT NULL AUTO_INCREMENT,
                `pool_id` INT UNSIGNED NOT NULL,
                `prediction_date` DATE NOT NULL,
                `predicted_usage` DECIMAL(15,4) NOT NULL,
                `predicted_peak` DECIMAL(15,4) NOT NULL,
                `confidence_level` DECIMAL(5,2) NOT NULL DEFAULT 0.00,
                `prediction_model` VARCHAR(100) NOT NULL DEFAULT 'linear',
                `horizon_days` INT UNSIGNED NOT NULL DEFAULT 30,
                `created_at` DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
                PRIMARY KEY (`id`)
            ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4
        ");
        
        Capsule::statement("
            CREATE TABLE IF NOT EXISTS `mod_capacity_alerts` (
                `id` INT UNSIGNED NOT NULL AUTO_INCREMENT,
                `pool_id` INT UNSIGNED NOT NULL,
                `alert_type` ENUM('threshold', 'bottleneck', 'scaling', 'forecast') NOT NULL,
                `threshold_percentage` DECIMAL(5,2) NULL,
                `current_utilization` DECIMAL(5,2) NULL,
                `message` TEXT NOT NULL,
                `severity` ENUM('info', 'warning', 'critical') NOT NULL DEFAULT 'warning',
                `is_resolved` TINYINT(1) NOT NULL DEFAULT 0,
                `resolved_at` DATETIME NULL,
                `created_at` DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
                PRIMARY KEY (`id`)
            ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4
        ");
    }
    
    protected function initializePools() {
        $defaultPools = [
            ['name' => 'Compute Resources', 'type' => 'compute', 'unit' => 'cores'],
            ['name' => 'Storage Capacity', 'type' => 'storage', 'unit' => 'GB'],
            ['name' => 'Bandwidth', 'type' => 'bandwidth', 'unit' => 'TB'],
            ['name' => 'Memory', 'type' => 'memory', 'unit' => 'GB'],
        ];
        
        foreach ($defaultPools as $pool) {
            $exists = Capsule::table('mod_capacity_pools')->where('pool_name', $pool['name'])->exists();
            if (!$exists) {
                Capsule::table('mod_capacity_pools')->insert($pool);
            }
        }
    }
    
    public function getOverview() {
        $pools = Capsule::table('mod_capacity_pools')->get();
        $overview = [];
        
        foreach ($pools as $pool) {
            $utilization = $pool->current_capacity > 0 
                ? ($pool->current_usage / $pool->current_capacity) * 100 
                : 0;
            
            $overview[] = [
                'id' => $pool->id,
                'name' => $pool->pool_name,
                'type' => $pool->pool_type,
                'current_capacity' => $pool->current_capacity,
                'current_usage' => $pool->current_usage,
                'peak_usage' => $pool->peak_usage,
                'utilization_rate' => round($utilization, 2),
                'unit' => $pool->unit,
                'status' => $this->getStatusFromUtilization($utilization),
            ];
        }
        
        return $overview;
    }
    
    protected function getStatusFromUtilization($utilization) {
        if ($utilization >= 90) return 'critical';
        if ($utilization >= 75) return 'warning';
        return 'healthy';
    }
    
    public function takeDailySnapshot() {
        $pools = Capsule::table('mod_capacity_pools')->get();
        $today = Carbon::today();
        
        foreach ($pools as $pool) {
            $existingSnapshot = Capsule::table('mod_capacity_snapshots')
                ->where('pool_id', $pool->id)
                ->where('snapshot_date', $today)
                ->exists();
            
            if (!$existingSnapshot) {
                $activeServices = Capsule::table('tblhosting')
                    ->where('domainstatus', 'Active')
                    ->count();
                
                $newServices = Capsule::table('tblhosting')
                    ->whereDate('createddate', $today)
                    ->where('domainstatus', 'Active')
                    ->count();
                
                $cancelledServices = Capsule::table('tblhosting')
                    ->whereDate('terminateddate', $today)
                    ->where('domainstatus', 'Terminated')
                    ->count();
                
                $utilization = $pool->current_capacity > 0 
                    ? ($pool->current_usage / $pool->current_capacity) * 100 
                    : 0;
                
                Capsule::table('mod_capacity_snapshots')->insert([
                    'pool_id' => $pool->id,
                    'usage_value' => $pool->current_usage,
                    'utilization_rate' => $utilization,
                    'active_services' => $activeServices,
                    'new_services_count' => $newServices,
                    'cancelled_services_count' => $cancelledServices,
                    'snapshot_date' => $today,
                ]);
            }
        }
    }
    
    public function updateCurrentUsage() {
        $this->updatePoolUsage('compute', $this->calculateComputeUsage());
        $this->updatePoolUsage('storage', $this->calculateStorageUsage());
        $this->updatePoolUsage('bandwidth', $this->calculateBandwidthUsage());
        $this->updatePoolUsage('memory', $this->calculateMemoryUsage());
    }
    
    protected function calculateComputeUsage() {
        return Capsule::table('tblhosting')
            ->where('domainstatus', 'Active')
            ->join('tblproducts', 'tblhosting.packageid', '=', 'tblproducts.id')
            ->sum('tblproducts.qty');
    }
    
    protected function calculateStorageUsage() {
        return Capsule::table('tblhosting')
            ->where('domainstatus', 'Active')
            ->sum('diskusage') / 1024;
    }
    
    protected function calculateBandwidthUsage() {
        $currentMonth = Carbon::now()->startOfMonth();
        return Capsule::table('tblbillableitems')
            ->where('createddate', '>=', $currentMonth)
            ->sum('amount');
    }
    
    protected function calculateMemoryUsage() {
        return Capsule::table('tblhosting')
            ->where('domainstatus', 'Active')
            ->sum(' bandwidthref'?) / 1024;
    }
    
    protected function updatePoolUsage($type, $usage) {
        Capsule::table('mod_capacity_pools')
            ->where('pool_type', $type)
            ->update([
                'current_usage' => $usage,
                'peak_usage' => Capsule::raw("GREATEST(peak_usage, $usage"),
                'updated_at' => Carbon::now(),
            ]);
    }
    
    public function checkAlerts() {
        $pools = Capsule::table('mod_capacity_pools')->get();
        $config = json_decode(\App::get_config('capacity_planning'), true) ?? [];
        
        $warningThreshold = $config['alert_threshold_warning'] ?? 75;
        $criticalThreshold = $config['alert_threshold_critical'] ?? 90;
        
        foreach ($pools as $pool) {
            $utilization = $pool->current_capacity > 0 
                ? ($pool->current_usage / $pool->current_capacity) * 100 
                : 0;
            
            if ($utilization >= $criticalThreshold) {
                $this->createAlert($pool->id, 'threshold', $criticalThreshold, $utilization, 
                    "Critical: {$pool->pool_name} at {$utilization}% utilization", 'critical');
            } elseif ($utilization >= $warningThreshold) {
                $this->createAlert($pool->id, 'threshold', $warningThreshold, $utilization,
                    "Warning: {$pool->pool_name} at {$utilization}% utilization", 'warning');
            }
        }
    }
    
    protected function createAlert($poolId, $alertType, $threshold, $utilization, $message, $severity) {
        $exists = Capsule::table('mod_capacity_alerts')
            ->where('pool_id', $poolId)
            ->where('alert_type', $alertType)
            ->where('is_resolved', 0)
            ->exists();
        
        if (!$exists) {
            Capsule::table('mod_capacity_alerts')->insert([
                'pool_id' => $poolId,
                'alert_type' => $alertType,
                'threshold_percentage' => $threshold,
                'current_utilization' => $utilization,
                'message' => $message,
                'severity' => $severity,
                'created_at' => Carbon::now(),
            ]);
        }
    }
    
    public function getScalingRecommendations() {
        $recommendations = [];
        $pools = Capsule::table('mod_capacity_pools')->get();
        
        foreach ($pools as $pool) {
            $utilization = $pool->current_capacity > 0 
                ? ($pool->current_usage / $pool->current_capacity) * 100 
                : 0;
            
            if ($utilization > 75) {
                $projectedNeed = $pool->current_usage * 1.2;
                $recommendedCapacity = ceil($projectedNeed / 0.7);
                
                $recommendations[] = [
                    'pool_id' => $pool->id,
                    'pool_name' => $pool->pool_name,
                    'pool_type' => $pool->pool_type,
                    'current_utilization' => round($utilization, 2),
                    'recommended_capacity' => $recommendedCapacity,
                    'current_capacity' => $pool->current_capacity,
                    'capacity_increase' => $recommendedCapacity - $pool->current_capacity,
                    'urgency' => $utilization > 90 ? 'high' : 'medium',
                ];
            }
        }
        
        return $recommendations;
    }
    
    public function runForecastUpdate() {
        $forecast = new ForecastEngine();
        $pools = Capsule::table('mod_capacity_pools')->get();
        
        foreach ($pools as $pool) {
            $forecast->generateForecast($pool->id, 30);
        }
    }
}
```

### lib/ForecastEngine.php

```php
<?php
/**
 * Forecast Engine
 * Predictive capacity analysis
 */

namespace WHMCS\Module\CapacityPlanning;

use Illuminate\Database\Capsule\Manager as Capsule;
use Carbon\Carbon;

class ForecastEngine {
    
    /**
     * Generate forecast for a pool
     */
    public function generateForecast($poolId, $horizonDays = 30) {
        $snapshots = Capsule::table('mod_capacity_snapshots')
            ->where('pool_id', $poolId)
            ->where('snapshot_date', '>=', Carbon::now()->subDays(90))
            ->orderBy('snapshot_date', 'asc')
            ->get();
        
        if ($snapshots->count() < 7) {
            return [
                'success' => false,
                'msg' => 'Insufficient historical data for forecasting',
                'predictions' => [],
            ];
        }
        
        $predictions = $this->calculatePredictions($poolId, $snapshots, $horizonDays);
        $this->storePredictions($poolId, $predictions);
        
        return [
            'success' => true,
            'predictions' => $predictions,
            'model' => 'linear_regression',
            'confidence' => $this->calculateConfidence($snapshots),
        ];
    }
    
    protected function calculatePredictions($poolId, $snapshots, $horizonDays) {
        $values = $snapshots->pluck('usage_value')->toArray();
        $dates = $snapshots->pluck('snapshot_date')->toArray();
        
        // Simple linear regression
        $n = count($values);
        $sumX = $sumY = $sumXY = $sumX2 = 0;
        
        foreach ($values as $i => $value) {
            $sumX += $i;
            $sumY += $value;
            $sumXY += $i * $value;
            $sumX2 += $i * $i;
        }
        
        $slope = ($n * $sumXY - $sumX * $sumY) / ($n * $sumX2 - $sumX * $sumX);
        $intercept = ($sumY - $slope * $sumX) / $n;
        
        $predictions = [];
        $lastDate = Carbon::parse(end($dates));
        $lastValue = end($values);
        
        for ($day = 1; $day <= $horizonDays; $day++) {
            $predDate = $lastDate->copy()->addDays($day);
            $predictedValue = $intercept + $slope * ($n + $day);
            $predictedPeak = $predictedValue * 1.15;
            
            $predictions[] = [
                'date' => $predDate->toDateString(),
                'predicted_usage' => max(0, $predictedValue),
                'predicted_peak' => max(0, $predictedPeak),
                'trend' => $slope > 0 ? 'increasing' : 'decreasing',
            ];
        }
        
        return $predictions;
    }
    
    protected function calculateConfidence($snapshots) {
        if ($snapshots->count() < 14) {
            return 50.0;
        }
        
        $values = $snapshots->pluck('usage_value')->toArray();
        $mean = array_sum($values) / count($values);
        
        $variance = 0;
        foreach ($values as $value) {
            $variance += pow($value - $mean, 2);
        }
        $variance /= count($values);
        $stdDev = sqrt($variance);
        
        $coefficient = $stdDev / ($mean ?: 1);
        $confidence = max(0, min(100, 100 - ($coefficient * 50)));
        
        return round($confidence, 2);
    }
    
    protected function storePredictions($poolId, $predictions) {
        Capsule::table('mod_capacity_predictions')
            ->where('pool_id', $poolId)
            ->where('prediction_date', '>=', Carbon::today())
            ->delete();
        
        foreach ($predictions as $pred) {
            Capsule::table('mod_capacity_predictions')->insert([
                'pool_id' => $poolId,
                'prediction_date' => $pred['date'],
                'predicted_usage' => $pred['predicted_usage'],
                'predicted_peak' => $pred['predicted_peak'],
                'confidence_level' => 75,
                'prediction_model' => 'linear',
                'horizon_days' => 30,
                'created_at' => Carbon::now(),
            ]);
        }
    }
}
```

### lib/AlertManager.php

```php
<?php
/**
 * Alert Manager
 */

namespace WHMCS\Module\CapacityPlanning;

use Illuminate\Database\Capsule\Manager as Capsule;
use Carbon\Carbon;

class AlertManager {
    
    public function getActiveAlerts($severity = null) {
        $query = Capsule::table('mod_capacity_alerts')
            ->where('is_resolved', 0)
            ->orderBy('created_at', 'desc');
        
        if ($severity) {
            $query->where('severity', $severity);
        }
        
        return $query->get();
    }
    
    public function resolveAlert($alertId) {
        return Capsule::table('mod_capacity_alerts')
            ->where('id', $alertId)
            ->update([
                'is_resolved' => 1,
                'resolved_at' => Carbon::now(),
            ]);
    }
    
    public function getAlertHistory($poolId = null, $days = 30) {
        $query = Capsule::table('mod_capacity_alerts')
            ->where('created_at', '>=', Carbon::now()->subDays($days))
            ->orderBy('created_at', 'desc');
        
        if ($poolId) {
            $query->where('pool_id', $poolId);
        }
        
        return $query->get();
    }
}
```

## API Endpoints

```
GET  /api/v1/capacity/overview           - Get capacity overview
GET  /api/v1/capacity/pools              - List all pools
GET  /api/v1/capacity/pools/{id}         - Get pool details
GET  /api/v1/capacity/snapshots/{poolId} - Get historical snapshots
GET  /api/v1/capacity/forecast/{poolId}  - Get forecasts
GET  /api/v1/capacity/alerts             - Get active alerts
POST /api/v1/capacity/alerts/{id}/resolve - Resolve an alert
GET  /api/v1/capacity/recommendations    - Get scaling recommendations
```

## Hooks Integration

```php
// Client sign-up capacity check
add_hook('CustomRegistrationRules', 1, function($params) {
    $planner = new \WHMCS\Module\CapacityPlanning\CapacityPlanner();
    $overview = $planner->getOverview();
    
    foreach ($overview as $pool) {
        if ($pool['utilization_rate'] >= 95) {
            return [
                'error' => "Capacity limit reached for {$pool['name']}. Please try again later.",
            ];
        }
    }
});

// Service creation capacity validation
add_hook('ServiceAdd', 1, function($params) {
    $planner = new \WHMCS\Module\CapacityPlanning\CapacityPlanner();
    $overview = $planner->getOverview();
    $targetPool = null;
    
    foreach ($overview as $pool) {
        if ($pool['type'] === $params['product_type']) {
            $targetPool = $pool;
            break;
        }
    }
    
    if ($targetPool && $targetPool['utilization_rate'] >= 90) {
        logActivity("Capacity warning: Service created at {$targetPool['utilization_rate']}% utilization");
    }
});
```

## Configuration Options

| Option | Type | Default | Description |
|--------|------|---------|-------------|
| forecast_horizon | text | 30 | Days ahead to forecast |
| alert_threshold_warning | text | 75 | Warning threshold % |
| alert_threshold_critical | text | 90 | Critical threshold % |
| snapshot_frequency | dropdown | daily | How often to capture usage |
| auto_scale_enabled | checkbox | false | Enable auto-scaling suggestions |
