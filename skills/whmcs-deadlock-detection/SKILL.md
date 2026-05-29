---
name: whmcs-deadlock-detection
description: Deadlock analysis for WHMCS
category: Troubleshooting & Debugging
version: 1.0.0
---

# WHMCS Deadlock Detection Skill

## Overview
This skill provides patterns for detecting and analyzing database deadlocks in WHMCS.

## Implementation Patterns

### Deadlock Detector
```php
<?php
/**
 * WHMCS Deadlock Detection
 * Detects and analyzes deadlocks
 */

namespace WHMCS\Module\Database;

class DeadlockDetector {
    private $db;

    public function __construct() {
        $this->db = \WHMCS\Database\Capsule::connection();
    }

    /**
     * Get recent deadlocks
     */
    public function getRecentDeadlocks(int $limit = 10): array {
        return $this->db->select(
            "SHOW ENGINE INNODB STATUS"
        );
    }

    /**
     * Parse deadlock report
     */
    public function parseDeadlockReport(string $status): array {
        if (preg_match('/LATEST DETECTED DEADLOCK(.+?)---\s/s', $status, $match)) {
            $deadlockInfo = $match[1];

            return [
                'detected' => true,
                'threads' => $this->extractThreads($deadlockInfo),
                'victim' => $this->extractVictim($deadlockInfo)
            ];
        }

        return ['detected' => false];
    }

    /**
     * Log deadlock for analysis
     */
    public function logDeadlock(array $deadlock): void {
        $this->db->insert('mod_deadlock_logs', [
            'report' => json_encode($deadlock),
            'created_at' => date('Y-m-d H:i:s')
        ]);
    }
}
```

## Database Schema
```sql
CREATE TABLE `mod_deadlock_logs` (
  `id` INT AUTO_INCREMENT PRIMARY KEY,
  `report` TEXT NOT NULL,
  `analyzed` TINYINT(1) DEFAULT 0,
  `created_at' DATETIME NOT NULL
);
```

## Best Practices

1. **Log All Deadlocks**: Capture every deadlock
2. **Analyze Patterns**: Find recurring deadlock patterns
3. **Transaction Review**: Simplify transactions
4. **Index Optimization**: Reduce lock scope
5. **Retry Logic**: Implement deadlock retry

## Related Skills

- whmcs-lock-contention
- whmcs-database-indexing
- whmcs-debug-mode
- whmcs-incident-response