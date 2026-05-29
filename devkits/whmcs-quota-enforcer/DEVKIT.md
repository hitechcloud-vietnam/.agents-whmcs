# WHMCS Quota Enforcer DevKit

## Overview

Quota enforcement system for WHMCS that monitors and enforces usage limits with automatic actions on quota violations.

## Module Files

```php
<?php
/**
 * WHMCS Quota Enforcer Module
 */

if (!defined("WHMCS")) {
    die("This file cannot be accessed directly");
}

require_once __DIR__ . '/lib/QuotaEnforcer.php';

function whmcs_quota_enforcer_activate() {
    $enforcer = new QuotaEnforcer();
    return $enforcer->activate();
}

function whmcs_quota_enforcer_check($userId, $resourceType) {
    $enforcer = new QuotaEnforcer();
    return $enforcer->checkQuota($userId, $resourceType);
}

function whmcs_quota_enforcer_enforce($userId, $resourceType) {
    $enforcer = new QuotaEnforcer();
    return $enforcer->enforceQuota($userId, $resourceType);
}
```

### lib/QuotaEnforcer.php

```php
<?php
namespace WHMCS\Module\QuotaEnforcer;

use Illuminate\Database\Capsule\Manager as Capsule;
use Carbon\Carbon;

class QuotaEnforcer {
    
    public function activate() {
        try {
            $this->createTables();
            return ['success' => true, 'msg' => 'Quota Enforcer module activated'];
        } catch (\Exception $e) {
            return ['success' => false, 'msg' => $e->getMessage()];
        }
    }
    
    protected function createTables() {
        Capsule::statement("
            CREATE TABLE IF NOT EXISTS `mod_quota_limits` (
                `id` INT UNSIGNED NOT NULL AUTO_INCREMENT,
                `user_id` INT UNSIGNED NOT NULL,
                `resource_type` VARCHAR(50) NOT NULL,
                `hard_limit` DECIMAL(15,4) NOT NULL,
                `soft_limit` DECIMAL(15,4) NULL,
                `current_usage` DECIMAL(15,4) NOT NULL DEFAULT 0,
                `enforcement_action` ENUM('block', 'warn', 'throttle') NOT NULL DEFAULT 'block',
                `reset_period` VARCHAR(20) DEFAULT 'monthly',
                `last_reset_at` DATETIME NULL,
                PRIMARY KEY (`id`),
                UNIQUE KEY `uk_user_resource` (`user_id`, `resource_type`)
            ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4
        ");
        
        Capsule::statement("
            CREATE TABLE IF NOT EXISTS `mod_quota_violations` (
                `id` INT UNSIGNED NOT NULL AUTO_INCREMENT,
                `quota_id` INT UNSIGNED NOT NULL,
                `violation_type` ENUM('soft_limit', 'hard_limit', 'rate_limit') NOT NULL,
                `usage_amount` DECIMAL(15,4) NOT NULL,
                `action_taken` VARCHAR(50) NOT NULL,
                `created_at` DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
                PRIMARY KEY (`id`)
            ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4
        ");
    }
    
    public function checkQuota($userId, $resourceType) {
        $quota = Capsule::table('mod_quota_limits')
            ->where('user_id', $userId)
            ->where('resource_type', $resourceType)
            ->first();
        
        if (!$quota) {
            return ['allowed' => true, 'status' => 'no_limit'];
        }
        
        $usagePercent = $quota->hard_limit > 0 ? ($quota->current_usage / $quota->hard_limit) * 100 : 0;
        
        if ($usagePercent >= 100) {
            return [
                'allowed' => false,
                'status' => 'hard_limit_exceeded',
                'enforcement_action' => $quota->enforcement_action,
            ];
        }
        
        if ($quota->soft_limit && $usagePercent >= ($quota->soft_limit / $quota->hard_limit) * 100) {
            return [
                'allowed' => true,
                'status' => 'soft_limit_warning',
                'usage_percent' => $usagePercent,
            ];
        }
        
        return [
            'allowed' => true,
            'status' => 'ok',
            'usage_percent' => $usagePercent,
            'remaining' => $quota->hard_limit - $quota->current_usage,
        ];
    }
    
    public function enforceQuota($userId, $resourceType) {
        $check = $this->checkQuota($userId, $resourceType);
        
        if (!$check['allowed']) {
            $quota = Capsule::table('mod_quota_limits')
                ->where('user_id', $userId)
                ->where('resource_type', $resourceType)
                ->first();
            
            Capsule::table('mod_quota_violations')->insert([
                'quota_id' => $quota->id,
                'violation_type' => 'hard_limit',
                'usage_amount' => $quota->current_usage,
                'action_taken' => $quota->enforcement_action,
            ]);
            
            // Take enforcement action
            switch ($quota->enforcement_action) {
                case 'block':
                    return ['blocked' => true, 'action' => 'blocked'];
                case 'throttle':
                    return ['blocked' => false, 'action' => 'throttled'];
                case 'warn':
                    return ['blocked' => false, 'action' => 'warning_sent'];
            }
        }
        
        return ['blocked' => false];
    }
    
    public function resetQuotas() {
        $quotas = Capsule::table('mod_quota_limits')
            ->where('reset_period', 'monthly')
            ->where('last_reset_at', '<', Carbon::now()->startOfMonth())
            ->get();
        
        foreach ($quotas as $quota) {
            Capsule::table('mod_quota_limits')
                ->where('id', $quota->id)
                ->update([
                    'current_usage' => 0,
                    'last_reset_at' => Carbon::now(),
                ]);
        }
    }
}

add_hook('DailyCronJob', 1, function() {
    $enforcer = new \WHMCS\Module\QuotaEnforcer\QuotaEnforcer();
    $enforcer->resetQuotas();
});
```

## API Endpoints

```
POST /api/v1/quota-enforcer/check        - Check quota
POST /api/v1/quota-enforcer/enforce      - Enforce quota
POST /api/v1/quota-enforcer/set-limit    - Set limit
```
