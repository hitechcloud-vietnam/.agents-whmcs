# WHMCS Database Indexing

## Skill Description
Implement proper database indexing strategies for WHMCS modules to optimize query performance, reduce load times, and handle high-traffic scenarios efficiently.

## Prerequisites
- WHMCS 7.0+ installation
- PHP 7.4+
- MySQL/MariaDB database
- Understanding of database query optimization

## Step-by-Step Implementation

### 1. Index Manager
```php
<?php
// includes/database/IndexManager.php

namespace WHMCS\Module\YourModule\Database;

class IndexManager
{
    private string $table;

    public function __construct(string $table)
    {
        $this->table = $table;
    }

    public function addIndex(string|array $columns, string $name = null, string $type = 'INDEX'): bool
    {
        if (is_string($columns)) {
            $columns = [$columns];
        }

        $indexName = $name ?? $this->generateIndexName($columns);

        // Check if index exists
        if ($this->indexExists($indexName)) {
            return false;
        }

        $columnList = implode(', ', $columns);
        $sql = "CREATE {$type} {$indexName} ON {$this->table} ({$columnList})";

        global $db;
        $db->query($sql);

        return true;
    }

    public function addUniqueIndex(string|array $columns, string $name = null): bool
    {
        return $this->addIndex($columns, $name, 'UNIQUE');
    }

    public function addFullTextIndex(string|array $columns, string $name = null): bool
    {
        return $this->addIndex($columns, $name, 'FULLTEXT');
    }

    public function dropIndex(string $name): bool
    {
        if (!$this->indexExists($name)) {
            return false;
        }

        global $db;
        $db->query("DROP INDEX {$name} ON {$this->table}");

        return true;
    }

    public function indexExists(string $name): bool
    {
        global $db;

        $result = $db->select(
            "SELECT COUNT(*) as cnt FROM information_schema.statistics
             WHERE table_schema = DATABASE()
             AND table_name = ?
             AND index_name = ?",
            [$this->table, $name]
        );

        return $result[0]['cnt'] > 0;
    }

    public function getIndexes(): array
    {
        global $db;

        return $db->select(
            "SELECT INDEX_NAME, COLUMN_NAME, NON_UNIQUE, SEQ_IN_INDEX
             FROM information_schema.statistics
             WHERE table_schema = DATABASE()
             AND table_name = ?
             ORDER BY INDEX_NAME, SEQ_IN_INDEX",
            [$this->table]
        );
    }

    public function analyzeTable(): bool
    {
        global $db;
        $db->query("ANALYZE TABLE {$this->table}");

        return $db->affectedRows() > 0;
    }

    public function optimizeTable(): bool
    {
        global $db;
        $db->query("OPTIMIZE TABLE {$this->table}");

        return $db->affectedRows() > 0;
    }

    private function generateIndexName(array $columns): string
    {
        return 'idx_' . $this->table . '_' . implode('_', $columns);
    }

    public static function analyzeSlowQueries(int $thresholdSeconds = 1): array
    {
        global $db;

        $result = $db->select(
            "SELECT * FROM mysql.slow_log
             WHERE db = DATABASE()
             AND query_time > ?
             ORDER BY query_time DESC
             LIMIT 50",
            [$thresholdSeconds]
        );

        return $result;
    }

    public static function explainQuery(string $sql, array $params = []): array
    {
        global $db;

        $sql = 'EXPLAIN ' . $sql;

        if (!empty($params)) {
            $sql = $db->prepare($sql, $params);
        }

        $db->query($sql);

        return $db->fetchAll();
    }
}
```

### 2. Query Optimizer
```php
<?php
// includes/database/QueryOptimizer.php

namespace WHMCS\Module\YourModule\Database;

class QueryOptimizer
{
    public static function optimizeSelect(
        string $table,
        array $columns = ['*'],
        array $conditions = [],
        array $joins = [],
        ?int $limit = null,
        ?int $offset = null,
        array $orderBy = []
    ): array {
        $sql = "SELECT " . implode(', ', $columns);
        $sql .= " FROM {$table}";

        $params = [];

        // Add joins
        foreach ($joins as $join) {
            $sql .= " {$join['type']} JOIN {$join['table']}";
            $sql .= " ON {$join['on']}";
        }

        // Add WHERE conditions
        if (!empty($conditions)) {
            $sql .= " WHERE ";
            $whereParts = [];

            foreach ($conditions as $column => $value) {
                if (is_array($value)) {
                    $placeholders = implode(',', array_fill(0, count($value), '?'));
                    $whereParts[] = "{$column} IN ({$placeholders})";
                    $params = array_merge($params, $value);
                } elseif ($value === null) {
                    $whereParts[] = "{$column} IS NULL";
                } else {
                    $whereParts[] = "{$column} = ?";
                    $params[] = $value;
                }
            }

            $sql .= implode(' AND ', $whereParts);
        }

        // Add ORDER BY
        if (!empty($orderBy)) {
            $sql .= " ORDER BY ";
            $orderParts = [];

            foreach ($orderBy as $column => $direction) {
                $orderParts[] = "{$column} {$direction}";
            }

            $sql .= implode(', ', $orderParts);
        }

        // Add LIMIT and OFFSET
        if ($limit !== null) {
            $sql .= " LIMIT ?";
            $params[] = $limit;

            if ($offset !== null) {
                $sql .= " OFFSET ?";
                $params[] = $offset;
            }
        }

        return ['sql' => $sql, 'params' => $params];
    }

    public static function buildPaginatedQuery(
        string $table,
        array $columns,
        array $conditions = [],
        int $page = 1,
        int $perPage = 20
    ): array {
        $query = self::optimizeSelect($table, $columns, $conditions);
        $countQuery = self::optimizeSelect($table, ['COUNT(*) as total'], $conditions);

        global $db;

        // Get total count
        $db->query($countQuery['sql'], $countQuery['params']);
        $result = $db->fetch();
        $total = $result['total'] ?? 0;

        // Add pagination
        $offset = ($page - 1) * $perPage;
        $query['sql'] .= " LIMIT ? OFFSET ?";
        $query['params'][] = $perPage;
        $query['params'][] = $offset;

        return [
            'query' => $query,
            'pagination' => [
                'total' => $total,
                'page' => $page,
                'per_page' => $perPage,
                'total_pages' => (int) ceil($total / $perPage)
            ]
        ];
    }

    public static function getIndexRecommendations(string $table): array
    {
        global $db;

        $recommendations = [];

        // Get slow queries involving this table
        $slowQueries = $db->select(
            "SELECT query, avg_rows_examined
             FROM mysql.slow_log
             WHERE db = DATABASE()
             AND query LIKE ?
             LIMIT 10",
            ["%{$table}%"]
        );

        foreach ($slowQueries as $query) {
            // Analyze EXPLAIN output
            $explanation = self::explainQuery($query['query']);

            foreach ($explanation as $row) {
                if (($row['type'] ?? '') === 'ALL') {
                    $recommendations[] = [
                        'type' => 'full_scan',
                        'table' => $row['table'] ?? '',
                        'message' => 'Full table scan detected',
                        'severity' => 'high'
                    ];
                }

                if (($row['possible_keys'] ?? '') && !($row['key'] ?? '')) {
                    $recommendations[] = [
                        'type' => 'missing_index',
                        'table' => $row['table'] ?? '',
                        'possible_keys' => $row['possible_keys'] ?? '',
                        'severity' => 'medium'
                    ];
                }
            }
        }

        return $recommendations;
    }
}
```

### 3. Recommended Indexes for WHMCS Tables
```php
<?php
// includes/database/RecommendedIndexes.php

namespace WHMCS\Module\YourModule\Database;

class RecommendedIndexes
{
    public static function applyToWHMCS(): void
    {
        $indexes = [
            // Clients table indexes
            'tblclients' => [
                ['columns' => ['email'], 'name' => 'idx_email', 'type' => 'UNIQUE'],
                ['columns' => ['status'], 'name' => 'idx_status'],
                ['columns' => ['created_at'], 'name' => 'idx_created'],
                ['columns' => ['groupid', 'status'], 'name' => 'idx_group_status']
            ],

            // Hosting/Services table indexes
            'tblhosting' => [
                ['columns' => ['userid'], 'name' => 'idx_userid'],
                ['columns' => ['domainstatus'], 'name' => 'idx_domainstatus'],
                ['columns' => ['nextduedate'], 'name' => 'idx_nextduedate'],
                ['columns' => ['userid', 'domainstatus'], 'name' => 'idx_user_status'],
                ['columns' => ['packageid', 'domainstatus'], 'name' => 'idx_product_status']
            ],

            // Orders table indexes
            'tblorders' => [
                ['columns' => ['userid'], 'name' => 'idx_userid'],
                ['columns' => ['status'], 'name' => 'idx_status'],
                ['columns' => ['date', 'status'], 'name' => 'idx_date_status'],
                ['columns' => ['userid', 'status'], 'name' => 'idx_user_status']
            ],

            // Invoices table indexes
            'tblinvoices' => [
                ['columns' => ['userid'], 'name' => 'idx_userid'],
                ['columns' => ['status'], 'name' => 'idx_status'],
                ['columns' => ['duedate'], 'name' => 'idx_duedate'],
                ['columns' => ['userid', 'status'], 'name' => 'idx_user_status'],
                ['columns' => ['status', 'duedate'], 'name' => 'idx_status_duedate']
            ],

            // Tickets table indexes
            'tbltickets' => [
                ['columns' => ['userid'], 'name' => 'idx_userid'],
                ['columns' => ['status'], 'name' => 'idx_status'],
                ['columns' => ['did'], 'name' => 'idx_department'],
                ['columns' => ['userid', 'status'], 'name' => 'idx_user_status']
            ]
        ];

        foreach ($indexes as $table => $tableIndexes) {
            $manager = new IndexManager($table);

            foreach ($tableIndexes as $index) {
                try {
                    $manager->addIndex(
                        $index['columns'],
                        $index['name'],
                        $index['type'] ?? 'INDEX'
                    );
                } catch (\Exception $e) {
                    // Index may already exist
                }
            }
        }
    }

    public static function createCustomTableIndexes(): void
    {
        // API Keys table indexes
        $apiKeysManager = new IndexManager('mod_yourmodule_api_keys');
        $apiKeysManager->addIndex('user_id', 'idx_user_id');
        $apiKeysManager->addUniqueIndex('api_key', 'idx_api_key');
        $apiKeysManager->addIndex(['is_active', 'expires_at'], 'idx_active_expires');

        // Metrics table indexes
        $metricsManager = new IndexManager('mod_yourmodule_api_metrics');
        $metricsManager->addIndex('created_at', 'idx_created_at');
        $metricsManager->addIndex(['endpoint', 'created_at'], 'idx_endpoint_time');
        $metricsManager->addIndex('status_code', 'idx_status_code');

        // Audit log indexes
        $auditManager = new IndexManager('mod_yourmodule_audit_log');
        $auditManager->addIndex('user_id', 'idx_user_id');
        $auditManager->addIndex('created_at', 'idx_created_at');
        $auditManager->addIndex(['user_id', 'created_at'], 'idx_user_time');
    }
}
```

### 4. Index Analysis Tool
```php
<?php
// includes/database/IndexAnalyzer.php

namespace WHMCS\Module\YourModule\Database;

class IndexAnalyzer
{
    public static function analyze(string $table): array
    {
        $manager = new IndexManager($table);
        $indexes = $manager->getIndexes();

        $analysis = [
            'table' => $table,
            'total_indexes' => count($indexes),
            'indexes' => [],
            'recommendations' => []
        ];

        // Group indexes by name
        $grouped = [];
        foreach ($indexes as $idx) {
            $name = $idx['INDEX_NAME'];
            if (!isset($grouped[$name])) {
                $grouped[$name] = [
                    'name' => $name,
                    'columns' => [],
                    'is_unique' => $idx['NON_UNIQUE'] == 0,
                    'cardinality' => 0
                ];
            }
            $grouped[$name]['columns'][] = $idx['COLUMN_NAME'];
        }

        $analysis['indexes'] = array_values($grouped);

        // Analyze query patterns
        $analysis['recommendations'] = self::generateRecommendations($table, $grouped);

        return $analysis;
    }

    private static function generateRecommendations(string $table, array $indexes): array
    {
        $recommendations = [];

        // Check for missing indexes on foreign keys
        $foreignKeys = self::getForeignKeys($table);

        foreach ($foreignKeys as $fk) {
            $found = false;

            foreach ($indexes as $index) {
                if (in_array($fk['column'], $index['columns']) && count($index['columns']) <= 2) {
                    $found = true;
                    break;
                }
            }

            if (!$found) {
                $recommendations[] = [
                    'type' => 'add_index',
                    'column' => $fk['column'],
                    'reason' => 'Foreign key column should be indexed',
                    'priority' => 'high'
                ];
            }
        }

        // Check for duplicate indexes
        $seen = [];
        foreach ($indexes as $index) {
            $key = implode(',', $index['columns']);

            if (isset($seen[$key])) {
                $recommendations[] = [
                    'type' => 'remove_duplicate',
                    'index1' => $index['name'],
                    'index2' => $seen[$key],
                    'reason' => 'Duplicate indexes on same columns',
                    'priority' => 'medium'
                ];
            } else {
                $seen[$key] = $index['name'];
            }
        }

        return $recommendations;
    }

    private static function getForeignKeys(string $table): array
    {
        global $db;

        return $db->select(
            "SELECT COLUMN_NAME, REFERENCED_TABLE_NAME, REFERENCED_COLUMN_NAME
             FROM information_schema.KEY_COLUMN_USAGE
             WHERE TABLE_SCHEMA = DATABASE()
             AND TABLE_NAME = ?
             AND REFERENCED_TABLE_NAME IS NOT NULL",
            [$table]
        );
    }

    public static function generateReport(): string
    {
        global $db;

        $tables = $db->select(
            "SELECT TABLE_NAME
             FROM information_schema.TABLES
             WHERE TABLE_SCHEMA = DATABASE()
             AND TABLE_NAME LIKE 'mod_yourmodule_%'"
        );

        $report = "# Database Index Analysis Report\n\n";
        $report .= "Generated: " . date('Y-m-d H:i:s') . "\n\n";

        foreach ($tables as $table) {
            $analysis = self::analyze($table['TABLE_NAME']);

            $report .= "## {$analysis['table']}\n\n";
            $report .= "Total Indexes: {$analysis['total_indexes']}\n\n";

            if (!empty($analysis['recommendations'])) {
                $report .= "### Recommendations\n\n";

                foreach ($analysis['recommendations'] as $rec) {
                    $report .= "- [{$rec['priority']}] {$rec['type']}: {$rec['reason']}\n";
                }

                $report .= "\n";
            }
        }

        return $report;
    }
}
```

## Common Pitfalls and Solutions

| Pitfall | Solution |
|---------|----------|
| Too many indexes | Indexes slow down INSERT/UPDATE operations |
| Wide indexes | Index only columns you query |
| Low cardinality indexes | Avoid indexing boolean or low-cardinality columns |
| Unused indexes | Monitor and remove unused indexes |
| Composite index order | Put most selective column first |

## Security Considerations

1. **Don't index sensitive data** - Avoid indexing passwords or PII
2. **Secure index metadata** - Limit access to index information
3. **Monitor index usage** - Track which indexes are actually used
4. **Plan index maintenance** - Schedule regular index optimization

## Testing Checklist

- [ ] Test index creation
- [ ] Test index removal
- [ ] Test duplicate index prevention
- [ ] Test EXPLAIN output analysis
- [ ] Test query performance before/after
- [ ] Test composite index ordering
- [ ] Test full-text index functionality
- [ ] Test index recommendations generation

## Reference Links

- [MySQL Indexes](https://dev.mysql.com/doc/refman/8.0/en/mysql-indexes.html)
- [EXPLAIN Output Format](https://dev.mysql.com/doc/refman/8.0/en/explain-output.html)
- [Index Hints](https://dev.mysql.com/doc/refman/8.0/en/index-hints.html)
- [Database Index Design](https://use-the-index-luke.com/)
