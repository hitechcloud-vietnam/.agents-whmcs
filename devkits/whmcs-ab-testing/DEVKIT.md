# WHMCS A/B Testing Framework DevKit

## Overview

A/B testing framework for WHMCS enabling controlled experiments, statistical analysis, and data-driven optimization decisions.

## Module Files

```php
<?php
/**
 * WHMCS A/B Testing Framework Module
 */

if (!defined("WHMCS")) {
    die("This file cannot be accessed directly");
}

require_once __DIR__ . '/lib/ABTestingFramework.php';

function whmcs_ab_testing_activate() {
    $framework = new ABTestingFramework();
    return $framework->activate();
}

function whmcs_ab_testing_get_variant($experimentId, $userId = null) {
    $framework = new ABTestingFramework();
    return $framework->getVariant($experimentId, $userId);
}

function whmcs_ab_testing_track($experimentId, $variant, $goal, $value = 1) {
    $framework = new ABTestingFramework();
    return $framework->trackConversion($experimentId, $variant, $goal, $value);
}
```

### lib/ABTestingFramework.php

```php
<?php
namespace WHMCS\Module\ABTestingFramework;

use Illuminate\Database\Capsule\Manager as Capsule;
use Carbon\Carbon;

class ABTestingFramework {
    
    public function activate() {
        try {
            $this->createTables();
            return ['success' => true, 'msg' => 'A/B Testing module activated'];
        } catch (\Exception $e) {
            return ['success' => false, 'msg' => $e->getMessage()];
        }
    }
    
    protected function createTables() {
        Capsule::statement("
            CREATE TABLE IF NOT EXISTS `mod_ab_experiments` (
                `id` INT UNSIGNED NOT NULL AUTO_INCREMENT,
                `experiment_name` VARCHAR(255) NOT NULL,
                `experiment_code` VARCHAR(50) NOT NULL,
                `description` TEXT NULL,
                `variants` JSON NOT NULL,
                `traffic_split` JSON NOT NULL,
                `target_users` JSON NULL,
                `status` ENUM('draft', 'running', 'paused', 'completed') NOT NULL DEFAULT 'draft',
                `start_date` DATETIME NULL,
                `end_date` DATETIME NULL,
                `min_sample_size` INT UNSIGNED DEFAULT 100,
                `confidence_level` DECIMAL(5,2) DEFAULT 95.00,
                `created_at` DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
                PRIMARY KEY (`id`)
            ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4
        ");
        
        Capsule::statement("
            CREATE TABLE IF NOT EXISTS `mod_ab_assignments` (
                `id` BIGINT UNSIGNED NOT NULL AUTO_INCREMENT,
                `experiment_id` INT UNSIGNED NOT NULL,
                `user_id` INT UNSIGNED NULL,
                `variant` VARCHAR(50) NOT NULL,
                `assigned_at` DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
                PRIMARY KEY (`id`),
                UNIQUE KEY `uk_experiment_user` (`experiment_id`, `user_id`)
            ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4
        ");
        
        Capsule::statement("
            CREATE TABLE IF NOT EXISTS `mod_ab_conversions` (
                `id` BIGINT UNSIGNED NOT NULL AUTO_INCREMENT,
                `experiment_id` INT UNSIGNED NOT NULL,
                `variant` VARCHAR(50) NOT NULL,
                `goal` VARCHAR(100) NOT NULL,
                `value` DECIMAL(15,4) DEFAULT 1,
                `converted_at` DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
                PRIMARY KEY (`id`),
                INDEX `idx_experiment_variant` (`experiment_id`, `variant`)
            ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4
        ");
    }
    
    public function createExperiment($data) {
        $id = Capsule::table('mod_ab_experiments')->insertGetId([
            'experiment_name' => $data['name'],
            'experiment_code' => 'EXP-' . strtoupper(substr(uniqid(), -6)),
            'variants' => json_encode($data['variants']),
            'traffic_split' => json_encode($data['traffic_split'] ?? []),
            'min_sample_size' => $data['sample_size'] ?? 100,
        ]);
        
        return ['success' => true, 'experiment_id' => $id];
    }
    
    public function getVariant($experimentId, $userId = null) {
        // Check existing assignment
        $assignment = Capsule::table('mod_ab_assignments')
            ->where('experiment_id', $experimentId)
            ->where('user_id', $userId)
            ->first();
        
        if ($assignment) {
            return $assignment->variant;
        }
        
        // Create new assignment
        $experiment = Capsule::table('mod_ab_experiments')->where('id', $experimentId)->first();
        $variants = json_decode($experiment->variants, true);
        $split = json_decode($experiment->traffic_split, true) ?? array_fill_keys($variants, 100 / count($variants));
        
        // Deterministic assignment based on user
        $bucket = $userId ? crc32($userId . $experimentId) % 100 : rand(0, 99);
        
        $cumulative = 0;
        foreach ($variants as $variant) {
            $cumulative += $split[$variant] ?? (100 / count($variants));
            if ($bucket < $cumulative) {
                Capsule::table('mod_ab_assignments')->insert([
                    'experiment_id' => $experimentId,
                    'user_id' => $userId,
                    'variant' => $variant,
                ]);
                return $variant;
            }
        }
        
        return $variants[0];
    }
    
    public function trackConversion($experimentId, $variant, $goal, $value = 1) {
        Capsule::table('mod_ab_conversions')->insert([
            'experiment_id' => $experimentId,
            'variant' => $variant,
            'goal' => $goal,
            'value' => $value,
        ]);
        
        return ['success' => true];
    }
    
    public function getResults($experimentId) {
        $conversions = Capsule::table('mod_ab_conversions')
            ->where('experiment_id', $experimentId)
            ->get();
        
        $assignments = Capsule::table('mod_ab_assignments')
            ->where('experiment_id', $experimentId)
            ->get();
        
        $experiment = Capsule::table('mod_ab_experiments')->where('id', $experimentId)->first();
        $variants = json_decode($experiment->variants, true);
        
        $results = [];
        foreach ($variants as $variant) {
            $variantConversions = $conversions->where('variant', $variant);
            $variantAssignments = $assignments->where('variant', $variant);
            
            $conversions_count = $variantConversions->count();
            $sample_size = $variantAssignments->count();
            $conversion_rate = $sample_size > 0 ? ($conversions_count / $sample_size) * 100 : 0;
            
            $results[$variant] = [
                'conversions' => $conversions_count,
                'sample_size' => $sample_size,
                'conversion_rate' => round($conversion_rate, 2),
            ];
        }
        
        return $results;
    }
}
```

## API Endpoints

```
POST /api/v1/ab/experiments              - Create experiment
GET  /api/v1/ab/experiments            - List experiments
POST /api/v1/ab/experiments/{id}/track  - Track conversion
GET  /api/v1/ab/experiments/{id}/results - Get results
```
