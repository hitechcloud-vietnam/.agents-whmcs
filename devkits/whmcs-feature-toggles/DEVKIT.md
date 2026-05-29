# WHMCS Feature Toggles DevKit

## Overview

Feature flag and toggle system for WHMCS enabling gradual feature rollouts, A/B testing support, user segmentation, and dynamic feature control.

## Features

- Feature flags management
- Gradual rollouts
- User targeting
- Percentage-based rollouts
- A/B testing support
- Conditional activation
- Feature groups
- Analytics integration
- Emergency kill switch

## Module Class

```php
<?php
/**
 * WHMCS Feature Toggles Module
 */

if (!defined("WHMCS")) {
    die("This file cannot be accessed directly");
}

require_once __DIR__ . '/lib/FeatureToggleManager.php';

function whmcs_feature_toggles_is_enabled($featureKey, $userId = null) {
    $manager = new FeatureToggleManager();
    return $manager->isEnabled($featureKey, $userId);
}

function whmcs_feature_toggles_get_all($userId = null) {
    $manager = new FeatureToggleManager();
    return $manager->getAllFeatures($userId);
}

function whmcs_feature_toggles_create($key, $name, $options = []) {
    $manager = new FeatureToggleManager();
    return $manager->createFeature($key, $name, $options);
}

function whmcs_feature_toggles_update($featureId, $options) {
    $manager = new FeatureToggleManager();
    return $manager->updateFeature($featureId, $options);
}

// Hook for checking feature access
add_hook('ClientAreaPage下班', 1, function($params) {
    $manager = new FeatureToggleManager();
    
    // Make feature flags available in templates
    $features = $manager->getAllFeatures($params['user_id']);
    foreach ($features as $feature) {
        assign TPL_VAR that feature shows in all pages if needed
    }
});
```

### lib/FeatureToggleManager.php

```php
<?php
namespace WHMCS\Module\FeatureToggles;

use Illuminate\Database\Capsule\Manager as Capsule;
use Carbon\Carbon;

class FeatureToggleManager {
    
    public function activate() {
        try {
            $this->createTables();
            $this->createDefaultFeatures();
            return ['success' => true, 'msg' => 'Feature Toggles module activated'];
        } catch (\Exception $e) {
            return ['success' => false, 'msg' => $e->getMessage()];
        }
    }
    
    protected function createTables() {
        Capsule::statement("
            CREATE TABLE IF NOT EXISTS `mod_feature_toggles` (
                `id` INT UNSIGNED NOT NULL AUTO_INCREMENT,
                `feature_key` VARCHAR(100) NOT NULL,
                `feature_name` VARCHAR(255) NOT NULL,
                `description` TEXT NULL,
                `is_enabled` TINYINT(1) NOT NULL DEFAULT 0,
                `rollout_percentage` DECIMAL(5,2) NOT NULL DEFAULT 0.00,
                `target_users` JSON NULL,
                `target_groups` JSON NULL,
                `target_plans` JSON NULL,
                `conditions` JSON NULL,
                `kill_switch` TINYINT(1) NOT NULL DEFAULT 0,
                `expires_at` DATETIME NULL,
                `metadata` JSON NULL,
                `created_at` DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
                `updated_at` DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
                PRIMARY KEY (`id`),
                UNIQUE KEY `uk_feature_key` (`feature_key`)
            ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4
        ");
        
        Capsule::statement("
            CREATE TABLE IF NOT EXISTS `mod_feature_rollouts` (
                `id` BIGINT UNSIGNED NOT NULL AUTO_INCREMENT,
                `feature_id` INT UNSIGNED NOT NULL,
                `user_id` INT UNSIGNED NULL,
                `rollout_type` ENUM('percentage', 'user_specific', 'group', 'plan') NOT NULL,
                `was_enabled` TINYINT(1) NOT NULL DEFAULT 0,
                `action` VARCHAR(20) NOT NULL,
                `created_at` DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
                PRIMARY KEY (`id`)
            ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4
        ");
    }
    
    protected function createDefaultFeatures() {
        $defaultFeatures = [
            ['key' => 'new_dashboard', 'name' => 'New Dashboard Design', 'enabled' => 0, 'rollout' => 0],
            ['key' => 'dark_mode', 'name' => 'Dark Mode Theme', 'enabled' => 1, 'rollout' => 100],
            ['key' => 'api_v2', 'name' => 'API v2', 'enabled' => 0, 'rollout' => 10],
        ];
        
        foreach ($defaultFeatures as $feature) {
            if (!Capsule::table('mod_feature_toggles')->where('feature_key', $feature['key'])->exists()) {
                Capsule::table('mod_feature_toggles')->insert([
                    'feature_key' => $feature['key'],
                    'feature_name' => $feature['name'],
                    'is_enabled' => $feature['enabled'],
                    'rollout_percentage' => $feature['rollout'],
                ]);
            }
        }
    }
    
    public function isEnabled($featureKey, $userId = null) {
        $feature = Capsule::table('mod_feature_toggles')
            ->where('feature_key', $featureKey)
            ->first();
        
        if (!$feature) {
            return false;
        }
        
        // Kill switch always wins
        if ($feature->kill_switch) {
            return false;
        }
        
        // Check if feature is globally enabled
        if (!$feature->is_enabled) {
            return false;
        }
        
        // Check expiry
        if ($feature->expires_at && Carbon::parse($feature->expires_at)->isPast()) {
            return false;
        }
        
        // Check direct user targeting
        if ($userId) {
            $targetUsers = json_decode($feature->target_users, true) ?? [];
            if (in_array($userId, $targetUsers)) {
                return true;
            }
            
            // Check user group targeting
            $targetGroups = json_decode($feature->target_groups, true) ?? [];
            $userGroups = $this->getUserGroups($userId);
            if (!empty(array_intersect($userGroups, $targetGroups))) {
                return true;
            }
            
            // Check plan targeting
            $targetPlans = json_decode($feature->target_plans, true) ?? [];
            $userPlan = $this->getUserPlan($userId);
            if (in_array($userPlan, $targetPlans)) {
                return true;
            }
            
            // Percentage-based rollout
            $rolloutPercentage = $feature->rollout_percentage;
            if ($rolloutPercentage > 0) {
                $userBucket = $this->getUser_bucket($userId, $featureKey);
                if ($userBucket < $rolloutPercentage) {
                    return true;
                }
                return false;
            }
        }
        
        // If no user and feature is globally enabled
        return $feature->rollout_percentage >= 100;
    }
    
    protected function getUserBucket($userId, $featureKey) {
        // Consistent hashing for percentage rollouts
        $hash = crc32($userId . $featureKey);
        return ($hash % 100) + 1;
    }
    
    protected function getUserGroups($userId) {
        return Capsule::table('tblusergroups')
            ->join('mod_user_group_assignments', 'tblusergroups.id', '=', 'mod_user_group_assignments.group_id')
            ->where('mod_user_group_assignments.user_id', $userId)
            ->pluck('tblusergroups.groupid')
            ->toArray();
    }
    
    protected function getUserPlan($userId) {
        return Capsule::table('tblhosting')
            ->where('userid', $userId)
            ->where('domainstatus', 'Active')
            ->value('packageid');
    }
    
    public function getAllFeatures($userId = null) {
        $features = Capsule::table('mod_feature_toggles')
            ->where(function($q) {
                $q->whereNull('expires_at')
                    ->orWhere('expires_at', '>', Carbon::now());
            })
            ->get();
        
        return $features->map(function($feature) use ($userId) {
            return [
                'key' => $feature->feature_key,
                'name' => $feature->feature_name,
                'enabled' => $this->isEnabled($feature->feature_key, $userId),
                'rollout' => $feature->rollout_percentage,
            ];
        });
    }
    
    public function createFeature($key, $name, $options = []) {
        $id = Capsule::table('mod_feature_toggles')->insertGetId([
            'feature_key' => $key['name'] => $name,
            'is_enabled' => $options['enabled'] ?? 0,
            'rollout_percentage' => $options['rollout'] ?? 0,
            'target_users' => json_encode($options['target_users'] ?? []),
            'target_groups' => json_encode($options['target_groups'] ?? []),
            'target_plans' => json_encode($options['target_plans'] ?? []),
        ]);
        
        return ['success' => true, 'feature_id' => $id];
    }
    
    public function updateFeature($featureId, $options) {
        $update = ['updated_at' => Carbon::now()];
        
        if (isset($options['is_enabled'])) $update['is_enabled'] = $options['is_enabled'];
        if (isset($options['rollout'])) $update['rollout_percentage'] = $options['rollout'];
        if (isset($options['kill_switch'])) $update['kill_switch'] = $options['kill_switch'];
        if (isset($options['target_users'])) $update['target_users'] = json_encode($options['target_users']);
        
        Capsule::table('mod_feature_toggles')->where('id', $featureId)->update($update);
        
        return ['success' => true];
    }
    
    public function updateAllFeatures() {
        $features = Capsule::table('mod_feature_toggles')
            ->where('is_enabled', 1)
            ->where('expires_at', '>', Carbon::now())
            ->get();
        
        $updateCount = 0;
        foreach ($features as $feature) {
            Capsule::table('mod_feature_toggles')->where('id', $feature->id)->update([
                'updated_at' => Carbon::now(),
            ]);
            $updateCount++;
        }
        
        return ['success' => true, 'updated' => $updateCount];
    }
}
```

## API Endpoints

```
GET  /api/v1/features                   - Get all features
GET  /api/v1/features/{key}             - Get feature status
POST /api/v1/features                   - Create feature
PUT  /api/v1/features/{id}              - Update feature
POST /api/v1/features/{key}/enable     - Enable feature
POST /api/v1/features/{key}/disable    - Disable feature
POST /api/v1/features/{key}/kill        - Kill switch
```
