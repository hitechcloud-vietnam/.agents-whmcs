---
name: whmcs-database-indexing
description: Index optimization for WHMCS
category: Performance & Monitoring
version: 1.0.0
---

# WHMCS Database Index Optimization Skill

## Overview
This skill provides patterns and implementations for optimizing database indexes in WHMCS for improved query performance.

## Implementation Patterns

### Database Index Manager
```php
<?php
/**
 * WHMCS Database Index Optimization
 * Manages database indexes
 */

namespace WHMCS\Module\Database;

class IndexManager {
    private $db;

    public function __construct() {
        $this->db = \WHMCS\Database\Capsule::connection();
    }

    /**
     * Analyze slow queries and suggest indexes
     */
    public function analyzeSlowQueries(int $limit = 100): array {
        $queries = $this->db->select(
            "SELECT * FROM mod_slow_queries
             ORDER BY execution_time DESC
             LIMIT ?",
            [$limit]
        );

        $suggestions = [];

        foreach ($queries as $query) {
            $indexSuggestion = $this->suggestIndex($query->query);
            if ($indexSuggestion) {
                $suggestions[] = $indexSuggestion;
            }
        }

        return [
            'queries_analyzed' => count($queries),
            'suggestions' => $suggestions
        ];
    }

    /**
     * Create suggested index
     */
    public function createIndex(string $table, array $columns): array {
        $indexName = 'idx_' . implode('_', $columns);

        // Check if index exists
        if ($this->indexExists($table, $indexName)) {
            return ['success' => false, 'reason' => 'Index already exists'];
        }

        $sql = "CREATE INDEX {$indexName} ON {$table} (" . implode(', ', $columns) . ")";
        $this->db->statement($sql);

        return [
            'success' => true,
            'index_name' => $indexName,
            'table' => $table,
            'columns' => $columns
        ];
    }

    /**
     * Get index recommendations
     */
    public function getRecommendations(): array {
        // Analyze common query patterns
        return [
            ['table' => 'tblhosting', 'columns' => ['userid'], 'reason' => 'User queries'],
            ['table' => 'tblhosting', 'columns' => ['domainstatus'], 'reason' => 'Status filtering'],
            ['table' => 'tblorders', 'columns' => ['userid', 'date'], 'reason' => 'Order history'],
            ['table' => 'tickets', 'columns' => ['userid', 'status'], 'reason' => 'Ticket lookup']
        ];
    }
}
```

## Database Schema
```sql
CREATE TABLE `mod_slow_queries` (
  `id` INT AUTO_INCREMENT PRIMARY KEY,
  `query` TEXT NOT NULL,
  `execution_time` DECIMAL(10,4) NOT NULL,
  `rows_examined` INT,
  `analyzed` TINYINT(1) DEFAULT 0,
  `created_at` DATETIME NOT NULL,
  INDEX `idx_execution_time` (`execution_time`)
);
```

## Common Indexes

| Table | Columns | Use Case |
|-------|---------|----------|
| tblhosting | userid | Client services |
| tblhosting | domainstatus | Status filtering |
| tblorders | userid, date | Order history |
| tblinvoices | userid, status | Invoice lookup |

## Best Practices

1. **Analyze Queries**: Use EXPLAIN to understand query plans
2. **Composite Indexes**: Consider column order in composites
3. **Monitor Impact**: Check before/after performance
4. **Avoid Over-Indexing**: Each index adds write overhead
5. **Regular Review**: Analyze and clean up unused indexes

## Related Skills

- whmcs-slow-query
- whmcs-database-design
- whmcs-query-caching
- whmcs-read-replica-config