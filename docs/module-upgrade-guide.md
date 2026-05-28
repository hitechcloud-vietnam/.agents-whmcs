# WHMCS Module Upgrade Guide

**Version:** 8.0 | **Updated:** 2026-05-28

## Overview

This guide covers upgrading WHMCS modules across versions, including database migrations, API changes, backward compatibility, and rollback procedures.

---

## Version Detection

### Automatic Version Detection

```php
/**
 * Check and trigger upgrade on module load
 */
function your_module_checkVersion()
{
    $storedVersion = get_config_var('your_module_version');
    $currentVersion = '1.2.0'; // From MetaData
    
    if ($storedVersion === null) {
        // First installation
        return your_module_initialSetup();
    }
    
    if (version_compare($storedVersion, $currentVersion, '<')) {
        // Upgrade needed
        return your_module_upgrade($storedVersion, $currentVersion);
    }
}

/**
 * Run upgrade
 */
function your_module_upgrade($fromVersion, $toVersion)
{
    logActivity("Your Module: Starting upgrade from {$fromVersion} to {$toVersion}");
    
    $versions = ['1.0.0', '1.0.1', '1.1.0', '1.1.1', '1.2.0'];
    
    foreach ($versions as $version) {
        if (version_compare($fromVersion, $version, '<') && 
            version_compare($version, $toVersion, '<=')) {
            
            performUpgrade($version);
        }
    }
    
    update_config_var('your_module_version', $toVersion);
    
    logActivity("Your Module: Upgrade to {$toVersion} completed");
    
    return [
        'success'     => true,
        'old_version' => $fromVersion,
        'new_version' => $toVersion,
    ];
}
```

## Database Migrations

### Schema Changes

```php
/**
 * Upgrade to 1.0.1 - Fix table indexes
 */
function upgrade_101()
{
    // Drop incorrect index and create correct one
    $sql = "ALTER TABLE `mod_your_table` 
            DROP INDEX `idx_clinet_id`,
            ADD INDEX `idx_client_id` (`client_id`)";
    full_query($sql);
    
    logActivity("Your Module: Fixed client_id index");
}

/**
 * Upgrade to 1.1.0 - Add new table
 */
function upgrade_110()
{
    // Create new table for caching
    $sql = "CREATE TABLE IF NOT EXISTS `mod_your_cache` (
        `id` INT(10) AUTO_INCREMENT PRIMARY KEY,
        `key` VARCHAR(100) NOT NULL,
        `value` TEXT,
        `expires_at` DATETIME,
        UNIQUE KEY `idx_key` (`key`),
        INDEX `idx_expires` (`expires_at`)
    ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4";
    full_query($sql);
    
    logActivity("Your Module: Created cache table");
}

/**
 * Upgrade to 1.1.1 - Add missing column
 */
function upgrade_111()
{
    // Check if column exists first
    $columns = WHMCS\Database\Capsule::schema()
        ->getColumnListing('mod_your_table');
    
    if (!in_array('priority', $columns)) {
        $sql = "ALTER TABLE `mod_your_table` 
                ADD COLUMN `priority` TINYINT(1) NOT NULL DEFAULT 0 AFTER `status`";
        full_query($sql);
    }
    
    logActivity("Your Module: Added priority column");
}

/**
 * Upgrade to 1.2.0 - Add index and change data format
 */
function upgrade_120()
{
    // Add composite index
    $sql = "ALTER TABLE `mod_your_table` 
            ADD INDEX `idx_status_priority` (`status`, `priority`)";
    full_query($sql);
    
    // Migrate old data format
    migrateToNewFormat();
}

/**
 * Perform upgrade based on version
 */
function performUpgrade($version)
{
    switch ($version) {
        case '1.0.1':
            upgrade_101();
            break;
            
        case '1.1.0':
            upgrade_110();
            break;
            
        case '1.1.1':
            upgrade_111();
            break;
            
        case '1.2.0':
            upgrade_120();
            break;
    }
}
```

### Data Migration

```php
/**
 * Migrate data to new format
 */
function migrateToNewFormat()
{
    // Fetch records with old data format
    $records = WHMCS\Database\Capsule::table('mod_your_table')
        ->whereNotNull('old_config')
        ->whereNull('new_config')
        ->limit(100)
        ->get();
    
    $migrated = 0;
    
    foreach ($records as $record) {
        // Transform old data to new format
        $newData = transformOldConfig($record->old_config);
        
        WHMCS\Database\Capsule::table('mod_your_table')
            ->where('id', $record->id)
            ->update([
                'new_config' => json_encode($newData),
                'migrated'   => 1,
            ]);
        
        $migrated++;
    }
    
    logActivity("Your Module: Migrated {$migrated} records to new format");
    
    return $migrated;
}

/**
 * Transform old configuration format
 */
function transformOldConfig($oldConfig)
{
    // Parse old serialized data
    $old = @unserialize($oldConfig);
    
    if ($old === false) {
        return [];
    }
    
    // Map to new format
    return [
        'enabled'   => $old['is_enabled'] ?? false,
        'api_key'   => $old['key'] ?? '',
        'webhook_url' => $old['callback'] ?? '',
        'settings'  => [
            'sync_interval' => $old['sync_time'] ?? 60,
            'retry_count'  => $old['attempts'] ?? 3,
        ],
    ];
}
```

## API Changes

### Handling API Versioning

```php
/**
 * Support multiple API versions
 */
function your_module_apiCall($endpoint, $data, $options = [])
{
    $version = $options['version'] ?? '1.0';
    $fallback = $options['fallback'] ?? true;
    
    $baseUrl = $version === '2.0' 
        ? 'https://api.v2.yourservice.com' 
        : 'https://api.yourservice.com';
    
    try {
        $response = makeApiCall($baseUrl . $endpoint, $data);
        return $response;
        
    } catch (ApiException $e) {
        // Fallback to v1 if v2 fails
        if ($version === '2.0' && $fallback) {
            logActivity("Your Module: Falling back to API v1");
            return your_module_apiCall($endpoint, $data, [
                'version' => '1.0',
                'fallback' => false,
            ]);
        }
        
        throw $e;
    }
}

/**
 * Check for deprecated functions
 */
function your_module_checkDeprecation()
{
    $debugEnabled = get_config_var('your_module_debug');
    
    // Check if using deprecated functions
    $deprecatedFunctions = [
        'old_process_function' => 'new_process_function',
        'legacy_api_call'      => 'api_call_v2',
    ];
    
    foreach ($deprecatedFunctions as $old => $new) {
        if (function_exists($old)) {
            if ($debugEnabled) {
                logActivity("Your Module: Function {$old} is deprecated. Use {$new} instead.");
            }
        }
    }
}
```

## Backward Compatibility

### Deprecation Pattern

```php
/**
 * Mark function as deprecated with fallback
 */
function old_function_name($param1, $param2)
{
    // Log deprecation warning in debug mode
    if (get_config_var('your_module_debug')) {
        logActivity("Your Module: old_function_name() is deprecated since 1.2.0. Use new_function_name() instead.");
    }
    
    // Call new function with mapped parameters
    return new_function_name($param1, $param2, [
        'legacy_mode' => true,
    ]);
}

/**
 * Configuration migration
 */
function migrateConfigIfNeeded()
{
    $config = get_config_var('your_module_config');
    
    // Check if using old config format
    if (is_string($config)) {
        $oldConfig = @unserialize($config);
        
        if ($oldConfig !== false) {
            // Convert to new JSON format
            $newConfig = [
                'api_key'     => $oldConfig['key'] ?? '',
                'api_secret'  => $oldConfig['secret'] ?? '',
                'test_mode'   => $oldConfig['sandbox'] ?? false,
                'debug'       => false,
            ];
            
            update_config_var('your_module_config', json_encode($newConfig));
            
            logActivity("Your Module: Migrated configuration to new format");
        }
    }
}
```

## Rollback Procedures

### Rollback Handler

```php
/**
 * Rollback to previous version
 */
function your_module_rollback($fromVersion, $toVersion)
{
    logActivity("Your Module: Starting rollback from {$fromVersion} to {$toVersion}");
    
    // Reverse migrations in reverse order
    $versions = ['1.2.0', '1.1.1', '1.1.0', '1.0.1', '1.0.0'];
    
    foreach ($versions as $version) {
        if (version_compare($toVersion, $version, '<=') && 
            version_compare($version, $fromVersion, '<')) {
            
            rollbackVersion($version);
        }
    }
    
    update_config_var('your_module_version', $toVersion);
    
    logActivity("Your Module: Rollback to {$toVersion} completed");
}

/**
 * Rollback specific version
 */
function rollbackVersion($version)
{
    switch ($version) {
        case '1.2.0':
            // Remove new indexes
            $sql = "ALTER TABLE `mod_your_table` DROP INDEX `idx_status_priority`";
            full_query_missing_ok($sql); // Don't fail if doesn't exist
            break;
            
        case '1.1.1':
            // Added column, keep data
            break;
            
        case '1.1.0':
            // Keep new table, will be removed on next deactivation
            break;
    }
    
    logActivity("Your Module: Rolled back changes from version {$version}");
}
```

## Pre-Upgrade Checklist

```php
/**
 * Run pre-upgrade checks
 */
function your_module_preUpgradeCheck($fromVersion, $toVersion)
{
    $issues = [];
    
    // Check PHP version
    if (version_compare(PHP_VERSION, '7.4', '<')) {
        $issues[] = 'PHP 7.4+ required for this upgrade';
    }
    
    // Check WHMCS version
    if (version_compare(WHMCS_VERSION, '8.0', '<')) {
        $issues[] = 'WHMCS 8.0+ required for this upgrade';
    }
    
    // Check disk space (5MB minimum)
    $freeSpace = disk_free_space(ROOTDIR);
    if ($freeSpace < 5 * 1024 * 1024) {
        $issues[] = 'Insufficient disk space (5MB minimum required)';
    }
    
    // Check for required tables
    $requiredTables = ['mod_your_table'];
    foreach ($requiredTables as $table) {
        if (!WHMCS\Database\Capsule::schema()->hasTable($table)) {
            $issues[] = "Required table {$table} not found";
        }
    }
    
    // Check database permissions
    try {
        $sql = "ALTER TABLE `mod_your_table` ADD COLUMN `_test` INT(1) NULL";
        full_query($sql);
        full_query("ALTER TABLE `mod_your_table` DROP COLUMN `_test`");
    } catch (\Exception $e) {
        $issues[] = 'Database ALTER permission required for upgrade';
    }
    
    if (!empty($issues)) {
        return [
            'success' => false,
            'issues'  => $issues,
        ];
    }
    
    return ['success' => true];
}
```

## Upgrade Execution

### Safe Upgrade Process

```php
/**
 * Execute safe upgrade
 */
function your_module_safeUpgrade($fromVersion, $toVersion)
{
    // Step 1: Pre-upgrade checks
    $checks = your_module_preUpgradeCheck($fromVersion, $toVersion);
    if (!$checks['success']) {
        return [
            'success' => false,
            'error'   => 'Pre-upgrade checks failed: ' . implode(', ', $checks['issues']),
        ];
    }
    
    // Step 2: Backup current state
    $backupId = your_module_createBackup();
    
    // Step 3: Create database backup query
    $backupSql = your_module_generateBackupSql();
    set_config_var('your_module_backup_sql', $backupSql);
    set_config_var('your_module_backup_id', $backupId);
    
    // Step 4: Execute upgrade in transaction
    try {
        WHMCS\Database\Capsule::connection()->beginTransaction();
        
        your_module_upgrade($fromVersion, $toVersion);
        
        WHMCS\Database\Capsule::connection()->commit();
        
    } catch (\Exception $e) {
        WHMCS\Database\Capsule::connection()->rollBack();
        
        // Attempt rollback
        your_module_restoreFromBackup($backupId);
        
        return [
            'success' => false,
            'error'   => 'Upgrade failed: ' . $e->getMessage(),
        ];
    }
    
    // Step 5: Verify upgrade
    if (!your_module_verifyUpgrade($toVersion)) {
        // Restore from backup
        your_module_restoreFromBackup($backupId);
        
        return [
            'success' => false,
            'error'   => 'Upgrade verification failed',
        ];
    }
    
    return [
        'success'   => true,
        'new_version' => $toVersion,
        'backup_id'   => $backupId,
    ];
}

/**
 * Create backup before upgrade
 */
function your_module_createBackup()
{
    $backupId = uniqid('backup_');
    $backupDir = ROOTDIR . '/modules/addons/your_addon/backups';
    
    if (!is_dir($backupDir)) {
        mkdir($backupDir, 0755, true);
    }
    
    $backupFile = "{$backupDir}/{$backupId}.sql";
    
    // Export module configuration
    $config = get_config_var('your_module_config');
    
    file_put_contents($backupFile, json_encode([
        'version' => get_config_var('your_module_version'),
        'config'  => $config,
        'timestamp' => date('Y-m-d H:i:s'),
    ]));
    
    return $backupId;
}

/**
 * Verify upgrade completed successfully
 */
function your_module_verifyUpgrade($version)
{
    // Check stored version
    $storedVersion = get_config_var('your_module_version');
    if ($storedVersion !== $version) {
        return false;
    }
    
    // Check required tables exist with correct schema
    $requiredColumns = [
        'mod_your_table' => ['id', 'client_id', 'status', 'created_at'],
    ];
    
    foreach ($requiredColumns as $table => $columns) {
        $existingColumns = WHMCS\Database\Capsule::schema()
            ->getColumnListing($table);
        
        foreach ($columns as $column) {
            if (!in_array($column, $existingColumns)) {
                return false;
            }
        }
    }
    
    return true;
}
```

## Post-Upgrade Actions

```php
/**
 * Post-upgrade tasks
 */
function your_module_postUpgrade($fromVersion, $toVersion)
{
    // Clear module cache
    $cache = WHMCS\Module\Cache::getInstance('YourModule');
    $cache->clear();
    
    // Clear opcode cache
    if (function_exists('opcache_reset')) {
        opcache_reset();
    }
    
    // Notify admin
    send_admin_notification(
        'system',
        'Your Module Upgraded',
        "Your Module has been upgraded from {$fromVersion} to {$toVersion}."
    );
    
    // Log upgrade completion
    logActivity("Your Module: Post-upgrade tasks completed");
    
    return true;
}
```

---

## Related Skills and Workflows

- `module-versioning-guide` - Version management
- `module-database-patterns` - Database migrations
- `module-release-checklist` - Release procedures
- `module-testing-strategies` - Testing upgrades
