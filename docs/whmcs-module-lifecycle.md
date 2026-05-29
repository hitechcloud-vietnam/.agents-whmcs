# WHMCS Module Lifecycle

Complete reference for module lifecycle management (activation, deactivation, upgrade).

## Overview

All WHMCS modules follow a consistent lifecycle with activation, deactivation, and upgrade callbacks.

## Lifecycle Functions

### activate()

Called when module is activated.

```php
/**
 * Module activation callback
 * 
 * @return array ['status' => 'success'|'error', 'description' => string, 'additionaldata' => mixed]
 */
function yourmodule_activate(): array
{
    try {
        // Create required database tables
        if (!Capsule::schema()->hasTable('mod_yourmodule_data')) {
            Capsule::schema()->create('mod_yourmodule_data', function($table) {
                $table->increments('id');
                $table->integer('userid')->unsigned();
                $table->string('status', 50)->default('inactive');
                $table->timestamps();
            });
        }
        
        // Create indexes
        Capsule::schema()->table('mod_yourmodule_data', function($table) {
            $table->index(['userid']);
        });
        
        // Set default configuration
        Capsule::table('tblconfiguration')->insert([
            'setting' => 'YourModuleDefaultValue',
            'value' => 'default',
        ]);
        
        // Register hooks
        add_hook('ClientAreaPrimaryNavbar', 1, 'yourmodule_clientNavbarHook');
        add_hook('ServiceProvision', 10, 'yourmodule_serviceProvisionHook');
        
        return [
            'status' => 'success',
            'description' => 'Module activated successfully. Default settings have been configured.',
        ];
        
    } catch (Exception $e) {
        return [
            'status' => 'error',
            'description' => 'Failed to activate module: ' . $e->getMessage(),
        ];
    }
}
```

### deactivate()

Called when module is deactivated.

```php
/**
 * Module deactivation callback
 * 
 * @param array $params Contains 'force' boolean
 * @return array ['status' => 'success'|'error', 'description' => string]
 */
function yourmodule_deactivate(array $params = []): array
{
    try {
        $force = $params['force'] ?? false;
        
        // Check for active services
        if (!$force) {
            $activeCount = Capsule::table('mod_yourmodule_data')
                ->where('status', 'active')
                ->count();
            
            if ($activeCount > 0) {
                return [
                    'status' => 'error',
                    'description' => "Cannot deactivate: {$activeCount} active services exist. Force deactivation will stop all services.",
                ];
            }
        }
        
        // Remove hooks
        remove_hook('ClientAreaPrimaryNavbar', 'yourmodule_clientNavbarHook');
        remove_hook('ServiceProvision', 'yourmodule_serviceProvisionHook');
        
        // Clean up cache
        $cache = \WHMCS\File\Cache::factory('YourModule');
        $cache->deleteAll();
        
        return [
            'status' => 'success',
            'description' => 'Module deactivated successfully.',
        ];
        
    } catch (Exception $e) {
        return [
            'status' => 'error',
            'description' => 'Failed to deactivate module: ' . $e->getMessage(),
        ];
    }
}
```

### upgrade()

Called when module is upgraded to a new version.

```php
/**
 * Module upgrade callback
 * 
 * @param array $params Contains 'from_version' and 'to_version'
 * @return array ['status' => 'success'|'error', 'description' => string]
 */
function yourmodule_upgrade(array $params): array
{
    try {
        $fromVersion = $params['from_version'];
        $toVersion = $params['to_version'];
        
        logActivity("Upgrading YourModule from {$fromVersion} to {$toVersion}");
        
        // Version-specific upgrades
        if (version_compare($fromVersion, '1.1', '<')) {
            // Upgrade to 1.1
            upgradeToV1_1();
        }
        
        if (version_compare($fromVersion, '1.2', '<')) {
            // Upgrade to 1.2
            upgradeToV1_2();
        }
        
        // Update stored version
        Capsule::table('tblconfiguration')
            ->where('setting', 'YourModuleVersion')
            ->update(['value' => $toVersion]);
        
        return [
            'status' => 'success',
            'description' => "Upgraded from {$fromVersion} to {$toVersion} successfully.",
        ];
        
    } catch (Exception $e) {
        return [
            'status' => 'error',
            'description' => 'Upgrade failed: ' . $e->getMessage(),
        ];
    }
}

/**
 * Upgrade to version 1.1
 */
function upgradeToV1_1(): void
{
    // Add new column
    if (!Capsule::schema()->hasColumn('mod_yourmodule_data', 'new_field')) {
        Capsule::schema()->table('mod_yourmodule_data', function($table) {
            $table->string('new_field', 100)->default('');
        });
    }
    
    logActivity('YourModule: Applied schema upgrade for v1.1');
}

/**
 * Upgrade to version 1.2
 */
function upgradeToV1_2(): void
{
    // Create new table
    if (!Capsule::schema()->hasTable('mod_yourmodule_settings')) {
        Capsule::schema()->create('mod_yourmodule_settings', function($table) {
            $table->increments('id');
            $table->integer('userid')->unsigned();
            $table->text('settings')->nullable();
        });
    }
    
    logActivity('YourModule: Applied schema upgrade for v1.2');
}
```

## Version Management

### getVersion()

```php
/**
 * Get current module version
 */
function yourmodule_getVersion(): string
{
    $config = Capsule::table('tblconfiguration')
        ->where('setting', 'YourModuleVersion')
        ->first();
    
    return $config ? $config->value : '0.0';
}
```

### isInstalled()

```php
/**
 * Check if module is installed
 */
function yourmodule_isInstalled(): bool
{
    return Capsule::schema()->hasTable('mod_yourmodule_data');
}
```

### isConfigured()

```php
/**
 * Check if module is properly configured
 */
function yourmodule_isConfigured(): bool
{
    $requiredConfig = ['ApiKey', 'SecretKey'];
    
    foreach ($requiredConfig as $key) {
        $config = Capsule::table('tblconfiguration')
            ->where('setting', 'YourModule' . $key)
            ->first();
        
        if (!$config || empty($config->value)) {
            return false;
        }
    }
    
    return true;
}
```

## Migration Support

### migrate()

```php
/**
 * Migrate data for major version changes
 */
function yourmodule_migrate(array $params): array
{
    try {
        $migrationType = $params['type'] ?? 'full';
        
        switch ($migrationType) {
            case 'full':
                migrateAllData();
                break;
            
            case 'incremental':
                migrateIncrementalData($params['last_migration_id'] ?? 0);
                break;
            
            case 'rollback':
                rollbackMigration($params['rollback_id']);
                break;
        }
        
        return [
            'status' => 'success',
            'migrated_count' => $migratedCount,
            'next_migration_id' => $nextId,
        ];
        
    } catch (Exception $e) {
        return [
            'status' => 'error',
            'description' => $e->getMessage(),
        ];
    }
}
```

## Best Practices

1. **Always return arrays** - Never throw exceptions from lifecycle functions
2. **Check dependencies** - Verify required tables exist before operations
3. **Log activities** - Use logActivity for upgrade tracking
4. **Handle rollbacks** - Provide rollback capability for failed upgrades
5. **Clean up gracefully** - Handle forced deactivation properly
6. **Version incrementally** - Support incremental upgrades

## Related Documentation

- [whmcs-module-upgrade.md](whmcs-module-upgrade.md)
- [whmcs-module-error-handling.md](whmcs-module-error-handling.md)