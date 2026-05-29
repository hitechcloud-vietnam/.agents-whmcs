# WHMCS Database Migrations

## Skill Description
Implement database migration patterns for WHMCS modules using Laravel migrations to manage schema changes, version control, and deployment automation.

## Prerequisites
- WHMCS 7.0+ installation
- PHP 7.4+
- Basic understanding of Laravel migrations
- Database access with DDL permissions

## Step-by-Step Implementation

### 1. Migration Runner
```php
<?php
// includes/migrations/MigrationRunner.php

namespace WHMCS\Module\YourModule\Migrations;

class MigrationRunner
{
    private string $migrationsPath;
    private string $tableName = 'mod_yourmodule_migrations';

    public function __construct(string $migrationsPath = null)
    {
        $this->migrationsPath = $migrationsPath ?? dirname(__DIR__) . '/migrations';
    }

    public function run(): array
    {
        $this->ensureMigrationsTable();
        $migrations = $this->getPendingMigrations();
        $results = [];

        foreach ($migrations as $migration) {
            $results[] = $this->runMigration($migration);
        }

        return $results;
    }

    private function ensureMigrationsTable(): void
    {
        global $db;

        $db->query("
            CREATE TABLE IF NOT EXISTS {$this->tableName} (
                id INT AUTO_INCREMENT PRIMARY KEY,
                migration VARCHAR(255) NOT NULL,
                batch INT NOT NULL,
                executed_at DATETIME DEFAULT CURRENT_TIMESTAMP,
                UNIQUE KEY idx_migration (migration)
            ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4
        ");
    }

    private function getPendingMigrations(): array
    {
        global $db;

        $executed = $db->select(
            "SELECT migration FROM {$this->tableName} ORDER BY migration"
        );

        $executedNames = array_column($executed, 'migration');

        $files = glob($this->migrationsPath . '/*.php');
        $pending = [];

        foreach ($files as $file) {
            $name = basename($file, '.php');

            if (!in_array($name, $executedNames)) {
                $pending[] = [
                    'name' => $name,
                    'path' => $file
                ];
            }
        }

        sort($pending);
        return $pending;
    }

    private function runMigration(array $migration): array
    {
        $startTime = microtime(true);

        try {
            require_once $migration['path'];

            $className = $this->getMigrationClassName($migration['name']);
            $migrationClass = new $className();

            if (!method_exists($migrationClass, 'up')) {
                throw new \Exception("Migration missing up() method");
            }

            // Begin transaction
            global $db;
            $db->query('START TRANSACTION');

            $migrationClass->up();

            // Record migration
            $batch = $this->getNextBatchNumber();
            $db->insert($this->tableName, [
                'migration' => $migration['name'],
                'batch' => $batch
            ]);

            $db->query('COMMIT');

            $duration = round(microtime(true) - $startTime, 4);

            return [
                'name' => $migration['name'],
                'status' => 'success',
                'duration' => $duration
            ];

        } catch (\Exception $e) {
            global $db;
            $db->query('ROLLBACK');

            return [
                'name' => $migration['name'],
                'status' => 'failed',
                'error' => $e->getMessage()
            ];
        }
    }

    private function getNextBatchNumber(): int
    {
        global $db;

        $result = $db->select("SELECT MAX(batch) as max_batch FROM {$this->tableName}");
        return ($result[0]['max_batch'] ?? 0) + 1;
    }

    private function getMigrationClassName(string $filename): string
    {
        $parts = array_filter(explode('_', $filename));
        $classParts = array_slice($parts, 4); // Skip date prefix
        $className = '';

        foreach ($classParts as $part) {
            $className .= ucfirst(strtolower($part));
        }

        return $className;
    }

    public function rollback(int $steps = 1): array
    {
        global $db;

        $results = [];

        $migrations = $db->select(
            "SELECT * FROM {$this->tableName} ORDER BY batch DESC, id DESC LIMIT ?",
            [$steps]
        );

        foreach ($migrations as $migration) {
            $filePath = $this->migrationsPath . '/' . $migration['migration'] . '.php';

            if (!file_exists($filePath)) {
                $results[] = [
                    'name' => $migration['migration'],
                    'status' => 'failed',
                    'error' => 'Migration file not found'
                ];
                continue;
            }

            $results[] = $this->rollbackMigration($migration, $filePath);
        }

        return $results;
    }

    private function rollbackMigration(array $migration, string $filePath): array
    {
        require_once $filePath;

        $className = $this->getMigrationClassName($migration['migration']);

        try {
            $migrationClass = new $className();

            if (!method_exists($migrationClass, 'down')) {
                throw new \Exception("Migration missing down() method");
            }

            global $db;
            $db->query('START TRANSACTION');

            $migrationClass->down();

            $db->delete($this->tableName, 'id = ?', [$migration['id']]);

            $db->query('COMMIT');

            return [
                'name' => $migration['migration'],
                'status' => 'success'
            ];

        } catch (\Exception $e) {
            global $db;
            $db->query('ROLLBACK');

            return [
                'name' => $migration['migration'],
                'status' => 'failed',
                'error' => $e->getMessage()
            ];
        }
    }

    public function reset(): array
    {
        return $this->rollback(PHP_INT_MAX);
    }

    public function status(): array
    {
        global $db;

        $executed = $db->select(
            "SELECT migration, batch, executed_at FROM {$this->tableName} ORDER BY migration"
        );

        $files = glob($this->migrationsPath . '/*.php');
        $allMigrations = [];

        foreach ($files as $file) {
            $name = basename($file, '.php');
            $allMigrations[$name] = [
                'name' => $name,
                'executed' => false,
                'batch' => null,
                'executed_at' => null
            ];
        }

        foreach ($executed as $m) {
            if (isset($allMigrations[$m['migration']])) {
                $allMigrations[$m['migration']]['executed'] = true;
                $allMigrations[$m['migration']]['batch'] = $m['batch'];
                $allMigrations[$m['migration']]['executed_at'] = $m['executed_at'];
            }
        }

        return array_values($allMigrations);
    }
}
```

### 2. Migration Base Class
```php
<?php
// includes/migrations/Migration.php

namespace WHMCS\Module\YourModule\Migrations;

abstract class Migration
{
    protected string $tableName = '';
    protected string $engine = 'InnoDB';
    protected string $charset = 'utf8mb4';
    protected string $collation = 'utf8mb4_unicode_ci';

    abstract public function up(): void;

    abstract public function down(): void;

    protected function createTable(string $tableName, callable $callback): void
    {
        $blueprint = new Blueprint($tableName);
        $callback($blueprint);

        $sql = $this->compileCreateTable($blueprint);

        global $db;
        $db->query($sql);
    }

    protected function dropTable(string $tableName): void
    {
        global $db;
        $db->query("DROP TABLE IF EXISTS {$tableName}");
    }

    protected function addColumn(string $table, string $column, string $type, array $attributes = []): void
    {
        $definition = $this->compileColumn($column, $type, $attributes);
        $sql = "ALTER TABLE {$table} ADD COLUMN {$definition}";

        if (isset($attributes['after'])) {
            $sql .= " AFTER {$attributes['after']}";
        }

        global $db;
        $db->query($sql);
    }

    protected function dropColumn(string $table, string $column): void
    {
        global $db;
        $db->query("ALTER TABLE {$table} DROP COLUMN {$column}");
    }

    protected function renameColumn(string $table, string $from, string $to): void
    {
        global $db;
        $db->query("ALTER TABLE {$table} CHANGE {$from} {$to} {$this->getColumnTypeForRename($table, $from)}");
    }

    protected function createIndex(string $table, string|array $columns, string $name = null, string $type = 'INDEX'): void
    {
        if (is_string($columns)) {
            $columns = [$columns];
        }

        $indexName = $name ?? $table . '_' . implode('_', $columns) . '_index';

        $columnList = implode(', ', $columns);
        $sql = "CREATE {$type} {$indexName} ON {$table} ({$columnList})";

        global $db;
        $db->query($sql);
    }

    protected function dropIndex(string $table, string $name): void
    {
        global $db;
        $db->query("DROP INDEX {$name} ON {$table}");
    }

    protected function createForeignKey(
        string $table,
        string $column,
        string $references,
        string $on = 'CASCADE'
    ): void {
        $sql = "ALTER TABLE {$table} ADD CONSTRAINT {$table}_{$column}_fk
                FOREIGN KEY ({$column}) REFERENCES {$references} ON DELETE {$on}";

        global $db;
        $db->query($sql);
    }

    private function compileCreateTable(Blueprint $blueprint): string
    {
        $columns = [];
        $primaryKeys = [];

        foreach ($blueprint->getColumns() as $column) {
            $columns[] = $this->compileColumn(
                $column['name'],
                $column['type'],
                $column['attributes']
            );

            if (isset($column['attributes']['primary'])) {
                $primaryKeys[] = $column['name'];
            }
        }

        $sql = "CREATE TABLE {$blueprint->getTable()} (\n";
        $sql .= "  " . implode(",\n  ", $columns);

        if (!empty($primaryKeys)) {
            $sql .= ",\n  PRIMARY KEY (" . implode(', ', $primaryKeys) . ")";
        }

        $sql .= "\n) ENGINE={$this->engine} DEFAULT CHARSET={$this->charset} COLLATE={$this->collation}";

        return $sql;
    }

    private function compileColumn(string $name, string $type, array $attributes): string
    {
        $definition = "{$name} " . $this->mapType($type);

        if (isset($attributes['length'])) {
            $definition .= "({$attributes['length']})";
        }

        if (isset($attributes['precision']) && isset($attributes['scale'])) {
            $definition .= "({$attributes['precision']}, {$attributes['scale']})";
        }

        if (isset($attributes['unsigned'])) {
            $definition .= ' UNSIGNED';
        }

        if (isset($attributes['default'])) {
            $definition .= " DEFAULT '{$attributes['default']}'";
        } elseif (isset($attributes['nullable']) && $attributes['nullable']) {
            $definition .= ' NULL';
        } else {
            $definition .= ' NOT NULL';
        }

        if (isset($attributes['auto_increment'])) {
            $definition .= ' AUTO_INCREMENT';
        }

        if (isset($attributes['collation'])) {
            $definition .= " COLLATE {$attributes['collation']}";
        }

        if (isset($attributes['comment'])) {
            $definition .= " COMMENT '{$attributes['comment']}'";
        }

        return $definition;
    }

    private function mapType(string $type): string
    {
        $types = [
            'bigIncrements' => 'BIGINT',
            'bigInteger' => 'BIGINT',
            'binary' => 'BLOB',
            'boolean' => 'TINYINT(1)',
            'char' => 'CHAR',
            'date' => 'DATE',
            'datetime' => 'DATETIME',
            'decimal' => 'DECIMAL',
            'double' => 'DOUBLE',
            'enum' => 'ENUM',
            'float' => 'FLOAT',
            'increments' => 'INT',
            'integer' => 'INT',
            'json' => 'JSON',
            'jsonb' => 'JSON',
            'longText' => 'LONGTEXT',
            'mediumIncrements' => 'MEDIUMINT',
            'mediumInteger' => 'MEDIUMINT',
            'mediumText' => 'MEDIUMTEXT',
            'smallIncrements' => 'SMALLINT',
            'smallInteger' => 'SMALLINT',
            'string' => 'VARCHAR',
            'text' => 'TEXT',
            'time' => 'TIME',
            'timestamp' => 'TIMESTAMP',
            'tinyIncrements' => 'TINYINT',
            'tinyInteger' => 'TINYINT',
            'tinyText' => 'TINYTEXT',
            'uuid' => 'CHAR(36)'
        ];

        return $types[$type] ?? $type;
    }
}
```

### 3. Blueprint Class
```php
<?php
// includes/migrations/Blueprint.php

namespace WHMCS\Module\YourModule\Migrations;

class Blueprint
{
    private string $table;
    private array $columns = [];

    public function __construct(string $table)
    {
        $this->table = $table;
    }

    public function getTable(): string
    {
        return $this->table;
    }

    public function getColumns(): array
    {
        return $this->columns;
    }

    public function id(string $name = 'id'): self
    {
        return $this->addColumn($name, 'increments', ['primary' => true]);
    }

    public function bigId(string $name = 'id'): self
    {
        return $this->addColumn($name, 'bigIncrements', ['primary' => true]);
    }

    public function string(string $name, int $length = 255): self
    {
        return $this->addColumn($name, 'string', ['length' => $length]);
    }

    public function text(string $name): self
    {
        return $this->addColumn($name, 'text');
    }

    public function integer(string $name): self
    {
        return $this->addColumn($name, 'integer');
    }

    public function bigInteger(string $name): self
    {
        return $this->addColumn($name, 'bigInteger');
    }

    public function boolean(string $name): self
    {
        return $this->addColumn($name, 'boolean', ['default' => 0]);
    }

    public function datetime(string $name): self
    {
        return $this->addColumn($name, 'datetime');
    }

    public function timestamp(string $name): self
    {
        return $this->addColumn($name, 'timestamp');
    }

    public function json(string $name): self
    {
        return $this->addColumn($name, 'json');
    }

    public function decimal(string $name, int $precision = 10, int $scale = 2): self
    {
        return $this->addColumn($name, 'decimal', [
            'precision' => $precision,
            'scale' => $scale
        ]);
    }

    public function enum(string $name, array $values): self
    {
        $valuesStr = implode(',', array_map(fn($v) => "'{$v}'", $values));
        return $this->addColumn($name, 'enum', ['values' => $valuesStr]);
    }

    public function uuid(string $name): self
    {
        return $this->addColumn($name, 'uuid');
    }

    public function timestamps(): self
    {
        $this->addColumn('created_at', 'timestamp', ['nullable' => true]);
        $this->addColumn('updated_at', 'timestamp', ['nullable' => true]);
        return $this;
    }

    public function softDeletes(): self
    {
        $this->addColumn('deleted_at', 'timestamp', ['nullable' => true]);
        return $this;
    }

    public function nullable(): array
    {
        return ['nullable' => true];
    }

    public function default(mixed $value): array
    {
        return ['default' => $value];
    }

    public function unsigned(): array
    {
        return ['unsigned' => true];
    }

    private function addColumn(string $name, string $type, array $attributes = []): self
    {
        $this->columns[] = [
            'name' => $name,
            'type' => $type,
            'attributes' => $attributes
        ];

        return $this;
    }
}
```

### 4. Example Migration
```php
<?php
// migrations/2024_01_15_000001_create_api_keys_table.php

use WHMCS\Module\YourModule\Migrations\Migration;
use WHMCS\Module\YourModule\Migrations\Blueprint;

class CreateApiKeysTable extends Migration
{
    public function up(): void
    {
        $this->createTable('mod_yourmodule_api_keys', function (Blueprint $table) {
            $table->id();
            $table->bigInteger('user_id')->unsigned();
            $table->string('name', 255);
            $table->string('api_key', 64)->unique();
            $table->string('api_secret', 255);
            $table->enum('scopes', ['read', 'write', 'admin']);
            $table->boolean('is_active')->default(1);
            $table->timestamp('expires_at')->nullable();
            $table->timestamp('last_used_at')->nullable();
            $table->timestamps();
        });

        // Create index
        $this->createIndex('mod_yourmodule_api_keys', 'user_id', 'idx_user_id');
        $this->createIndex('mod_yourmodule_api_keys', 'api_key', 'idx_api_key', 'UNIQUE');
    }

    public function down(): void
    {
        $this->dropTable('mod_yourmodule_api_keys');
    }
}
```

## Common Pitfalls and Solutions

| Pitfall | Solution |
|---------|----------|
| Migration file naming | Use consistent date-based naming |
| Missing rollback | Always implement down() method |
| Schema drift | Track migrations in version control |
| Failed migrations | Use transactions with appropriate isolation |
| Column conflicts | Check column existence before adding |

## Testing Checklist

- [ ] Test migration execution
- [ ] Test rollback functionality
- [ ] Test duplicate migration prevention
- [ ] Test missing migration handling
- [ ] Test index creation
- [ ] Test foreign key constraints
- [ ] Test column type mapping
- [ ] Test migration status reporting

## Reference Links

- [Laravel Migrations](https://laravel.com/docs/database/migrations)
- [MySQL CREATE TABLE](https://dev.mysql.com/doc/refman/8.0/en/create-table.html)
- [Database Schema Migrations](https://www.martinfowler.com/articles/evodb.html)
