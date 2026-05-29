---
name: whmcs-cpu-profiling
description: CPU usage analysis for WHMCS
category: Troubleshooting & Debugging
version: 1.0.0
---

# WHMCS CPU Profiling Skill

## Overview
This skill provides patterns for analyzing CPU usage in WHMCS.

## Implementation Patterns

### CPU Profiler
```php
<?php
/**
 * WHMCS CPU Profiling
 * Profiles CPU usage
 */

namespace WHMCS\Module\Diagnostics\CPU;

class CPUProfiler {
    /**
     * Get current CPU usage
     */
    public function getCurrentUsage(): array {
        $load = sys_getloadavg();

        return [
            'load_1m' => round($load[0], 2),
            'load_5m' => round($load[1], 2),
            'load_15m' => round($load[2], 2)
        ];
    }

    /**
     * Profile code execution CPU
     */
    public function profileExecution(callable $code): array {
        $startTime = microtime(true);
        $startCpu = getrusage();

        $result = $code();

        $endCpu = getrusage();
        $duration = microtime(true) - $startTime;

        return [
            'duration_ms' => round($duration * 1000, 2),
            'user_time_ms' => ($endCpu['ru_utime.tv_sec'] - $startCpu['ru_utime.tv_sec']) * 1000,
            'system_time_ms' => ($endCpu['ru_stime.tv_sec'] - $startCpu['ru_stime.tv_sec']) * 1000
        ];
    }
}
```

## Best Practices

1. **Load Monitoring**: Track server load
2. **Process CPU**: Monitor per-process CPU
3. **Profile Hot Spots**: Find high CPU code
4. **Optimization**: Optimize CPU-intensive code
5. **Caching**: Cache computed results

## Related Skills

- whmcs-profiling
- whmcs-memory-debugging
- whmcs-monitoring-agent
- whmcs-incident-response