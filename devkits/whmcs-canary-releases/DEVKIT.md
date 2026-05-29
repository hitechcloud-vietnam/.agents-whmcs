# WHMCS Canary Releases DevKit

## Overview

Canary release management system for WHMCS enabling gradual feature rollouts with traffic splitting, metrics monitoring, and automatic rollback.

## Features

- Canary deployments
- Traffic splitting
- Automatic rollback
- Health monitoring
- Performance metrics
- Gradual rollout
- Feature comparison

## Module Files

```php
<?php
/**
 * WHMCS Canary Releases Module
 */

if (!defined("WHMCS")) {
    die("This file cannot be accessed directly");
}

require_once __DIR__ . '/lib/CanaryManager.php';

function whmcs_canary_releases_activate() {
    $manager = new CanaryManager();
    return $manager->activate();
}

function whmcs_canary_create_release($data) {
    $manager = new CanaryManager();
    return $manager->createRelease($data);
}

function whmcs_canary_get_status($releaseId) {
    $manager = new CanaryManager();
    return $manager->getReleaseStatus($releaseId);
}
```

### lib/CanaryManager.php

```php
<?php
namespace WHMCS\Module\CanaryReleases;

use Illuminate\Database\Capsule\Manager as Capsule;
use Carbon\Carbon;

class CanaryManager {
    
    public function activate() {
        try {
            $this->createTables();
            return ['success' => true, 'msg' => 'Canary Releases module activated'];
        } catch (\Exception $e) {
            return ['success' => false, 'msg' => $e->getMessage()];
        }
    }
    
    protected function createTables() {
        Capsule::statement("
            CREATE TABLE IF NOT EXISTS `mod_canary_releases` (
                `id` INT UNSIGNED NOT NULL AUTO_INCREMENT,
                `release_name` VARCHAR(255) NOT NULL,
                `feature_key` VARCHAR(100) NOT NULL,
                `control_version` VARCHAR(50) NOT NULL,
                `canary_version` VARCHAR(50) NOT NULL,
                `traffic_percentage` DECIMAL(5,2) NOT NULL DEFAULT 0.00,
                `target_users` JSON NULL,
                `status` ENUM('planning', 'deploying', 'monitoring', 'promoted', 'rolled_back', 'cancelled') NOT NULL DEFAULT 'planning',
                `rollback_threshold` DECIMAL(5,2) NULL,
                `metrics_threshold` JSON NULL,
                `started_at` DATETIME NULL,
                `completed_at` DATETIME NULL,
                `created_at` DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
                PRIMARY KEY (`id`)
            ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4
        ");
        
        Capsule::statement("
            CREATE TABLE IF NOT EXISTS `mod_canary_metrics` (
                `id` BIGINT UNSIGNED NOT NULL AUTO_INCREMENT,
                `release_id` INT UNSIGNED NOT NULL,
                `version` VARCHAR(50) NOT NULL,
                `metric_name` VARCHAR(100) NOT NULL,
                `metric_value` DECIMAL(15,4) NOT NULL,
                `sample_size` INT UNSIGNED NOT NULL DEFAULT 0,
                `recorded_at` DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
                PRIMARY KEY (`id`),
                INDEX `idx_release_metrics` (`release_id`, `recorded_at`)
            ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4
        ");
    }
    
    public function createRelease($data) {
        $id = Capsule::table('mod_canary_releases')->insertGetId([
            'release_name' => $data['name'],
            'feature_key' => $data['feature_key'],
            'control_version' => $data['control'],
            'canary_version' => $data['canary'],
            'traffic_percentage' => $data['initial_traffic'] ?? 10,
            'rollback_threshold' => $data['rollback_threshold'] ?? 5,
            'status' => 'planning',
        ]);
        
        return ['success' => true, 'release_id' => $id];
    }
    
    public function getReleaseStatus($releaseId) {
        $release = Capsule::table('mod_canary_releases')->where('id', $releaseId)->first();
        
        $metrics = Capsule::table('mod_canary_metrics')
            ->where('release_id', $releaseId)
            ->where('recorded_at', '>=', Carbon::now()->subHours(1))
            ->get();
        
        $controlMetrics = $metrics->where('version', $release->control_version);
        $canaryMetrics = $metrics->where('version', $release->canary_version);
        
        return [
            'release' => $release,
            'control_metrics' => $this->aggregateMetrics($controlMetrics),
            'canary_metrics' => $this->aggregateMetrics($canaryMetrics),
            'recommendation' => $this->getRecommendation($release, $controlMetrics, $canaryMetrics),
        ];
    }
    
    protected function aggregateMetrics($metrics) {
        if ($metrics->isEmpty()) {
            return ['error_rate' => 0, 'latency_avg' => 0];
        }
        
        return [
            'error_rate' => $metrics->where('metric_name', 'error_rate')->avg('metric_value'),
            'latency_avg' => $metrics->where('metric_name', 'latency')->avg('metric_value'),
            'requests' => $metrics->where('metric_name', 'requests')->sum('metric_value'),
        ];
    }
    
    protected function getRecommendation($release, $control, $canary) {
        if ($control->isEmpty() || $canary->isEmpty()) {
            return 'insufficient_data';
        }
        
        $controlError = $control->where('metric_name', 'error_rate')->avg('metric_value') ?? 0;
        $canaryError = $canary->where('metric_name', 'error_rate')->avg('metric_value') ?? 0;
        
        if ($canaryError > $controlError * (1 + $release->rollback_threshold / 100)) {
            return 'rollback';
        }
        
        if ($canaryError <= $controlError * 0.9) {
            return 'promote';
        }
        
        return 'continue';
    }
    
    public function recordMetrics($releaseId, $version, $metrics) {
        foreach ($metrics as $name => $value) {
            Capsule::table('mod_canary_metrics')->insert([
                'release_id' => $releaseId,
                'version' => $version,
                'metric_name' => $name,
                'metric_value' => $value,
            ]);
        }
    }
}
```

## API Endpoints

```
POST /api/v1/canary/releases             - Create release
GET  /api/v1/canary/releases/{id}       - Get release status
POST /api/v1/canary/releases/{id}/promote - Promote canary
POST /api/v1/canary/releases/{id}/rollback - Rollback
POST /api/v1/canary/releases/{id}/metrics - Record metrics
```
