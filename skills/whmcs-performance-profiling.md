# WHMCS Performance Profiling

## Skill Description
Set up performance profiling for WHMCS modules to identify bottlenecks, optimize code, and improve response times.

## Prerequisites
- Xdebug or similar
- Profiling tools
- Performance testing knowledge

## Step-by-Step Implementation

### 1. Profiler Wrapper
```php
<?php
// includes/profiling/Profiler.php

namespace WHMCS\Module\YourModule\Profiling;

class Profiler
{
    private array $markers = [];
    private float $startTime;
    private array $memory = [];

    public function __construct()
    {
        $this->startTime = microtime(true);
        $this->memory['start'] = memory_get_usage(true);
    }

    public function mark(string $name): void
    {
        $this->markers[$name] = [
            'time' => microtime(true),
            'memory' => memory_get_usage(true)
        ];
    }

    public function measure(string $start, string $end): array
    {
        if (!isset($this->markers[$start]) || !isset($this->markers[$end])) {
            return ['error' => 'Marker not found'];
        }

        $startMarker = $this->markers[$start];
        $endMarker = $this->markers[$end];

        return [
            'duration' => $endMarker['time'] - $startMarker['time'],
            'memory_delta' => $endMarker['memory'] - $startMarker['memory']
        ];
    }

    public function getReport(): array
    {
        $totalTime = microtime(true) - $this->startTime;
        $totalMemory = memory_get_usage(true) - $this->memory['start'];

        $report = [
            'total_time' => $totalTime,
            'total_memory' => $totalMemory,
            'peak_memory' => memory_get_peak_usage(true),
            'markers' => []
        ];

        $lastMarker = null;

        foreach ($this->markers as $name => $marker) {
            $duration = $lastMarker
                ? $marker['time'] - $this->markers[$lastMarker]['time']
                : 0;

            $report['markers'][$name] = [
                'time' => $marker['time'] - $this->startTime,
                'duration' => $duration,
                'memory' => $marker['memory'] - $this->memory['start']
            ];

            $lastMarker = $name;
        }

        return $report;
    }

    public static function profile(callable $callback): array
    {
        $profiler = new self();
        $profiler->mark('start');

        $result = $callback();

        $profiler->mark('end');

        return [
            'result' => $result,
            'profile' => $profiler->getReport()
        ];
    }
}
```

### 2. Query Profiler
```php
<?php
// includes/profiling/QueryProfiler.php

namespace WHMCS\Module\YourModule\Profiling;

class QueryProfiler
{
    private array $queries = [];
    private bool $enabled = false;

    public function enable(): void
    {
        $this->enabled = true;
        $this->queries = [];
    }

    public function disable(): void
    {
        $this->enabled = false;
    }

    public function log(string $sql, array $params = [], float $time = 0): void
    {
        if (!$this->enabled) {
            return;
        }

        $this->queries[] = [
            'sql' => $sql,
            'params' => $params,
            'time' => $time,
            'timestamp' => microtime(true),
            'backtrace' => debug_backtrace(DEBUG_BACKTRACE_IGNORE_ARGS, 5)
        ];
    }

    public function getReport(): array
    {
        $totalTime = array_sum(array_column($this->queries, 'time'));
        $uniqueQueries = $this->getUniqueQueries();

        return [
            'total_queries' => count($this->queries),
            'unique_queries' => count($uniqueQueries),
            'total_time' => $totalTime,
            'avg_time' => count($this->queries) > 0 ? $totalTime / count($this->queries) : 0,
            'slow_queries' => array_filter($this->queries, fn($q) => $q['time'] > 0.1),
            'duplicates' => $this->findDuplicateQueries($uniqueQueries)
        ];
    }

    private function getUniqueQueries(): array
    {
        $unique = [];

        foreach ($this->queries as $query) {
            $key = md5($query['sql']);
            if (!isset($unique[$key])) {
                $unique[$key] = $query;
            }
        }

        return array_values($unique);
    }

    private function findDuplicateQueries(array $uniqueQueries): array
    {
        $counts = array_count_values(array_map(fn($q) => md5($q['sql']), $this->queries));

        return array_filter($uniqueQueries, fn($q) => $counts[md5($q['sql'])] > 1);
    }
}
```

## Common Pitfalls and Solutions

| Pitfall | Solution |
|---------|----------|
| Performance overhead | Enable only when needed |
| Large reports | Limit report size |

## Security Considerations

1. **Don't log sensitive data** - Filter queries
2. **Secure profiles** - Protect from access

## Testing Checklist

- [ ] Test profiling
- [ ] Identify slow queries
- [ ] Optimize bottlenecks

## Reference Links

- [Xdebug Profiling](https://xdebug.org/docs/profiler)
- [Blackfire](https://blackfire.io/)
