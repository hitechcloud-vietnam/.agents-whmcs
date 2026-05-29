# WHMCS Testing - Performance Tests

## Skill Description
Implement performance testing for WHMCS modules to identify bottlenecks, verify response times, and ensure scalability.

## Prerequisites
- WHMCS 7.0+ installation
- PHP 7.4+ with PHPUnit
- Benchmark tools (phpbench, kcachegrind)
- Performance testing knowledge

## Step-by-Step Implementation

### 1. Performance Test Base
```php
<?php
// tests/Performance/PerformanceTestCase.php

namespace WHMCS\Module\YourModule\Tests\Performance;

use PHPUnit\Framework\TestCase;

abstract class PerformanceTestCase extends TestCase
{
    protected int $maxExecutionTimeMs = 100;
    protected int $maxMemoryMb = 64;

    protected function assertExecutionTime(int $expectedMs, callable $callback): void
    {
        $start = microtime(true);

        $callback();

        $duration = (microtime(true) - $start) * 1000;

        $this->assertLessThan(
            $expectedMs,
            $duration,
            "Execution took {$duration}ms, expected less than {$expectedMs}ms"
        );
    }

    protected function assertMemoryUsage(int $maxMb, callable $callback): void
    {
        $memoryBefore = memory_get_usage(true);

        $callback();

        $memoryAfter = memory_get_usage(true);
        $memoryUsed = ($memoryAfter - $memoryBefore) / 1024 / 1024;

        $this->assertLessThan(
            $maxMb,
            $memoryUsed,
            "Memory usage was {$memoryUsed}MB, expected less than {$maxMb}MB"
        );
    }
}
```

### 2. Performance Test Examples
```php
<?php
// tests/Performance/QueryPerformanceTest.php

namespace WHMCS\Module\YourModule\Tests\Performance;

class QueryPerformanceTest extends PerformanceTestCase
{
    public function testSingleQueryPerformance(): void
    {
        $this->assertExecutionTime(50, function () {
            global $db;

            $db->query("SELECT * FROM tblclients LIMIT 100");
            $db->fetchAll();
        });
    }

    public function testBatchQueryPerformance(): void
    {
        $this->assertExecutionTime(200, function () {
            global $db;

            for ($i = 0; $i < 10; $i++) {
                $db->query("SELECT * FROM tblclients LIMIT 10");
                $db->fetchAll();
            }
        });
    }

    public function testServiceListQuery(): void
    {
        $this->assertExecutionTime(100, function () {
            global $db;

            $db->query("
                SELECT h.*, c.firstname, c.lastname, p.name as product_name
                FROM tblhosting h
                JOIN tblclients c ON h.userid = c.id
                JOIN tblproducts p ON h.packageid = p.id
                WHERE h.domainstatus = 'Active'
                LIMIT 50
            ");

            $db->fetchAll();
        });
    }
}
```

## Common Pitfalls and Solutions

| Pitfall | Solution |
|---------|----------|
| Slow queries | Add indexes |
| Memory leaks | Clear memory in loops |
| N+1 queries | Use eager loading |

## Testing Checklist

- [ ] Measure query times
- [ ] Measure memory usage
- [ ] Profile slow functions
- [ ] Benchmark critical paths

## Reference Links

- [PHPBench](https://phpbench.readthedocs.io/)
- [Xdebug Profiling](https://xdebug.org/docs/profiler)
