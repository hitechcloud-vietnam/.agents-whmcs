---
name: whmcs-read-replica-config
description: Read replica setup for WHMCS
category: Performance & Monitoring
version: 1.0.0
---

# WHMCS Read Replica Configuration Skill

## Overview
This skill provides patterns for configuring read replicas in WHMCS.

## Implementation Patterns

### Read Replica Manager
```php
<?php
/**
 * WHMCS Read Replica Configuration
 * Manages read replicas
 */

namespace WHMCS\Module\Database;

class ReadReplicaManager {
    /**
     * Route query to appropriate server
     */
    public function routeQuery(string $query): array {
        $isRead = $this->isReadQuery($query);

        if ($isRead) {
            return ['server' => 'replica', 'type' => 'read'];
        }

        return ['server' => 'primary', 'type' => 'write'];
    }

    /**
     * Check if query is read-only
     */
    private function isReadQuery(string $query): bool {
        $readKeywords = ['SELECT', 'SHOW', 'DESCRIBE', 'EXPLAIN'];
        $upperQuery = strtoupper(trim($query));

        foreach ($readKeywords as $keyword) {
            if (strpos($upperQuery, $keyword) === 0) {
                return true;
            }
        }

        return false;
    }
}
```

## Best Practices

1. **Replication Lag**: Monitor replication lag
2. **Read/Write Split**: Route reads to replicas
3. **Failover**: Handle replica failures
4. **Consistency**: Consider replication lag for critical reads
5. **Connection Pooling**: Use connection pooling

## Related Skills

- whmcs-database-indexing
- whmcs-connection-pooling
- whmcs-query-caching
- whmcs-database-design