---
name: whmcs-connection-pooling
description: DB connection pool for WHMCS
category: Performance & Monitoring
version: 1.0.0
---

# WHMCS Connection Pooling Skill

## Overview
This skill provides patterns and implementations for database connection pooling in WHMCS.

## Implementation Patterns

### Connection Pool Manager
```php
<?php
/**
 * WHMCS Connection Pooling
 * Manages database connection pools
 */

namespace WHMCS\Module\Database;

class ConnectionPoolManager {
    private $db;

    public function __construct() {
        $this->db = \WHMCS\Database\Capsule::connection();
    }

    /**
     * Configure connection pool
     */
    public function configurePool(array $params): array {
        $config = [
            'min_connections' => $params['min'] ?? 5,
            'max_connections' => $params['max'] ?? 20,
            'idle_timeout' => $params['idle_timeout'] ?? 60,
            'connection_timeout' => $params['timeout'] ?? 30
        ];

        // Apply to database config
        Capsule::connection()->getPdo()->setAttribute(
            \PDO::ATTR_TIMEOUT,
            $config['connection_timeout']
        );

        return [
            'success' => true,
            'config' => $config
        ];
    }

    /**
     * Get pool stats
     */
    public function getPoolStats(): array {
        $status = \WHMCS\Database\Capsule::connection()->getPdo()
            ->query('SHOW PROCESSLIST')->fetchAll();

        return [
            'total_connections' => count($status),
            'active_connections' => count(array_filter($status, fn($s) => $s['Command'] === 'Query'))
        ];
    }
}
```

## Best Practices

1. **Pool Sizing**: Match to expected concurrency
2. **Timeout Settings**: Set appropriate timeouts
3. **Monitoring**: Track pool utilization
4. **Idle Cleanup**: Remove idle connections
5. **Connection Reuse**: Use persistent connections wisely

## Related Skills

- whmcs-database-indexing
- whmcs-read-replica-config
- whmcs-query-caching
- whmcs-queue-system