---
name: whmcs-lock-contention
description: Lock debugging for WHMCS
category: Troubleshooting & Debugging
version: 1.0.0
---

# WHMCS Lock Debugging Skill

## Overview
This skill provides patterns for debugging database lock contention in WHMCS.

## Implementation Patterns

### Lock Debug Manager
```php
<?php
/**
 * WHMCS Lock Debugging
 * Debugs database lock issues
 */

namespace WHMCS\Module\Database;

class LockDebugManager {
    private $db;

    public function __construct() {
        $this->db = \WHMCS\Database\Capsule::connection();
    }

    /**
     * Get current locks
     */
    public function getCurrentLocks(): array {
        return $this->db->select(
            "SHOW FULL PROCESSLIST"
        );
    }

    /**
     * Find blocking queries
     */
    public function findBlockingQueries(): array {
        return $this->db->select(
            "SELECT * FROM information_schema.INNODB_LOCKS
             WHERE locked_table LIKE '%whmcs%'"
        );
    }

    /**
     * Analyze wait times
     */
    public function analyzeWaitTimes(): array {
        return $this->db->select(
            "SELECT * FROM information_schema.INNODB_LOCK_WAITS"
        );
    }
}
```

## Best Practices

1. **Monitor Locks**: Track lock wait times
2. **Short Transactions**: Keep transactions short
3. **Lock Order**: Always access tables in same order
4. **Index Usage**: Reduce lock footprint with indexes
5. **Deadlock Detection**: Monitor and log deadlocks

## Related Skills

- whmcs-deadlock-detection
- whmcs-slow-query
- whmcs-database-indexing
- whmcs-debug-mode