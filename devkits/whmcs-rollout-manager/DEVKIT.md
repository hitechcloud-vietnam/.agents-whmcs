# WHMCS Rollout Manager DevKit

## Overview

Gradual feature rollout management system for WHMCS enabling controlled deployments with automatic traffic management and rollback capabilities.

## Module Files

```php
<?php
/**
 * WHMCS Rollout Manager Module
 */

if (!defined("WHMCS")) {
    die("This file cannot be accessed directly");
}

require_once __DIR__ . '/lib/RolloutManager.php';

function whmcs_rollout_manager_activate() {
    $manager = new RolloutManager();
    return $manager->activate();
}

function whmcs_rollout_start($featureKey, $initialPercentage = 10) {
    $manager = new RolloutManager();
    return $manager->startRollout($featureKey, $initialPercentage);
}

function whmcs_rollout_increase($rolloutId, $percentage) {
    $manager = new RolloutManager();
    return $manager->increasePercentage($rolloutId, $percentage);
}
```

### lib/RolloutManager.php

```php
<?php
namespace WHMCS\Module\RolloutManager;

use Illuminate\Database\Capsule\Manager as Capsule;
use Carbon\Carbon;

class RolloutManager {
    
    public function activate() {
        try {
            $this->createTables();
            return ['success' => true, 'msg' => 'Rollout Manager module activated'];
        } catch (\Exception $e) {
            return ['success' => false, 'msg' => $e->getMessage()];
        }
    }
    
    protected function createTables() {
        Capsule::statement("
            CREATE TABLE IF NOT EXISTS `mod_feature_rollouts` (
                `id` INT UNSIGNED NOT NULL AUTO_INCREMENT,
                `feature_key` VARCHAR(100) NOT NULL,
                `current_percentage` DECIMAL(5,2) NOT NULL DEFAULT 0.00,
                `status` ENUM('planning', 'rolling_out', 'monitoring', 'completed', 'rolled_back') NOT NULL DEFAULT 'planning',
                `health_score` DECIMAL(5,2) DEFAULT 100.00,
                `error_threshold` DECIMAL(5,2) DEFAULT 5.00,
                `started_at` DATETIME NULL,
                `completed_at` DATETIME NULL,
                `created_at` DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
                PRIMARY KEY (`id`),
                UNIQUE KEY `uk_feature` (`feature_key`)
            ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4
        ");
        
        Capsule::statement("
            CREATE TABLE IF NOT EXISTS `mod_rollout_metrics` (
                `id` BIGINT UNSIGNED NOT NULL AUTO_INCREMENT,
                `rollout_id` INT UNSIGNED NOT NULL,
                `metric_type` VARCHAR(50) NOT NULL,
                `value` DECIMAL(15,4) NOT NULL,
                `recorded_at` DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
                PRIMARY KEY (`id`)
            ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4
        ");
    }
    
    public function startRollout($featureKey, $initialPercentage = 10) {
        $id = Capsule::table('mod_feature_rollouts')->insertGetId([
            'feature_key' => $featureKey,
            'current_percentage' => $initialPercentage,
            'status' => 'rolling_out',
            'started_at' => Carbon::now(),
        ]);
        
        return ['success' => true, 'rollout_id' => $id];
    }
    
    public function increasePercentage($rolloutId, $percentage) {
        Capsule::table('mod_feature_rollouts')
            ->where('id', $rolloutId)
            ->update(['current_percentage' => $percentage]);
        
        return ['success' => true, 'new_percentage' => $percentage];
    }
    
    public function recordMetrics($rolloutId, $metrics) {
        foreach ($metrics as $type => $value) {
            Capsule::table('mod_rollout_metrics')->insert([
                'rollout_id' => $rolloutId,
                'metric_type' => $type,
                'value' => $value,
            ]);
        }
    }
    
    public function checkHealth($rolloutId) {
        $rollout = Capsule::table('mod_feature_rollouts')->where('id', $rolloutId)->first();
        $recentMetrics = Capsule::table('mod_rollout_metrics')
            ->where('rollout_id', $rolloutId)
            ->where('recorded_at', '>=', Carbon::now()->subHours(1))
            ->where('metric_type', 'error_rate')
            ->avg('value') ?? 0;
        
        $healthScore = 100 - ($recentMetrics * 10);
        
        Capsule::table('mod_feature_rollouts')
            ->where('id', $rolloutId)
            ->update(['health_score' => $healthScore]);
        
        if ($recentMetrics > $rollout->error_threshold) {
            return ['status' => 'degraded', 'health_score' => $healthScore, 'recommendation' => 'pause'];
        }
        
        return ['status' => 'healthy', 'health_score' => $healthScore, 'recommendation' => 'continue'];
    }
}
```

## API Endpoints

```
POST /api/v1/rollout/start              - Start rollout
POST /api/v1/rollout/{id}/increase       - Increase percentage
POST /api/v1/rollout/{id}/metrics        - Record metrics
GET  /api/v1/rollout/{id}/health         - Check health
POST /api/v1/rollout/{id}/pause          - Pause rollout
POST /api/v1/rollout/{id}/complete       - Complete rollout
```
