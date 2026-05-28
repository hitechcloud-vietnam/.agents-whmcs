# WHMCS Module Database Patterns

**Version:** 8.0 | **Updated:** 2026-05-28

## Overview

This guide covers database design patterns for WHMCS modules, including table creation, query patterns, migrations, and performance optimization.

---

## Table Creation

### Module Table Structure

```php
/**
 * Create module table during activation
 */
function your_module_activate()
{
    $sql = "CREATE TABLE IF NOT EXISTS `mod_your_table` (
        `id` INT(10) NOT NULL AUTO_INCREMENT PRIMARY KEY,
        `client_id` INT(10) NULL COMMENT 'Client association',
        `external_id` VARCHAR(255) NULL COMMENT 'External service ID',
        `status` ENUM('pending', 'active', 'inactive', 'suspended', 'deleted') NOT NULL DEFAULT 'pending',
        `config` TEXT NULL COMMENT 'JSON configuration',
        `data` TEXT NULL COMMENT 'Additional data (JSON)',
        `created_at` DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
        `updated_at` DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
        `deleted_at` DATETIME NULL COMMENT 'Soft delete timestamp',
        UNIQUE KEY `idx_external_id` (`external_id`),
        INDEX `idx_client_id` (`client_id`),
        INDEX `idx_status` (`status`),
        INDEX `idx_created_at` (`created_at`),
        CONSTRAINT `fk_client` FOREIGN KEY (`client_id`) 
            REFERENCES `tblclients`(`id`) ON DELETE SET NULL ON UPDATE CASCADE
    ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci";
    
    full_query($sql);
    
    return [
        'status'      => 'success',
        'description' => 'Module activated successfully',
    ];
}
```

### Metadata Table Pattern

```php
/**
 * Key-value metadata table
 */
function createMetadataTable()
{
    $sql = "CREATE TABLE IF NOT EXISTS `mod_your_metadata` (
        `id` INT(10) NOT NULL AUTO_INCREMENT PRIMARY KEY,
        `entity_type` VARCHAR(50) NOT NULL COMMENT 'e.g., client, invoice, order',
        `entity_id` INT(10) NOT NULL,
        `meta_key` VARCHAR(100) NOT NULL,
        `meta_value` TEXT NULL,
        `created_at` DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
        `updated_at` DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
        UNIQUE KEY `idx_entity_key` (`entity_type`, `entity_id`, `meta_key`),
        INDEX `idx_entity` (`entity_type`, `entity_id`),
        INDEX `idx_meta_key` (`meta_key`)
    ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4";
    
    full_query($sql);
}

/**
 * Store metadata
 */
function setMetadata($entityType, $entityId, $key, $value)
{
    return WHMCS\Database\Capsule::table('mod_your_metadata')
        ->updateOrInsert(
            ['entity_type' => $entityType, 'entity_id' => $entityId, 'meta_key' => $key],
            ['meta_value' => is_array($value) ? json_encode($value) : $value]
        );
}

/**
 * Get metadata
 */
function getMetadata($entityType, $entityId, $key = null)
{
    $query = WHMCS\Database\Capsule::table('mod_your_metadata')
        ->where('entity_type', $entityType)
        ->where('entity_id', $entityId);
    
    if ($key !== null) {
        $result = $query->where('meta_key', $key)->first();
        return $result ? json_decode($result->meta_value, true) ?? $result->meta_value : null;
    }
    
    $results = $query->get();
    $metadata = [];
    
    foreach ($results as $row) {
        $value = json_decode($row->meta_value, true);
        $metadata[$row->meta_key] = $value ?? $row->meta_value;
    }
    
    return $metadata;
}
```

## Query Patterns

### Basic CRUD Operations

```php
/**
 * Create record
 */
function createRecord($data)
{
    $data['created_at'] = date('Y-m-d H:i:s');
    
    $id = WHMCS\Database\Capsule::table('mod_your_table')
        ->insertGetId($data);
    
    return $id;
}

/**
 * Read single record
 */
function getRecord($id, $columns = ['*'])
{
    return WHMCS\Database\Capsule::table('mod_your_table')
        ->where('id', $id)
        ->whereNull('deleted_at')
        ->first($columns);
}

/**
 * Read by external ID
 */
function getRecordByExternalId($externalId, $columns = ['*'])
{
    return WHMCS\Database\Capsule::table('mod_your_table')
        ->where('external_id', $externalId)
        ->whereNull('deleted_at')
        ->first($columns);
}

/**
 * Update record
 */
function updateRecord($id, $data)
{
    $data['updated_at'] = date('Y-m-d H:i:s');
    
    return WHMCS\Database\Capsule::table('mod_your_table')
        ->where('id', $id)
        ->update($data);
}

/**
 * Soft delete record
 */
function deleteRecord($id)
{
    return WHMCS\Database\Capsule::table('mod_your_table')
        ->where('id', $id)
        ->update([
            'status'    => 'deleted',
            'deleted_at' => date('Y-m-d H:i:s'),
        ]);
}

/**
 * Hard delete (use with caution)
 */
function permanentlyDeleteRecord($id)
{
    return WHMCS\Database\Capsule::table('mod_your_table')
        ->where('id', $id)
        ->delete();
}
```

### Complex Queries

```php
/**
 * Get active records with client info
 */
function getActiveRecordsWithClient($limit = 50, $offset = 0)
{
    return WHMCS\Database\Capsule::table('mod_your_table')
        ->join('tblclients', 'mod_your_table.client_id', '=', 'tblclients.id')
        ->where('mod_your_table.status', 'active')
        ->whereNull('mod_your_table.deleted_at')
        ->select(
            'mod_your_table.*',
            'tblclients.firstname',
            'tblclients.lastname',
            'tblclients.email'
        )
        ->orderBy('mod_your_table.created_at', 'desc')
        ->limit($limit)
        ->offset($offset)
        ->get();
}

/**
 * Get records with subquery
 */
function getRecordsWithStats($status)
{
    return WHMCS\Database\Capsule::table('mod_your_table as main')
        ->select([
            'main.*',
            WHMCS\Database\Capsule::raw('(
                SELECT COUNT(*) 
                FROM mod_your_activity 
                WHERE entity_id = main.id
            ) as activity_count')
        ])
        ->where('main.status', $status)
        ->whereNull('main.deleted_at')
        ->get();
}

/**
 * Aggregate query
 */
function getStatsByStatus()
{
    return WHMCS\Database\Capsule::table('mod_your_table')
        ->whereNull('deleted_at')
        ->select(
            'status',
            WHMCS\Database\Capsule::raw('COUNT(*) as count'),
            WHMCS\Database\Capsule::raw('MAX(created_at) as last_created')
        )
        ->groupBy('status')
        ->get();
}
```

## Pagination

```php
/**
 * Paginated query
 */
function getPaginatedRecords($page = 1, $perPage = 20, $filters = [])
{
    $query = WHMCS\Database\Capsule::table('mod_your_table')
        ->whereNull('deleted_at');
    
    // Apply filters
    if (!empty($filters['status'])) {
        $query->where('status', $filters['status']);
    }
    
    if (!empty($filters['client_id'])) {
        $query->where('client_id', $filters['client_id']);
    }
    
    if (!empty($filters['search'])) {
        $query->where(function($q) use ($filters) {
            $q->where('external_id', 'like', '%' . $filters['search'] . '%')
              ->orWhere('data', 'like', '%' . $filters['search'] . '%');
        });
    }
    
    // Get total count
    $total = $query->count();
    
    // Get paginated results
    $records = $query
        ->orderBy('created_at', 'desc')
        ->limit($perPage)
        ->offset(($page - 1) * $perPage)
        ->get();
    
    return [
        'data'  => $records,
        'total' => $total,
        'page'  => $page,
        'per_page' => $perPage,
        'total_pages' => ceil($total / $perPage),
    ];
}
```

## Migrations

### Schema Migration

```php
/**
 * Migration pattern for version upgrades
 */
function your_module_upgrade($fromVersion, $toVersion)
{
    $versions = ['1.0.0', '1.0.1', '1.1.0', '1.1.1', '1.2.0'];
    
    foreach ($versions as $version) {
        if (version_compare($fromVersion, $version, '<') && 
            version_compare($version, $toVersion, '<=')) {
            
            performMigration($version);
        }
    }
    
    update_config_var('your_module_version', $toVersion);
}

function performMigration($version)
{
    switch ($version) {
        case '1.0.1':
            // Add missing index
            $sql = "ALTER TABLE `mod_your_table` 
                    ADD INDEX `idx_status_created` (`status`, `created_at`)";
            full_query($sql);
            break;
            
        case '1.1.0':
            // Create new table
            $sql = "CREATE TABLE IF NOT EXISTS `mod_your_activity` (
                `id` INT(10) NOT NULL AUTO_INCREMENT PRIMARY KEY,
                `entity_id` INT(10) NOT NULL,
                `action` VARCHAR(50) NOT NULL,
                `created_at` DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
                INDEX `idx_entity_id` (`entity_id`)
            ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4";
            full_query($sql);
            break;
            
        case '1.1.1':
            // Add new column
            $sql = "ALTER TABLE `mod_your_table` 
                    ADD COLUMN `priority` TINYINT(1) NOT NULL DEFAULT 0 AFTER `status`";
            full_query($sql);
            break;
            
        case '1.2.0':
            // Data migration
            migrateConfigToJson();
            break;
    }
}

function migrateConfigToJson()
{
    // Fetch old records with string config
    $records = WHMCS\Database\Capsule::table('mod_your_table')
        ->whereRaw("JSON_VALID(config) = 0")
        ->whereNotNull('config')
        ->get();
    
    foreach ($records as $record) {
        // Convert serialized string to JSON
        $configArray = @unserialize($record->config);
        if ($configArray !== false) {
            WHMCS\Database\Capsule::table('mod_your_table')
                ->where('id', $record->id)
                ->update(['config' => json_encode($configArray)]);
        }
    }
}
```

## Transactions

```php
/**
 * Database transaction
 */
function atomicOperation($clientId, $data)
{
    return WHMCS\Database\Capsule::transaction(function() use ($clientId, $data) {
        // Operation 1: Create main record
        $mainId = WHMCS\Database\Capsule::table('mod_your_table')
            ->insertGetId([
                'client_id'  => $clientId,
                'status'     => 'active',
                'created_at' => date('Y-m-d H:i:s'),
            ]);
        
        // Operation 2: Create related record
        WHMCS\Database\Capsule::table('mod_your_metadata')
            ->insert([
                'entity_type' => 'record',
                'entity_id'   => $mainId,
                'meta_key'    => 'initial_data',
                'meta_value'  => json_encode($data),
            ]);
        
        // Operation 3: Log activity
        WHMCS\Database\Capsule::table('mod_your_activity')
            ->insert([
                'entity_id'  => $mainId,
                'action'     => 'created',
                'created_at' => date('Y-m-d H:i:s'),
            ]);
        
        return $mainId;
    });
}
```

---

## Related Skills and Workflows

- `module-versioning-guide` - Version migration
- `database-indexing-guide` - Index creation
- `module-cache-guide` - Query caching
- `module-performance-best-practices` - Performance optimization
