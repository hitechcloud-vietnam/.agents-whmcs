---
name: whmcs-memory-debugging
description: Memory leak detection for WHMCS
category: Troubleshooting & Debugging
version: 1.0.0
---

# WHMCS Memory Debugging Skill

## Overview
This skill provides patterns for detecting memory leaks in WHMCS.

## Implementation Patterns

### Memory Debugger
```php
<?php
/**
 * WHMCS Memory Debugging
 * Detects memory leaks
 */

namespace WHMCS\Module\Diagnostics\Memory;

class MemoryDebugger {
    private $snapshots = [];

    /**
     * Take memory snapshot
     */
    public function snapshot(string $label): array {
        $this->snapshots[$label] = [
            'usage' => memory_get_usage(true),
            'peak' => memory_get_peak_usage(true),
            'allocated' => count(debug_backtrace())
        ];

        return [
            'label' => $label,
            'memory_mb' => round(memory_get_usage(true) / 1024 / 1024, 2),
            'peak_mb' => round(memory_get_peak_usage(true) / 1024 / 1024, 2)
        ];
    }

    /**
     * Detect memory growth
     */
    public function detectGrowth(): array {
        $labels = array_keys($this->snapshots);
        if (count($labels) < 2) {
            return ['growth_detected' => false];
        }

        $first = $this->snapshots[$labels[0]];
        $last = $this->snapshots[$labels[count($labels) - 1]];

        $growthBytes = $last['usage'] - $first['usage'];
        $growthPercent = ($growthBytes / $first['usage']) * 100;

        return [
            'growth_detected' => $growthBytes > 1024 * 1024, // 1MB
            'growth_bytes' => $growthBytes,
            'growth_percent' => round($growthPercent, 2),
            'snapshots' => count($this->snapshots)
        ];
    }
}
```

## Best Practices

1. **Regular Snapshots**: Track memory at key points
2. **Trend Analysis**: Look for memory growth patterns
3. **Peak Monitoring**: Watch for peak usage
4. **Clean Up**: Unset large objects
5. **Reference Cycles**: Break circular references

## Related Skills

- whmcs-profiling
- whmcs-debug-mode
- whmcs-crash-analysis
- whmcs-cpu-profiling