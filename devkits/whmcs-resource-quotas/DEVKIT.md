# WHMCS Resource Quotas DevKit

## Overview

Resource quota enforcement system for WHMCS that manages and monitors resource usage per tenant, enforces limits, handles overages, and provides quota visibility.

## Features

- Per-tenant quota limits
- Multiple resource types
- Soft/hard limits
- Overage handling
- Quota warnings
- Automatic enforcement
- Usage tracking
- Quota history

## Module Class

```php
<?php
/**
 * WHMCS Resource Quotas Module
 */

if (!defined("WHMCS")) {
    die("This file cannot be accessed directly");
}

require_once __DIR__ . '/lib/QuotaManager.php';

function whmcs_resource_quotas_check($userId, $resourceType, $requestedAmount) {
    $manager = new QuotaManager();
    return $manager->checkQuota($userId, $resourceType, $requestedAmount);
}

function whmcs_resource_quotas_consume($userId, $resourceType, $amount) {
    $manager = new QuotaManager();
    return $manager->consume($userId, $resourceType, $amount);
}

function whmcs_resource_quotas_get_usage($userId) {
    $manager = new QuotaManager();
    return $manager->getUsage($userId);
}

function whmcs_resource_quotas_set_limit($userId, $resourceType, $limit) {
    $manager = new QuotaManager();
    return $manager->setLimit($userId, $resourceType, $limit);
}
```

### lib/QuotaManager.php

```php
<?php
namespace WHMCS\Module\ResourceQuotas;

use Illuminate\Database\Capsule\Manager as Capsule;
use Carbon\Carbon;

class QuotaManager {
    
    protected $resourceTypes = ['api_calls', 'storage', 'bandwidth', 'compute', 'users'];
    
    public function activate() {
        try {
            $this->createTables();
            return ['success' => true, 'msg' => 'Resource Quotas module activated'];
        } catch (\Exception $e) {
            return ['success' => false, 'msg' => $e->getMessage()];
        }
    }
    
    protected function createTables() {
        Capsule::statement("
            CREATE TABLE IF NOT EXISTS `mod_resource_quotas` (
                `id` INT UNSIGNED NOT NULL AUTO_INCREMENT,
                `user_id` INT UNSIGNED NOT NULL,
                `resource_type` VARCHAR(50) NOT NULL,
                `soft_limit` DECIMAL(15,4) NOT NULL DEFAULT 0,
                `hard_limit` DECIMAL(15,4) NOT NULL DEFAULT 0,
                `current_usage` DECIMAL(15,4) NOT NULL DEFAULT 0,
                `reset_period` VARCHAR(20) NOT NULL DEFAULT 'monthly',
                `created_at` DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
                PRIMARY KEY (`id`),
                UNIQUE KEY `uk_user_resource` (`user_id`, `resource_type`)
            ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4
        ");
        
        Capsule::statement("
            CREATE TABLE IF NOT EXISTS `mod_resource_usage_logs` (
                `id` BIGINT UNSIGNED NOT NULL AUTO_INCREMENT,
                `quota_id` INT UNSIGNED NOT NULL,
                `amount` DECIMAL(15,4) NOT NULL,
                `action` VARCHAR(20) NOT NULL,
                `created_at` DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
                PRIMARY KEY (`id`)
            ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4
        ");
    }
    
    public function checkQuota($userId, $resourceType, $requestedAmount) {
        $quota = $this->getOrCreateQuota($userId, $resourceType);
        
        $available = $quota->hard_limit - $quota->current_usage;
        
        if ($requestedAmount > $available) {
            return [
                'allowed' => false,
                'reason' => ' quota_exceeded',
                'available' => $available,
                'requested' => $requestedAmount,
                'percentage_used' => $quota->hard_limit > 0 
                    ? ($quota->current_usage / $quota->hard_limit) * 100 
                    : 0,
            ];
        }
        
        // Check if going into soft limit warning zone
        $softLimitUsed = $quota->soft_limit > 0 
            ? ($quota->current_usage + $requestedAmount) / $quota->soft_limit > 1 
            : false;
        
        return [
            'allowed' => true,
            'available' => $available,
            'warning' => $softLimitUsed,
        ];
    }
    
    protected function getOrCreateQuota($userId, $resourceType) {
        $quota = Capsule::table('mod_resource_quotas')
            ->where('user_id', $userId)
            ->where('resource_type', $resourceType)
            ->first();
        
        if ($quota) {
            return $quota;
        }
        
        // Create with defaults based on service
        $service = Capsule::table('tblhosting')
            ->where('userid', $userId)
            ->where('domainstatus', 'Active')
            ->first();
        
        $defaults = $this->getDefaultLimits($resourceType, $service->packageid ?? 0);
        
        $id = Capsule::table('mod_resource_quotas')->insertGetId([
            'user_id' => $userId,
            'resource_type' => $resourceType,
            'soft_limit' => $defaults['soft'],
            'hard_limit' => $defaults['hard'],
        ]);
        
        return Capsule::table('mod_resource_quotas')->where('id', $id)->first();
    }
    
    protected function getDefaultLimits($resourceType, $packageId) {
        $limits = [
            'api_calls' => ['soft' => 90000, 'hard' => 100000],
            'storage' => ['soft' => 4500, 'hard' => 5000],
            'bandwidth' => ['soft' => 450, 'hard' => 500],
        ];
        
        return $limits[$resourceType] ?? ['soft' => 90, 'hard' => 100];
    }
    
    public function consume($userId, $resourceType, $amount) {
        $check = $this->checkQuota($userId, $resourceType, $amount);
        
        if (!$check['allowed']) {
            return $check;
        }
        
        $quota = Capsule::table('mod_resource_quotas')
            ->where('user_id', $userId)
            ->where('resource_type', $resourceType)
            ->first();
        
        Capsule::table('mod_resource_quotas')
            ->where('id', $quota->id)
            ->increment('current_usage', $amount);
        
        Capsule::table('mod_resource_usage_logs')->insert([
            'quota_id' => $quota->id,
            'amount' => $amount,
            'action' => 'consume',
        ]);
        
        return ['success' => true, 'remaining' => $check['available'] - $amount];
    }
    
    public function getUsage($userId) {
        $quotas = Capsule::table('mod_resource_quotas')
            ->where('user_id', $userId)
            ->get();
        
        return $quotas->map(function($quota) {
            return [
                'resource_type' => $quota->resource_type,
                'usage' => $quota->current_usage,
                'soft_limit' => $quota->soft_limit,
                'hard_limit' => $quota->hard_limit,
                'percentage_used' => $quota->hard_limit > 0 
                    ? ($quota->current_usage / $quota->hard_limit) * 100 
                    : 0,
                'over_soft_limit' => $quota->current_usage > $quota->soft_limit,
                'over_hard_limit' => $quota->current_usage > $quota->hard_limit,
            ];
        });
    }
    
    public function setLimit($userId, $resourceType, $limit, $softLimit = null) {
        $update = ['hard_limit' => $limit];
        
        if ($softLimit !== null) {
            $update['soft_limit'] = $softLimit;
        }
        
        Capsule::table('mod_resource_quotas')
            ->where('user_id', $userId)
            ->where('resource_type', $resourceType)
            ->update($update);
        
        return ['success' => true];
    }
    
    public function resetQuotas($userId) {
        Capsule::table('mod_resource_quotas')
            ->where('user_id', $userId)
            ->update(['current_usage' => 0]);
        
        return ['success' => true];
    }
}
```

## API Endpoints

```
GET  /api/v1/quota/{userId}             - Get all quotas
POST /api/v1/quota/{userId}/check       - Check quota availability
POST /api/v1/quota/{userId}/consume     - Consume resources
POST /api/v1/quota/{userId}/reset      - Reset quotas
PUT  /api/v1/quota/{userId}/{type}     - Update quota limit
```
