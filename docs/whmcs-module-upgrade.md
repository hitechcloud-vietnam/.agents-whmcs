# WHMCS Module Upgrade Guide

## Overview

This guide covers upgrading WHMCS modules when WHMCS releases new versions or when you need to add new features.

## Version Management

### Version Constants

```php
<?php
function yourmodule_config(): array
{
    return [
        'name' => 'Your Module',
        'description' => 'Module description',
        'version' => '2.0.0',  // Update this version
        'author' => 'Your Name',
    ];
}

function yourmodule_activate(): array
{
    // Initial activation tasks
    Capsule::schema()->create('mod_yourmodule_data', function ($t) {
        $t->increments('id');
        $t->string('name');
        $t->timestamps();
    });
    
    return [
        'status' => 'success',
        'description' => 'Module activated successfully',
    ];
}

function yourmodule_upgrade(array $vars): void
{
    $currentVersion = $vars['version'];
    
    // Run sequential upgrades
    if (version_compare($currentVersion, '1.1.0', '<')) {
        upgradeTo110();
    }
    
    if (version_compare($currentVersion, '1.2.0', '<')) {
        upgradeTo120();
    }
    
    if (version_compare($currentVersion, '2.0.0', '<')) {
        upgradeTo200();
    }
}
```

## Upgrade Migrations

### Adding Columns

```php
<?php
function upgradeTo110(): void
{
    if (!Capsule::schema()->hasColumn('mod_yourmodule_data', 'new_field')) {
        Capsule::schema()->table('mod_yourmodule_data', function ($t) {
            $t->string('new_field')->nullable()->after('name');
        });
    }
    
    if (!Capsule::schema()->hasColumn('mod_yourmodule_data', 'status')) {
        Capsule::schema()->table('mod_yourmodule_data', function ($t) {
            $t->enum('status', ['active', 'inactive'])->default('active');
        });
    }
}
```

### Adding Indexes

```php
<?php
function upgradeTo120(): void
{
    // Add index for performance
    try {
        Capsule::statement(
            'ALTER TABLE mod_yourmodule_data ADD INDEX idx_status (status)'
        );
    } catch (\Exception $e) {
        // Index might already exist
    }
    
    // Create related table
    Capsule::schema()->create('mod_yourmodule_log', function ($t) {
        $t->increments('id');
        $t->integer('data_id');
        $t->string('action');
        $t->timestamp('created_at');
        $t->index('data_id');
    });
}
```

### Data Migration

```php
<?php
function upgradeTo200(): void
{
    // Migrate data format
    $oldData = Capsule::table('mod_yourmodule_config')
        ->where('setting', 'api_key')
        ->first();
    
    if ($oldData && !empty($oldData->value)) {
        // Hash the API key for security
        Capsule::table('mod_yourmodule_settings')->insert([
            'setting_name' => 'api_key_hash',
            'setting_value' => password_hash($oldData->value, PASSWORD_DEFAULT),
            'created_at' => date('Y-m-d H:i:s'),
        ]);
        
        // Remove old plain-text value
        Capsule::table('mod_yourmodule_config')
            ->where('setting', 'api_key')
            ->delete();
    }
    
    // Add new columns with defaults
    if (!Capsule::schema()->hasColumn('mod_yourmodule_data', 'priority')) {
        Capsule::schema()->table('mod_yourmodule_data', function ($t) {
            $t->integer('priority')->default(0);
        });
    }
}
```

### Schema Changes

```php
<?php
function safeSchemaChange(string $table, callable $up, callable $down): void
{
    try {
        if (!Capsule::schema()->hasTable($table)) {
            Capsule::schema()->create($table, $up);
        } else {
            Capsule::schema()->table($table, $up);
        }
    } catch (\Exception $e) {
        logModuleCall(
            'yourmodule',
            'Schema Upgrade',
            [],
            [],
            $e->getMessage()
        );
        throw $e;
    }
}

function upgradeTo210(): void
{
    safeSchemaChange(
        'mod_yourmodule_items',
        function ($t) {
            if (!$t->hasColumn('category')) {
                $t->string('category', 50)->default('general');
            }
            if (!$t->hasColumn('metadata')) {
                $t->json('metadata')->nullable();
            }
        },
        function ($t) {
            $t->dropColumn('category');
            $t->dropColumn('metadata');
        }
    );
}
```

## Version Comparison Helper

```php
<?php
class ModuleVersionManager {
    private string $currentVersion;
    
    public function __construct(string $currentVersion)
    {
        $this->currentVersion = $currentVersion;
    }
    
    public function needsUpgrade(string $targetVersion): bool
    {
        return version_compare($this->currentVersion, $targetVersion, '<');
    }
    
    public function runUpgrades(array $upgrades): void
    {
        ksort($upgrades); // Ensure ordered by version
        
        foreach ($upgrades as $version => $callback) {
            if ($this->needsUpgrade($version)) {
                $callback();
                $this->setModuleVersion($version);
            }
        }
    }
    
    private function setModuleVersion(string $version): void
    {
        Capsule::table('tbladdonmodules')
            ->where('module', 'yourmodule')
            ->update(['version' => $version]);
    }
}

// Usage in upgrade function
function yourmodule_upgrade(array $vars): void
{
    $manager = new ModuleVersionManager($vars['version']);
    
    $manager->runUpgrades([
        '1.1.0' => function() { upgradeTo110(); },
        '1.2.0' => function() { upgradeTo120(); },
        '2.0.0' => function() { upgradeTo200(); },
    ]);
}
```

## Rollback Support

```php
<?php
function yourmodule_deactivate(): array
{
    // Create backup of data
    $backupData = Capsule::table('mod_yourmodule_data')->get();
    
    // Store backup
    file_put_contents(
        __DIR__ . '/backup_' . date('Ymd') . '.json',
        json_encode($backupData)
    );
    
    // Drop tables
    Capsule::schema()->dropIfExists('mod_yourmodule_data');
    Capsule::schema()->dropIfExists('mod_yourmodule_log');
    
    return [
        'status' => 'success',
        'description' => 'Module deactivated. Data backed up.',
    ];
}
```

## Best Practices

1. **Never modify versions down** - Only increment versions
2. **Test all migrations** - Test upgrade from every version
3. **Keep backups** - Always backup before schema changes
4. **Log migrations** - Track all upgrade steps
5. **Make reversible migrations** - Support rollback when possible

## Related Documentation

- [WHMCS Module Security](/docs/whmcs-module-security.md)
- [WHMCS Database Migrations](/docs/whmcs-database-migrations.md)