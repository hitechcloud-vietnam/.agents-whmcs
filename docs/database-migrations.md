# Database Migration Strategies

Database migrations are essential for evolving your module's schema while maintaining data integrity. This guide covers strategies for managing database changes in WHMCS modules.

## Migration File Structure

### Recommended Directory Structure

```
modules/addons/your_module/
├── migrations/
│   ├── 2024_01_01_000001_create_initial_tables.php
│   ├── 2024_01_15_000002_add_indexes.php
│   ├── 2024_02_01_000003_add_new_columns.php
│   └── manifest.json
├── src/
│   └── Database/
│       ├── MigrationRunner.php
│       └── migrations.php
└── setup.php
```

### Migration Manifest

```json
{
  "version": "1.0.0",
  "migrations": [
    {
      "version": "1.0.0",
      "class": "CreateInitialTables",
      "description": "Create initial database tables",
      "applied_at": null
    },
    {
      "version": "1.1.0",
      "class": "AddIndexes",
      "description": "Add performance indexes",
      "applied_at": null
    }
  ],
  "current_version": null
}
```

## Migration Class Structure

### Base Migration Class

```php
<?php
/**
 * Base migration class for WHMCS modules
 */
abstract class BaseMigration
{
    protected $connection;
    protected $moduleName;

    public function __construct()
    {
        $this->connection = Capsule::connection();
        $this->moduleName = 'your_module';
    }

    /**
     * Run the migration (up)
     */
    abstract public function up(): void;

    /**
     * Reverse the migration (down)
     */
    abstract public function down(): void;

    /**
     * Get migration description
     */
    abstract public function getDescription(): string;

    /**
     * Get migration dependencies
     */
    public function getDependencies(): array
    {
        return [];
    }

    /**
     * Check if migration was already applied
     */
    protected function hasBeenApplied(): bool
    {
        $table = $this->getMigrationsTable();

        $result = $this->connection->selectOne(
            "SELECT * FROM $table WHERE version = ? AND module = ?",
            [$this->getVersion(), $this->moduleName]
        );

        return $result !== null;
    }

    /**
     * Record migration as applied
     */
    protected function recordApplied(): void
    {
        $table = $this->getMigrationsTable();

        $this->connection->insert(
            "INSERT INTO $table (version, module, applied_at) VALUES (?, ?, ?)",
            [$this->getVersion(), $this->moduleName, date('Y-m-d H:i:s')]
        );
    }

    /**
     * Remove migration record
     */
    protected function removeRecord(): void
    {
        $table = $this->getMigrationsTable();

        $this->connection->statement(
            "DELETE FROM $table WHERE version = ? AND module = ?",
            [$this->getVersion(), $this->moduleName]
        );
    }

    protected function getMigrationsTable(): string
    {
        return 'mod_your_module_migrations';
    }

    protected function getVersion(): string
    {
        return static::VERSION;
    }
}
```

## Creating Migrations

### Table Creation Migration

```php
<?php
/**
 * Migration: Create initial tables
 *
 * Version: 1.0.0
 */
class Migration_1_0_0_CreateInitialTables extends BaseMigration
{
    const VERSION = '1.0.0';

    public function up(): void
    {
        // Create main table
        if (!$this->tableExists('mod_your_module_data')) {
            Capsule::schema()->create('mod_your_module_data', function ($table) {
                $table->increments('id');
                $table->integer('user_id')->unsigned();
                $table->string('external_id', 100)->nullable();
                $table->string('name', 255);
                $table->text('settings')->nullable();
                $table->enum('status', ['active', 'inactive', 'suspended'])
                    ->default('active');
                $table->timestamps();
                $table->softDeletes();

                $table->index('user_id');
                $table->index('external_id');
                $table->unique('external_id');
            });
        }

        // Create settings table
        if (!$this->tableExists('mod_your_module_settings')) {
            Capsule::schema()->create('mod_your_module_settings', function ($table) {
                $table->increments('id');
                $table->string('key', 100)->unique();
                $table->text('value')->nullable();
                $table->timestamp('created_at')->useCurrent();
                $table->timestamp('updated_at')->useCurrent();
            });
        }

        $this->recordApplied();
    }

    public function down(): void
    {
        Capsule::schema()->dropIfExists('mod_your_module_data');
        Capsule::schema()->dropIfExists('mod_your_module_settings');
        $this->removeRecord();
    }

    public function getDescription(): string
    {
        return 'Create initial database tables';
    }

    private function tableExists(string $table): bool
    {
        return Capsule::schema()->hasTable($table);
    }
}
```

### Column Addition Migration

```php
<?php
/**
 * Migration: Add new columns
 *
 * Version: 1.1.0
 */
class Migration_1_1_0_AddNewColumns extends BaseMigration
{
    const VERSION = '1.1.0';

    public function up(): void
    {
        // Add new column if not exists
        if (!Capsule::schema()->hasColumn('mod_your_module_data', 'priority')) {
            Capsule::schema()->table('mod_your_module_data', function ($table) {
                $table->integer('priority')->unsigned()->default(0)->after('status');
            });
        }

        // Add JSON column for metadata
        if (!Capsule::schema()->hasColumn('mod_your_module_data', 'metadata')) {
            Capsule::schema()->table('mod_your_module_data', function ($table) {
                $table->json('metadata')->nullable()->after('settings');
            });
        }

        // Add foreign key
        Capsule::statement(
            "ALTER TABLE mod_your_module_data
             ADD CONSTRAINT fk_user_id
             FOREIGN KEY (user_id) REFERENCES tblclients(id) ON DELETE CASCADE"
        );

        $this->recordApplied();
    }

    public function down(): void
    {
        // Drop foreign key first
        Capsule::statement(
            "ALTER TABLE mod_your_module_data DROP FOREIGN KEY fk_user_id"
        );

        Capsule::schema()->table('mod_your_module_data', function ($table) {
            $table->dropColumn('priority');
            $table->dropColumn('metadata');
        });

        $this->removeRecord();
    }

    public function getDescription(): string
    {
        return 'Add priority and metadata columns';
    }

    public function getDependencies(): array
    {
        return ['1.0.0'];
    }
}
```

### Index Creation Migration

```php
<?php
/**
 * Migration: Add indexes for performance
 *
 * Version: 1.2.0
 */
class Migration_1_2_0_AddIndexes extends BaseMigration
{
    const VERSION = '1.2.0';

    public function up(): void
    {
        // Add composite index
        Capsule::schema()->table('mod_your_module_data', function ($table) {
            $table->index(['status', 'created_at'], 'idx_status_created');
            $table->index(['user_id', 'status'], 'idx_user_status');
        });

        // Add fulltext index for search
        if (Capsule::connection()->getDriverName() === 'mysql') {
            Capsule::statement(
                "ALTER TABLE mod_your_module_data
                 ADD FULLTEXT INDEX idx_name_fulltext (name)"
            );
        }

        $this->recordApplied();
    }

    public function down(): void
    {
        Capsule::schema()->table('mod_your_module_data', function ($table) {
            $table->dropIndex('idx_status_created');
            $table->dropIndex('idx_user_status');
        });

        if (Capsule::connection()->getDriverName() === 'mysql') {
            Capsule::statement("ALTER TABLE mod_your_module_data DROP INDEX idx_name_fulltext");
        }

        $this->removeRecord();
    }

    public function getDescription(): string
    {
        return 'Add performance indexes';
    }
}
```

## Migration Runner

```php
<?php
/**
 * Migration runner for module
 */
class MigrationRunner
{
    private $connection;
    private $migrationsPath;
    private $moduleName;
    private array $migrations = [];

    public function __construct(string $moduleName, string $migrationsPath)
    {
        $this->moduleName = $moduleName;
        $this->migrationsPath = $migrationsPath;
        $this->connection = Capsule::connection();

        $this->ensureMigrationsTable();
        $this->loadMigrations();
    }

    /**
     * Run all pending migrations
     */
    public function run(): MigrationResult
    {
        $pending = $this->getPendingMigrations();
        $results = [];

        foreach ($pending as $migration) {
            $result = $this->runMigration($migration);
            $results[] = $result;

            if (!$result->success) {
                return new MigrationResult(false, $results);
            }
        }

        return new MigrationResult(true, $results);
    }

    /**
     * Rollback the last migration
     */
    public function rollback(): bool
    {
        $lastMigration = $this->getLastAppliedMigration();

        if (!$lastMigration) {
            return true;
        }

        return $this->reverseMigration($lastMigration);
    }

    /**
     * Rollback to a specific version
     */
    public function rollbackTo(string $targetVersion): bool
    {
        $applied = $this->getAppliedMigrations();
        $toRollback = [];

        foreach ($applied as $migration) {
            if (version_compare($migration['version'], $targetVersion, '>')) {
                $toRollback[] = $migration;
            }
        }

        // Rollback in reverse order
        foreach (array_reverse($toRollback) as $migration) {
            if (!$this->reverseMigration($migration)) {
                return false;
            }
        }

        return true;
    }

    /**
     * Reset all migrations (drop everything)
     */
    public function reset(): bool
    {
        $applied = $this->getAppliedMigrations();

        foreach (array_reverse($applied) as $migration) {
            if (!$this->reverseMigration($migration)) {
                return false;
            }
        }

        return true;
    }

    /**
     * Get current database version
     */
    public function getCurrentVersion(): ?string
    {
        $lastMigration = $this->getLastAppliedMigration();
        return $lastMigration ? $lastMigration['version'] : null;
    }

    /**
     * Get migration status
     */
    public function getStatus(): array
    {
        $applied = array_column($this->getAppliedMigrations(), 'version');
        $all = array_keys($this->migrations);

        return [
            'current' => end($applied) ?: null,
            'total_migrations' => count($all),
            'applied' => count($applied),
            'pending' => count($all) - count($applied),
            'migrations' => $this->migrations,
            'applied_versions' => $applied,
        ];
    }

    private function ensureMigrationsTable(): void
    {
        Capsule::schema()->create('mod_' . $this->moduleName . '_migrations', function ($table) {
            $table->string('version', 50);
            $table->string('module', 100);
            $table->timestamp('applied_at');
            $table->primary(['version', 'module']);
        });
    }

    private function loadMigrations(): void
    {
        $files = glob($this->migrationsPath . '/Migration_*.php');

        foreach ($files as $file) {
            require_once $file;

            $className = pathinfo($file, PATHINFO_FILENAME);
            if (class_exists($className)) {
                $reflection = new ReflectionClass($className);
                if ($reflection->isSubclassOf(BaseMigration::class) && !$reflection->isAbstract()) {
                    $this->migrations[$reflection->getConstant('VERSION')] = $className;
                }
            }
        }

        ksort($this->migrations);
    }

    private function getPendingMigrations(): array
    {
        $applied = array_column($this->getAppliedMigrations(), 'version');
        $pending = [];

        foreach ($this->migrations as $version => $className) {
            if (!in_array($version, $applied)) {
                $pending[$version] = $className;
            }
        }

        return $pending;
    }

    private function getAppliedMigrations(): array
    {
        return $this->connection->select(
            "SELECT * FROM mod_{$this->moduleName}_migrations WHERE module = ? ORDER BY applied_at ASC",
            [$this->moduleName]
        );
    }

    private function getLastAppliedMigration(): ?array
    {
        return $this->connection->selectOne(
            "SELECT * FROM mod_{$this->moduleName}_migrations WHERE module = ? ORDER BY applied_at DESC LIMIT 1",
            [$this->moduleName]
        );
    }

    private function runMigration(array $migration): SingleMigrationResult
    {
        $className = $migration['class'];
        $version = array_search($className, $this->migrations) ?: key($this->migrations);

        try {
            $instance = new $className();
            $instance->up();

            return new SingleMigrationResult(true, "Migrated to $version");
        } catch (Exception $e) {
            return new SingleMigrationResult(false, "Failed: " . $e->getMessage());
        }
    }

    private function reverseMigration(array $migration): bool
    {
        $className = $migration['class'];

        try {
            $instance = new $className();
            $instance->down();
            return true;
        } catch (Exception $e) {
            logActivity("Migration rollback failed: " . $e->getMessage());
            return false;
        }
    }
}
```

## Safe Migration Practices

### Data Preservation Strategy

```php
<?php
/**
 * Safe data migration with backup
 */
class SafeDataMigration extends BaseMigration
{
    const VERSION = '2.0.0';

    public function up(): void
    {
        // Step 1: Create backup table
        $this->createBackupTable();

        // Step 2: Copy existing data
        $this->backupData();

        // Step 3: Modify structure
        $this->modifyStructure();

        // Step 4: Restore data
        $this->restoreData();

        // Step 5: Clean up backup
        $this->dropBackupTable();

        $this->recordApplied();
    }

    private function createBackupTable(): void
    {
        Capsule::statement(
            "CREATE TABLE mod_your_module_data_backup AS
             SELECT * FROM mod_your_module_data"
        );
    }

    private function backupData(): void
    {
        logActivity("Data backup created for migration " . static::VERSION);
    }

    private function modifyStructure(): void
    {
        Capsule::schema()->table('mod_your_module_data', function ($table) {
            $table->renameColumn('name', 'full_name');
            $table->string('email', 255)->nullable()->after('full_name');
        });
    }

    private function restoreData(): void
    {
        // Only restore if something goes wrong
        // Implementation depends on specific needs
    }

    private function dropBackupTable(): void
    {
        Capsule::schema()->dropIfExists('mod_your_module_data_backup');
    }
}
```

### Transaction-Based Migration

```php
<?php
/**
 * Transaction-wrapped migration
 */
class TransactionalMigration extends BaseMigration
{
    const VERSION = '2.1.0';

    public function up(): void
    {
        Capsule::connection()->transaction(function () {
            // Multiple operations in one transaction
            Capsule::schema()->table('mod_your_module_data', function ($table) {
                $table->integer('new_column')->default(0);
            });

            // Update existing rows
            Capsule::table('mod_your_module_data')
                ->whereNull('settings')
                ->update(['settings' => json_encode([])]);

            // Add index
            Capsule::schema()->table('mod_your_module_data', function ($table) {
                $table->index('new_column');
            });
        });

        $this->recordApplied();
    }
}
```

## Migration Testing

```php
<?php
/**
 * Migration test cases
 */
class MigrationTest extends WHMCS\Test\Unit\TestCase
{
    private $migrationsPath;
    private $runner;

    protected function setUp(): void
    {
        parent::setUp();

        $this->migrationsPath = __DIR__ . '/migrations';
        $this->runner = new MigrationRunner('test_module', $this->migrationsPath);

        // Reset to clean state
        $this->runner->reset();
    }

    public function testAllMigrationsRunSuccessfully(): void
    {
        $result = $this->runner->run();

        $this->assertTrue($result->success);
        $this->assertEmpty($result->errors);
    }

    public function testRollbackRestoresPreviousState(): void
    {
        // Run all migrations
        $this->runner->run();

        // Get current version
        $versionAfter = $this->runner->getCurrentVersion();

        // Rollback
        $this->runner->rollback();

        // Check version is previous
        $versionBefore = $this->runner->getCurrentVersion();
        $this->assertLessThan($versionAfter, $versionBefore);
    }

    public function testIdempotentMigration(): void
    {
        // Run migration
        $this->runner->run();

        // Run again - should not fail
        $result = $this->runner->run();

        $this->assertTrue($result->success);
    }

    public function testPendingMigrationsAreCorrect(): void
    {
        $status = $this->runner->getStatus();

        $this->assertEquals(3, $status['total_migrations']);
        $this->assertEquals(0, $status['applied']);
        $this->assertEquals(3, $status['pending']);
    }
}
```

## Best Practices Summary

1. **Use timestamp prefixes** - Prevents merge conflicts
2. **Keep migrations small** - Single responsibility
3. **Always provide rollback** - Enables recovery
4. **Test rollback scenarios** - Verify data integrity
5. **Use transactions** - Atomic operations
6. **Create backups** - Data preservation
7. **Document changes** - Clear migration purpose
8. **Validate dependencies** - Order matters

## Related Patterns

- [Module Versioning](./module-versioning.md) - Version management
- [Database Schema Design](./database-schema-design.md) - Schema planning
- [Repository Pattern](./repository-pattern.md) - Data access layer