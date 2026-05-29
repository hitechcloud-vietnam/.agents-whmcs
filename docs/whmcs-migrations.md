# WHMCS Database Migrations

## Overview

Database migrations manage database schema changes in a versioned, reversible manner.

## Migration Structure

```php
<?php
// Version 1.0.0
if (!Capsule::schema()->hasTable('mod_yourmodule_table')) {
    Capsule::schema()->create('mod_yourmodule_table', function ($t) {
        $t->increments('id');
        $t->string('name');
        $t->timestamps();
    });
}

// Version 1.1.0 - Column addition
if (!Capsule::schema()->hasColumn('mod_yourmodule_table', 'description')) {
    Capsule::schema()->table('mod_yourmodule_table', function ($t) {
        $t->text('description')->nullable()->after('name');
    });
}
```

## Migration in Module

```php
<?php
function yourmodule_activate(): array
{
    try {
        // Create initial tables
        Capsule::schema()->create('mod_yourmodule_data', function ($t) {
            $t->increments('id');
            $t->integer('user_id')->unsigned();
            $t->string('title', 200);
            $t->text('content')->nullable();
            $t->decimal('amount', 10, 2)->default(0);
            $t->enum('status', ['pending', 'active', 'completed'])->default('pending');
            $t->timestamps();
            
            $t->index('user_id');
            $t->index('status');
            
            $t->foreign('user_id')
                ->references('id')
                ->on('tblclients')
                ->onDelete('cascade');
        });
        
        Capsule::schema()->create('mod_yourmodule_settings', function ($t) {
            $t->increments('id');
            $t->string('key_name', 100)->unique();
            $t->text('key_value')->nullable();
            $t->timestamps();
        });
        
        return [
            'status' => 'success',
            'description' => 'Module activated',
        ];
        
    } catch (Exception $e) {
        return [
            'status' => 'error',
            'description' => $e->getMessage(),
        ];
    }
}

function yourmodule_upgrade(array $vars): void
{
    $currentVersion = $vars['version'];
    
    // Version-specific migrations
    if (version_compare($currentVersion, '1.1.0', '<')) {
        migrateTo110();
    }
    
    if (version_compare($currentVersion, '1.2.0', '<')) {
        migrateTo120();
    }
    
    if (version_compare($currentVersion, '2.0.0', '<')) {
        migrateTo200();
    }
}

function migrateTo110(): void
{
    if (!Capsule::schema()->hasColumn('mod_yourmodule_data', 'category')) {
        Capsule::schema()->table('mod_yourmodule_data', function ($t) {
            $t->string('category', 50)->default('general')->after('title');
        });
    }
}

function migrateTo120(): void
{
    // Add indexes
    try {
        Capsule::statement(
            'CREATE INDEX idx_category ON mod_yourmodule_data(category)'
        );
    } catch (Exception $e) {
        // Index might exist
    }
    
    // Create log table
    if (!Capsule::schema()->hasTable('mod_yourmodule_log')) {
        Capsule::schema()->create('mod_yourmodule_log', function ($t) {
            $t->increments('id');
            $t->integer('data_id')->unsigned();
            $t->string('action', 50);
            $t->text('details')->nullable();
            $t->timestamp('created_at');
            
            $t->index('data_id');
        });
    }
}

function migrateTo200(): void
{
    // Data migration - move data to new format
    $records = Capsule::table('mod_yourmodule_data')->get();
    
    foreach ($records as $record) {
        // Convert old format to new JSON format
        $metadata = [
            'original_status' => $record->status,
            'migrated_at' => date('Y-m-d H:i:s'),
        ];
        
        Capsule::table('mod_yourmodule_data')
            ->where('id', $record->id)
            ->update([
                'metadata' => json_encode($metadata),
            ]);
    }
    
    // Add new column if missing
    if (!Capsule::schema()->hasColumn('mod_yourmodule_data', 'priority')) {
        Capsule::schema()->table('mod_yourmodule_data', function ($t) {
            $t->integer('priority')->default(0);
        });
    }
}
```

## Safe Migrations

```php
<?php
class MigrationHelper
{
    public static function safeCreateTable(string $table, callable $callback): bool
    {
        try {
            if (Capsule::schema()->hasTable($table)) {
                return false;
            }
            
            Capsule::schema()->create($table, $callback);
            return true;
            
        } catch (Exception $e) {
            logModuleCall('migration', 'createTable', $table, [], $e->getMessage());
            return false;
        }
    }
    
    public static function safeAddColumn(string $table, string $column, callable $callback): bool
    {
        try {
            if (Capsule::schema()->hasColumn($table, $column)) {
                return false;
            }
            
            Capsule::schema()->table($table, $callback);
            return true;
            
        } catch (Exception $e) {
            logModuleCall('migration', 'addColumn', "{$table}.{$column}", [], $e->getMessage());
            return false;
        }
    }
    
    public static function safeAddIndex(string $table, string|array $columns, ?string $name = null): bool
    {
        try {
            $indexName = $name ?? 'idx_' . implode('_', (array) $columns);
            
            Capsule::schema()->table($table, function ($t) use ($columns, $indexName) {
                $t->index($columns, $indexName);
            });
            
            return true;
            
        } catch (Exception $e) {
            // Index might already exist
            return false;
        }
    }
}

// Usage
MigrationHelper::safeCreateTable('mod_yourmodule_data', function ($t) {
    $t->increments('id');
    $t->string('name');
    $t->timestamps();
});

MigrationHelper::safeAddColumn('mod_yourmodule_data', 'description', function ($t) {
    $t->text('description')->nullable();
});
```

## Rollback Migrations

```php
<?php
function yourmodule_deactivate(): array
{
    try {
        // Create backup of all data
        $backup = [];
        
        if (Capsule::schema()->hasTable('mod_yourmodule_data')) {
            $backup['data'] = Capsule::table('mod_yourmodule_data')->get();
        }
        
        if (Capsule::schema()->hasTable('mod_yourmodule_settings')) {
            $backup['settings'] = Capsule::table('mod_yourmodule_settings')->get();
        }
        
        // Save backup to file
        $backupPath = __DIR__ . '/backups/deactivate_' . date('Ymd_His') . '.json';
        file_put_contents($backupPath, json_encode($backup, JSON_PRETTY_PRINT));
        
        // Drop tables
        Capsule::schema()->dropIfExists('mod_yourmodule_log');
        Capsule::schema()->dropIfExists('mod_yourmodule_settings');
        Capsule::schema()->dropIfExists('mod_yourmodule_data');
        
        return [
            'status' => 'success',
            'description' => 'Module deactivated. Backup saved.',
        ];
        
    } catch (Exception $e) {
        return [
            'status' => 'error',
            'description' => $e->getMessage(),
        ];
    }
}
```

## Data Migration Patterns

```php
<?php
// Batch migration with progress tracking
function migrateLargeDataSet(): void
{
    $batchSize = 1000;
    $offset = 0;
    $totalProcessed = 0;
    
    do {
        $records = Capsule::table('old_table')
            ->offset($offset)
            ->limit($batchSize)
            ->get();
        
        foreach ($records as $record) {
            processRecord($record);
            $totalProcessed++;
        }
        
        $offset += $batchSize;
        
        // Update progress (for long migrations)
        Capsule::table('mod_migration_progress')
            ->updateOrInsert(
                ['task' => 'data_migration'],
                [
                    'processed' => $totalProcessed,
                    'last_run' => date('Y-m-d H:i:s'),
                ]
            );
        
    } while (count($records) === $batchSize);
}

// Conditional migration
function conditionalMigration(): void
{
    // Only migrate if data exists
    $hasOldData = Capsule::table('old_table')->exists();
    
    if ($hasOldData) {
        $oldRecords = Capsule::table('old_table')->get();
        
        foreach ($oldRecords as $old) {
            // Transform and insert to new table
            Capsule::table('new_table')->insert([
                'field1' => transformField1($old->field1),
                'field2' => transformField2($old->field2),
                'created_at' => date('Y-m-d H:i:s'),
            ]);
        }
    }
}
```

## Best Practices

1. **Always backup data** - Before modifying schemas
2. **Test migrations** - Test on staging first
3. **Make idempotent** - Check existence before creating
4. **Log migrations** - Track all schema changes
5. **Version sequentially** - Use semantic versioning

## Related Documentation

- [WHMCS Capsule Schema](/docs/whmcs-capsule-schema.md)
- [WHMCS Capsule Queries](/docs/whmcs-capsule-queries.md)