---
name: whmcs-crash-analysis
description: Crash report analysis for WHMCS
category: Troubleshooting & Debugging
version: 1.0.0
---

# WHMCS Crash Analysis Skill

## Overview
This skill provides patterns for analyzing crash reports in WHMCS.

## Implementation Patterns

### Crash Analyzer
```php
<?php
/**
 * WHMCS Crash Analysis
 * Analyzes crash reports
 */

namespace WHMCS\Module\Diagnostics\Crash;

class CrashAnalyzer {
    private $db;

    public function __construct() {
        $this->db = \WHMCS\Database\Capsule::connection();
    }

    /**
     * Analyze crash
     */
    public function analyzeCrash(string $crashId): array {
        $crash = $this->getCrash($crashId);

        return [
            'crash_id' => $crashId,
            'type' => $this->categorizeCrash($crash),
            'severity' => $this->assessSeverity($crash),
            'root_cause' => $this->findRootCause($crash),
            'recommendations' => $this->generateRecommendations($crash)
        ];
    }

    /**
     * Categorize crash type
     */
    private function categorizeCrash(array $crash): string {
        $message = $crash['message'] ?? '';

        if (strpos($message, 'memory') !== false) return 'memory';
        if (strpos($message, 'timeout') !== false) return 'timeout';
        if (strpos($message, 'connection') !== false) return 'connection';

        return 'unknown';
    }

    /**
     * Generate crash report
     */
    public function generateReport(string $crashId): string {
        $analysis = $this->analyzeCrash($crashId);

        $report = "=== Crash Report ===\n";
        $report .= "ID: {$crashId}\n";
        $report .= "Type: {$analysis['type']}\n";
        $report .= "Severity: {$analysis['severity']}\n";
        $report .= "Root Cause: {$analysis['root_cause']}\n";
        $report .= "\nRecommendations:\n";

        foreach ($analysis['recommendations'] as $rec) {
            $report .= "- {$rec}\n";
        }

        return $report;
    }
}
```

## Database Schema
```sql
CREATE TABLE `mod_crash_reports` (
  `id` VARCHAR(50) PRIMARY KEY,
  `service_id` INT,
  `type` VARCHAR(50),
  `message` TEXT,
  `stack_trace' TEXT,
  `context' TEXT,
  `analyzed' TINYINT(1) DEFAULT 0,
  `created_at' DATETIME NOT NULL
);
```

## Best Practices

1. **Capture Context**: Record all relevant context
2. **Stack Traces**: Always include stack trace
3. **Categorize**: Group similar crashes
4. **Prioritize**: Severity-based response
5. **Root Cause**: Always find root cause

## Related Skills

- whmcs-stack-traces
- whmcs-debug-mode
- whmcs-incident-response
- whmcs-memory-debugging