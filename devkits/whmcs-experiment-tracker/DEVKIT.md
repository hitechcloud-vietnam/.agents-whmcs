# WHMCS Experiment Tracker DevKit

## Overview

Experiment tracking system for WHMCS enabling statistical experiment management, hypothesis tracking, and results analysis.

## Module Files

```php
<?php
/**
 * WHMCS Experiment Tracker Module
 */

if (!defined("WHMCS")) {
    die("This file cannot be accessed directly");
}

require_once __DIR__ . '/lib/ExperimentTracker.php';

function whmcs_experiment_tracker_activate() {
    $tracker = new ExperimentTracker();
    return $tracker->activate();
}

function whmcs_experiment_create($data) {
    $tracker = new ExperimentTracker();
    return $tracker->createExperiment($data);
}
```

### lib/ExperimentTracker.php

```php
<?php
namespace WHMCS\Module\ExperimentTracker;

use Illuminate\Database\Capsule\Manager as Capsule;
use Carbon\Carbon;

class ExperimentTracker {
    
    public function activate() {
        try {
            $this->createTables();
            return ['success' => true, 'msg' => 'Experiment Tracker module activated'];
        } catch (\Exception $e) {
            return ['success' => false, 'msg' => $e->getMessage()];
        }
    }
    
    protected function createTables() {
        Capsule::statement("
            CREATE TABLE IF NOT EXISTS `mod_experiments` (
                `id` INT UNSIGNED NOT NULL AUTO_INCREMENT,
                `experiment_id` VARCHAR(64) NOT NULL,
                `hypothesis` TEXT NOT NULL,
                `description` TEXT NULL,
                `status` ENUM('hypothesis', 'designed', 'running', 'completed', 'archived') NOT NULL DEFAULT 'hypothesis',
                `start_date` DATE NULL,
                `end_date` DATE NULL,
                `confidence_level` DECIMAL(5,2) DEFAULT 95.00,
                `results` JSON NULL,
                `created_at` DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
                PRIMARY KEY (`id`)
            ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4
        ");
        
        Capsule::statement("
            CREATE TABLE IF NOT EXISTS `mod_experiment_observations` (
                `id` INT UNSIGNED NOT NULL AUTO_INCREMENT,
                `experiment_id` VARCHAR(64) NOT NULL,
                `metric_name` VARCHAR(100) NOT NULL,
                `variant` VARCHAR(50) NOT NULL,
                `value` DECIMAL(15,4) NOT NULL,
                `sample_size` INT UNSIGNED DEFAULT 0,
                `observed_at` DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
                PRIMARY KEY (`id`)
            ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4
        ");
    }
    
    public function createExperiment($data) {
        $experimentId = 'EXP-' . strtoupper(substr(uniqid(), 0, 8));
        
        $id = Capsule::table('mod_experiments')->insertGetId([
            'experiment_id' => $experimentId,
            'hypothesis' => $data['hypothesis'],
            'description' => $data['description'] ?? null,
            'status' => 'hypothesis',
        ]);
        
        return ['success' => true, 'experiment_id' => $experimentId];
    }
    
    public function recordObservation($experimentId, $metric, $variant, $value, $sampleSize = 1) {
        Capsule::table('mod_experiment_observations')->insert([
            'experiment_id' => $experimentId,
            'metric_name' => $metric,
            'variant' => $variant,
            'value' => $value,
            'sample_size' => $sampleSize,
        ]);
    }
}
```

## API Endpoints

```
POST /api/v1/experiments                 - Create experiment
GET  /api/v1/experiments               - List experiments
POST /api/v1/experiments/{id}/observe  - Record observation
GET  /api/v1/experiments/{id}/results   - Get results
```
