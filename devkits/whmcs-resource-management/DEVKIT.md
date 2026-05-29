# WHMCS Resource Management DevKit

## Overview

A comprehensive resource allocation and management system for WHMCS, enabling tracking, allocation, and monitoring of computing resources across customer accounts.

## Features

- Resource pool management
- Allocation tracking and limits
- Real-time resource monitoring
- Resource usage reporting
- Quota management
- Resource transfer between accounts

## Database Schema

```sql
CREATE TABLE IF NOT EXISTS `mod_resource_pools` (
    `id` INT UNSIGNED NOT NULL AUTO_INCREMENT,
    `pool_name` VARCHAR(255) NOT NULL,
    `pool_type` ENUM('compute', 'storage', 'bandwidth', 'memory') NOT NULL DEFAULT 'compute',
    `total_capacity` DECIMAL(15,4) NOT NULL DEFAULT 0,
    `available_capacity` DECIMAL(15,4) NOT NULL DEFAULT 0,
    `unit` VARCHAR(50) NOT NULL DEFAULT 'units',
    `is_active` TINYINT(1) NOT NULL DEFAULT 1,
    `created_at` DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
    `updated_at` DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    PRIMARY KEY (`id`),
    INDEX `idx_pool_type` (`pool_type`),
    INDEX `idx_is_active` (`is_active`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;

CREATE TABLE IF NOT EXISTS `mod_resource_allocations` (
    `id` INT UNSIGNED NOT NULL AUTO_INCREMENT,
    `user_id` INT UNSIGNED NOT NULL,
    `hosting_id` INT UNSIGNED NULL,
    `pool_id` INT UNSIGNED NOT NULL,
    `allocated_amount` DECIMAL(15,4) NOT NULL,
    `used_amount` DECIMAL(15,4) NOT NULL DEFAULT 0,
    `allocation_type` ENUM('guaranteed', 'burstable', 'shared') NOT NULL DEFAULT 'guaranteed',
    `expires_at` DATETIME NULL,
    `is_active` TINYINT(1) NOT NULL DEFAULT 1,
    `created_at` DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
    `updated_at` DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    PRIMARY KEY (`id`),
    INDEX `idx_user_id` (`user_id`),
    INDEX `idx_pool_id` (`pool_id`),
    INDEX `idx_is_active` (`is_active`),
    CONSTRAINT `fk_allocation_user` FOREIGN KEY (`user_id`) REFERENCES `tblusers`(`id`) ON DELETE CASCADE,
    CONSTRAINT `fk_allocation_pool` FOREIGN KEY (`pool_id`) REFERENCES `mod_resource_pools`(`id`) ON DELETE CASCADE
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;

CREATE TABLE IF NOT EXISTS `mod_resource_usage_logs` (
    `id` BIGINT UNSIGNED NOT NULL AUTO_INCREMENT,
    `allocation_id` INT UNSIGNED NOT NULL,
    `used_amount` DECIMAL(15,4) NOT NULL,
    `peak_amount` DECIMAL(15,4) NOT NULL DEFAULT 0,
    `usage_percentage` DECIMAL(5,2) NOT NULL DEFAULT 0.00,
    `recorded_at` DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
    PRIMARY KEY (`id`),
    INDEX `idx_allocation_id` (`allocation_id`),
    INDEX `idx_recorded_at` (`recorded_at`),
    CONSTRAINT `fk_usage_allocation` FOREIGN KEY (`allocation_id`) REFERENCES `mod_resource_allocations`(`id`) ON DELETE CASCADE
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
```

## Module Files

### Primary Class File: resource_management.php

```php
<?php
/**
 * WHMCS Resource Management Module
 *
 * @package     WHMCS\Module\ResourceManagement
 * @author      HiTech Cloud Development Team
 * @copyright   Copyright (c) 2024 HiTech Cloud
 * @license     https://whmcs.com/license/
 * 
 * @Module
 * @License     @license
 * @Language    English
 * @Version     1.0.0
 */

if (!defined("WHMCS")) {
    die("This file cannot be accessed directly");
}

require_once __DIR__ . '/lib/ResourceManager.php';
require_once __DIR__ . '/lib/ResourcePool.php';
require_once __DIR__ . '/lib/AllocationManager.php';

use WHMCS\Module\ResourceManagement\ResourceManager;
use WHMCS\Module\ResourceManagement\ResourcePool;
use WHMCS\Module\ResourceManagement\AllocationManager;

/**
 * Resource Management Module Hooks
 */

function resource_management_cron_config($params) {
    return [
        'resource_monitor' => [
            'FriendlyName' => 'Resource Monitor',
            'Type' => 'cron',
            'Frequency' => 'hourly',
            'description' => 'Update resource usage statistics and enforce quotas',
            'runOnAllServers' => true,
        ],
    ];
}

add_hook('DailyCronJob', 1, function($params) {
    $resourceManager = new ResourceManager();
    $resourceManager->processExpiredAllocations();
    $resourceManager->generateUsageReports();
});

add_hook('UserServiceRecalculateResourceUsage', 1, function($params) {
    $allocationManager = new AllocationManager();
    return $allocationManager->updateResourceUsage($params['user_id'], $params['service_id']);
});

/**
 * Activate Module
 * 
 * @return array{success: bool, msg: string}
 */
function whmcs_resource_management_activate() {
    $resourceManager = new ResourceManager();
    return $resourceManager->activate();
}

/**
 * Deactivate Module
 * 
 * @return array{success: bool, msg: string}
 */
function whmcs_resource_management_deactivate() {
    $resourceManager = new ResourceManager();
    return $resourceManager->deactivate();
}

/**
 * Upgrade Module
 * 
 * @param string $version
 * @return array{success: bool, msg: string}
 */
function whmcs_resource_management_upgrade($version) {
    $resourceManager = new ResourceManager();
    return $resourceManager->upgrade($version);
}

/**
 * Module Configuration
 * 
 * @return array
 */
function whmcs_resource_management_config() {
    return resource_management_cron_config([]);
}

/**
 * Get Available Resource Pools
 * 
 * @param int|null $userId
 * @param string|null $poolType
 * @return array
 */
function whmcs_resource_management_get_pools($userId = null, $poolType = null) {
    $pool = new ResourcePool();
    return $pool->getAvailablePools($userId, $poolType);
}

/**
 * Allocate Resources to User
 * 
 * @param int $userId
 * @param int $poolId
 * @param float $amount
 * @param string $type
 * @param Carbon|null $expiresAt
 * @return array{success: bool, msg: string, allocation_id: int}
 */
function whmcs_resource_management_allocate($userId, $poolId, $amount, $type = 'guaranteed', $expiresAt = null) {
    $allocationManager = new AllocationManager();
    return $allocationManager->allocateResources($userId, null, $poolId, $amount, $type, $expiresAt);
}

/**
 * Get User Resource Usage
 * 
 * @param int $userId
 * @param int|null $hostingId
 * @return array
 */
function whmcs_resource_management_get_usage($userId, $hostingId = null) {
    $allocationManager = new AllocationManager();
    return $allocationManager->getUserUsage($userId, $hostingId);
}
```

### Library: ResourceManager.php

```php
<?php
/**
 * Resource Manager
 * 
 * Handles core resource management operations
 */

namespace WHMCS\Module\ResourceManagement;

use WHMCS\Database\SimpleFacade as DB;
use Carbon\Carbon;
use Illuminate\Database\Capsule\Manager as Capsule;

class ResourceManager {
    
    protected $moduleName = 'ResourceManagement';
    protected $version = '1.0.0';
    
    /**
     * Activate the module
     * 
     * @return array{success: bool, msg: string}
     */
    public function activate() {
        try {
            $this->createTables();
            $this->createDefaultPools();
            $this->registerHooks();
            
            return [
                'success' => true,
                'msg' => 'Resource Management module activated successfully',
            ];
        } catch (\Exception $e) {
            return [
                'success' => false,
                'msg' => 'Failed to activate module: ' . $e->getMessage(),
            ];
        }
    }
    
    /**
     * Deactivate the module
     * 
     * @return array{success: bool, msg: string}
     */
    public function deactivate() {
        try {
            return [
                'success' => true,
                'msg' => 'Resource Management module deactivated successfully',
            ];
        } catch (\Exception $e) {
            return [
                'success' => false,
                'msg' => 'Failed to deactivate module: ' . $e->getMessage(),
            ];
        }
    }
    
    /**
     * Upgrade the module
     * 
     * @param string $version
     * @return array{success: bool, msg: string}
     */
    public function upgrade($version) {
        // Handle upgrades based on version
        return [
            'success' => true,
            'msg' => 'Module upgraded to version ' . $this->version,
        ];
    }
    
    /**
     * Create database tables
     */
    protected function createTables() {
        $schema = Capsule::schema();
        
        if (!$schema->hasTable('mod_resource_pools')) {
            Capsule::statement("
                CREATE TABLE IF NOT EXISTS `mod_resource_pools` (
                    `id` INT UNSIGNED NOT NULL AUTO_INCREMENT,
                    `pool_name` VARCHAR(255) NOT NULL,
                    `pool_type` ENUM('compute', 'storage', 'bandwidth', 'memory') NOT NULL DEFAULT 'compute',
                    `total_capacity` DECIMAL(15,4) NOT NULL DEFAULT 0,
                    `available_capacity` DECIMAL(15,4) NOT NULL DEFAULT 0,
                    `unit` VARCHAR(50) NOT NULL DEFAULT 'units',
                    `is_active` TINYINT(1) NOT NULL DEFAULT 1,
                    `created_at` DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
                    `updated_at` DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
                    PRIMARY KEY (`id`),
                    INDEX `idx_pool_type` (`pool_type`),
                    INDEX `idx_is_active` (`is_active`)
                ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4
            ");
        }
        
        if (!$schema->hasTable('mod_resource_allocations')) {
            Capsule::statement("
                CREATE TABLE IF NOT EXISTS `mod_resource_allocations` (
                    `id` INT UNSIGNED NOT NULL AUTO_INCREMENT,
                    `user_id` INT UNSIGNED NOT NULL,
                    `hosting_id` INT UNSIGNED NULL,
                    `pool_id` INT UNSIGNED NOT NULL,
                    `allocated_amount` DECIMAL(15,4) NOT NULL,
                    `used_amount` DECIMAL(15,4) NOT NULL DEFAULT 0,
                    `allocation_type` ENUM('guaranteed', 'burstable', 'shared') NOT NULL DEFAULT 'guaranteed',
                    `expires_at` DATETIME NULL,
                    `is_active` TINYINT(1) NOT NULL DEFAULT 1,
                    `created_at` DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
                    `updated_at` DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
                    PRIMARY KEY (`id`),
                    INDEX `idx_user_id` (`user_id`),
                    INDEX `idx_pool_id` (`pool_id`),
                    INDEX `idx_is_active` (`is_active`)
                ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4
            ");
        }
        
        if (!$schema->hasTable('mod_resource_usage_logs')) {
            Capsule::statement("
                CREATE TABLE IF NOT EXISTS `mod_resource_usage_logs` (
                    `id` BIGINT UNSIGNED NOT NULL AUTO_INCREMENT,
                    `allocation_id` INT UNSIGNED NOT NULL,
                    `used_amount` DECIMAL(15,4) NOT NULL,
                    `peak_amount` DECIMAL(15,4) NOT NULL DEFAULT 0,
                    `usage_percentage` DECIMAL(5,2) NOT NULL DEFAULT 0.00,
                    `recorded_at` DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
                    PRIMARY KEY (`id`),
                    INDEX `idx_allocation_id` (`allocation_id`),
                    INDEX `idx_recorded_at` (`recorded_at`)
                ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4
            ");
        }
    }
    
    /**
     * Create default resource pools
     */
    protected function createDefaultPools() {
        $defaultPools = [
            ['name' => 'Default Compute Pool', 'type' => 'compute', 'capacity' => 1000, 'unit' => 'core-hours'],
            ['name' => 'Default Storage Pool', 'type' => 'storage', 'capacity' => 10000, 'unit' => 'GB'],
            ['name' => 'Default Bandwidth Pool', 'type' => 'bandwidth', 'capacity' => 100000, 'unit' => 'GB'],
            ['name' => 'Default Memory Pool', 'type' => 'memory', 'capacity' => 1000, 'unit' => 'GB'],
        ];
        
        foreach ($defaultPools as $pool) {
            Capsule::table('mod_resource_pools')->insertGetId([
                'pool_name' => $pool['name'],
                'pool_type' => $pool['type'],
                'total_capacity' => $pool['capacity'],
                'available_capacity' => $pool['capacity'],
                'unit' => $pool['unit'],
                'is_active' => 1,
            ]);
        }
    }
    
    /**
     * Register module hooks
     */
    protected function registerHooks() {
        // Hooks registered via add_hook in main module file
    }
    
    /**
     * Process expired allocations
     */
    public function processExpiredAllocations() {
        $now = Carbon::now();
        
        Capsule::table('mod_resources_allocations')
            ->where('expires_at', '<', $now)
            ->where('is_active', 1)
            ->update(['is_active' => 0]);
        
        Capsule::table('mod_resource_pools')
            ->whereIn('id', function($query) {
                $query->select('pool_id')
                    ->from('mod_resource_allocations')
                    ->where('expires_at', '<', $now)
                    ->where('is_active', 0);
            })
            ->increment('available_capacity', function($query) {
                $query->from('mod_resource_allocations')
                    ->whereColumn('pool_id', 'mod_resource_pools.id')
                    ->where('is_active', 0)
                    ->selectRaw('SUM(allocated_amount)');
            });
    }
    
    /**
     * Generate usage reports
     */
    public function generateUsageReports() {
        $allocations = Capsule::table('mod_resource_allocations')
            ->where('is_active', 1)
            ->get();
        
        foreach ($allocations as $allocation) {
            $usageLogId = Capsule::table('mod_resource_usage_logs')->insertGetId([
                'allocation_id' => $allocation->id,
                'used_amount' => $allocation->used_amount,
                'peak_amount' => max($allocation->used_amount, $this->getPeakUsage($allocation->id)),
                'usage_percentage' => $allocation->allocated_amount > 0 
                    ? ($allocation->used_amount / $allocation->allocated_amount) * 100 
                    : 0,
            ]);
        }
    }
    
    /**
     * Get peak usage for an allocation
     * 
     * @param int $allocationId
     * @return float
     */
    protected function getPeakUsage($allocationId) {
        return Capsule::table('mod_resource_usage_logs')
            ->where('allocation_id', $allocationId)
            ->max('used_amount') ?? 0;
    }
}
```

### Library: ResourcePool.php

```php
<?php
/**
 * Resource Pool Manager
 * 
 * Manages resource pools and capacity
 */

namespace WHMCS\Module\ResourceManagement;

use Illuminate\Database\Capsule\Manager as Capsule;
use Carbon\Carbon;

class ResourcePool {
    
    /**
     * Get available pools
     * 
     * @param int|null $userId
     * @param string|null $poolType
     * @return array
     */
    public function getAvailablePools($userId = null, $poolType = null) {
        $query = Capsule::table('mod_resource_pools')
            ->where('is_active', 1)
            ->where('available_capacity', '>', 0);
        
        if ($poolType) {
            $query->where('pool_type', $poolType);
        }
        
        return $query->get()->toArray();
    }
    
    /**
     * Get pool by ID
     * 
     * @param int $poolId
     * @return object|null
     */
    public function getPoolById($poolId) {
        return Capsule::table('mod_resource_pools')
            ->where('id', $poolId)
            ->first();
    }
    
    /**
     * Create a new resource pool
     * 
     * @param array $data
     * @return int Pool ID
     */
    public function createPool(array $data) {
        return Capsule::table('mod_resource_pools')->insertGetId([
            'pool_name' => $data['name'],
            'pool_type' => $data['type'] ?? 'compute',
            'total_capacity' => $data['capacity'],
            'available_capacity' => $data['capacity'],
            'unit' => $data['unit'] ?? 'units',
            'is_active' => 1,
            'created_at' => Carbon::now(),
            'updated_at' => Carbon::now(),
        ]);
    }
    
    /**
     * Update pool capacity
     * 
     * @param int $poolId
     * @param float $newCapacity
     * @return bool
     */
    public function updatePoolCapacity($poolId, $newCapacity) {
        $pool = $this->getPoolById($poolId);
        if (!$pool) {
            return false;
        }
        
        $used = $pool->total_capacity - $pool->available_capacity;
        $newAvailable = max(0, $newCapacity - $used);
        
        return Capsule::table('mod_resource_pools')
            ->where('id', $poolId)
            ->update([
                'total_capacity' => $newCapacity,
                'available_capacity' => $newAvailable,
                'updated_at' => Carbon::now(),
            ]);
    }
    
    /**
     * Reserve capacity from pool
     * 
     * @param int $poolId
     * @param float $amount
     * @return bool
     */
    public function reserveCapacity($poolId, $amount) {
        $pool = $this->getPoolById($poolId);
        if (!$pool || $pool->available_capacity < $amount) {
            return false;
        }
        
        return Capsule::table('mod_resource_pools')
            ->where('id', $poolId)
            ->decrement('available_capacity', $amount);
    }
    
    /**
     * Release capacity to pool
     * 
     * @param int $poolId
     * @param float $amount
     * @return bool
     */
    public function releaseCapacity($poolId, $amount) {
        $pool = $this->getPoolById($poolId);
        if (!$pool) {
            return false;
        }
        
        $newAvailable = min($pool->total_capacity, $pool->available_capacity + $amount);
        
        return Capsule::table('mod_resource_pools')
            ->where('id', $poolId)
            ->update([
                'available_capacity' => $newAvailable,
                'updated_at' => Carbon::now(),
            ]);
    }
    
    /**
     * Get utilization percentage for a pool
     * 
     * @param Capsule $poolId
     * @return float
     */
    public function getUtilization($poolId) {
        $pool = $this->getPoolById($poolId);
        if (!$pool || $pool->total_capacity == 0) {
            return 0;
        }
        
        return (($pool->total_capacity - $pool->available_capacity) / $pool->total_capacity) * 100;
    }
}
```

### Library: AllocationManager.php

```php
<?php
/**
 * Allocation Manager
 * 
 * Manages resource allocations to users
 */

namespace WHMCS\Module\ResourceManagement;

use Illuminate\Database\Capsule\Manager as Capsule;
use Carbon\Carbon;
use WHMCS\Module\ResourceManagement\ResourcePool;

class AllocationManager {
    
    protected $resourcePool;
    
    public function __construct() {
        $this->resourcePool = new ResourcePool();
    }
    
    /**
     * Allocate resources to a user
     * 
     * @param int $userId
     * @param int|null $hostingId
     * @param int $poolId
     * @param float $amount
     * @param string $type
     * @param Carbon|null $expiresAt
     * @return array{success: bool, msg: string, allocation_id: int|null}
     */
    public function allocateResources($userId, $hostingId, $poolId, $amount, $type = 'guaranteed', $expiresAt = null) {
        if (!$this->resourcePool->reserveCapacity($poolId, $amount)) {
            return [
                'success' => false,
                'msg' => 'Insufficient capacity in pool',
                'allocation_id' => null,
            ];
        }
        
        try {
            $allocationId = Capsule::table('mod_resource_allocations')->insertGetId([
                'user_id' => $userId,
                'hosting_id' => $hostingId,
                'pool_id' => $poolId,
                'allocated_amount' => $amount,
                'used_amount' => 0,
                'allocation_type' => $type,
                'expires_at' => $expiresAt,
                'is_active' => 1,
                'created_at' => Carbon::now(),
                'updated_at' => Carbon::now(),
            ]);
            
            return [
                'success' => true,
                'msg' => 'Resources allocated successfully',
                'allocation_id' => $allocationId,
            ];
        } catch (\Exception $e) {
            $this->resourcePool->releaseCapacity($poolId, $amount);
            return [
                'success' => false,
                'msg' => 'Failed to create allocation: ' . $e->getMessage(),
                'allocation_id' => null,
            ];
        }
    }
    
    /**
     * Update resource usage
     * 
     * @param int $userId
     * @param int|null $hostingId
     * @param float|null $usedAmount
     * @return array
     */
    public function updateResourceUsage($userId, $hostingId = null, $usedAmount = null) {
        $query = Capsule::table('mod_resource_allocations')
            ->where('user_id', $userId)
            ->where('is_active', 1);
        
        if ($hostingId) {
            $query->where('hosting_id', $hostingId);
        }
        
        $allocations = $query->get();
        $updated = [];
        
        foreach ($allocations as $allocation) {
            if ($usedAmount !== null) {
                Capsule::table('mod_resource_allocations')
                    ->where('id', $allocation->id)
                    ->update([
                        'used_amount' => $usedAmount,
                        'updated_at' => Carbon::now(),
                    ]);
                $updated[] = $allocation->id;
            }
        }
        
        return [
            'success' => true,
            'updated_allocations' => $updated,
        ];
    }
    
    /**
     * Get user resource usage
     * 
     * @param int $userId
     * @param int|null $hostingId
     * @return array
     */
    public function getUserUsage($userId, $hostingId = null) {
        $query = Capsule::table('mod_resource_allocations')
            ->join('mod_resource_pools', 'mod_resource_allocations.pool_id', '=', 'mod_resource_pools.id')
            ->where('mod_resource_allocations.user_id', $userId)
            ->where('mod_resource_allocations.is_active', 1);
        
        if ($hostingId) {
            $query->where('mod_resource_allocations.hosting_id', $hostingId);
        }
        
        $allocations = $query->select([
            'mod_resource_allocations.*',
            'mod_resource_pools.pool_name',
            'mod_resource_pools.pool_type',
            'mod_resource_pools.unit',
        ])->get();
        
        $usage = [];
        foreach ($allocations as $allocation) {
            $usage[] = [
                'allocation_id' => $allocation->id,
                'pool_name' => $allocation->pool_name,
                'pool_type' => $allocation->pool_type,
                'allocated' => $allocation->allocated_amount,
                'used' => $allocation->used_amount,
                'remaining' => $allocation->allocated_amount - $allocation->used_amount,
                'usage_percentage' => $allocation->allocated_amount > 0 
                    ? ($allocation->used_amount / $allocation->allocated_amount) * 100 
                    : 0,
                'unit' => $allocation->unit,
            ];
        }
        
        return $usage;
    }
    
    /**
     * Revoke an allocation
     * 
     * @param int $allocationId
     * @return bool
     */
    public function revokeAllocation($allocationId) {
        $allocation = Capsule::table('mod_resource_allocations')
            ->where('id', $allocationId)
            ->first();
        
        if (!$allocation) {
            return false;
        }
        
        Capsule::table('mod_resource_allocations')
            ->where('id', $allocation->id)
            ->update(['is_active' => 0]);
        
        return $this->resourcePool->releaseCapacity($allocation->pool_id, $allocation->allocated_amount);
    }
}
```

## Hooks Integration

### Client Area Page Hooks

```php
// Client Area Resources Widget
add_hook('ClientAreaHomepagePanels', 1, function($params) {
    $allocationManager = new \WHMCS\Module\ResourceManagement\AllocationManager();
    $usage = $allocationManager->getUserUsage($params['user']->id);
    
    return [
        [
            'title' => 'My Resource Usage',
            'name' => 'resource_usage_widget',
            'type' => 'custom',
            'data' => [
                'usage' => $usage,
            ],
            'template' => 'resource_usage_widget',
        ],
    ];
});
```

### Admin Area Hooks

```php
// Admin Dashboard Resources Overview
add_hook('AdminHomepage', 1, function($params) {
    $pool = new \WHMCS\Module\ResourceManagement\ResourcePool();
    $pools = $pool->getAvailablePools();
    
    return [
        [
            'title' => 'Resource Pool Status',
            'name' => 'resource_pool_status',
            'type' => 'system',
            'data' => [
                'pools' => $pools,
            ],
        ],
    ];
});
```

## API Functions

### REST API Endpoints

```
GET  /api/v1/resources/pools          - List all resource pools
GET  /api/v1/resources/pools/{id}      - Get pool details
GET  /api/v1/resources/usage/{userId}  - Get user resource usage
POST /api/v1/resources/allocate        - Allocate resources
POST /api/v1/resources/revoke          - Revoke allocation
```

## Activation/Deactivation Functions

```php
/**
 * Module activation
 */
function whmcs_resource_management_activate() {
    try {
        $resourceManager = new \WHMCS\Module\ResourceManagement\ResourceManager();
        return $resourceManager->activate();
    } catch (\Exception $e) {
        return [
            'success' => false,
            'msg' => 'Activation failed: ' . $e->getMessage(),
        ];
    }
}

/**
 * Module deactivation
 */
function whmcs_resource_management_deactivate() {
    try {
        $resourceManager = new \WHMCS\Module\ResourceManagement\ResourceManager();
        return $resourceManager->deactivate();
    } catch (\Exception $e) {
        return [
            'success' => false,
            'msg' => 'Deactivation failed: ' . $e->getMessage(),
        ];
    }
}
```

## Configuration Options

| Option | Type | Default | Description |
|--------|------|---------|-------------|
| default_pool_type | select | compute | Default pool type for allocations |
| enable_auto_renewal | checkbox | true | Auto-renew expired allocations |
| usage_alert_threshold | text | 80 | Alert threshold percentage |
| enable_cron_updates | checkbox | true | Enable automatic usage updates |

## Best Practices

1. Always check pool availability before allocating
2. Use transactions for allocation operations
3. Monitor resource usage via cron jobs
4. Implement proper error handling
5. Use caching for pool statistics
6. Log all allocation changes for audit purposes
