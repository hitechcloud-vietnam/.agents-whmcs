# WHMCS Module Performance Audit Workflow
# Version: 1.0 | Updated: 2026-05-28

## Purpose

Audit and optimize WHMCS module performance.

## Steps

### 1. Identify Performance Bottlenecks

```php
// Enable query logging
Capsule::connection()->enableQueryLog();

// Run your operation
$operation();

// Analyze queries
$queries = Capsule::getQueryLog();
foreach ($queries as $q) {
    echo $q['query'] . "\n";
}
```

### 2. Database Optimization

```
Checklist:
□ Add indexes to frequently queried columns
□ Remove N+1 queries
□ Use eager loading
□ Implement query caching
□ Optimize JOINs
□ Use pagination for large datasets
```

### 3. Cache Implementation

```php
// Cache expensive operations
public function getServerStats(int $serverId): array {
    $cache = Capsule::table('mod_cache')
        ->where('key', "server_stats_$serverId")
        ->where('expires', '>', time())
        ->first();

    if ($cache) {
        return json_decode($cache->value, true);
    }

    // Expensive operation
    $stats = $this->calculateStats($serverId);

    // Cache for 5 minutes
    Capsule::table('mod_cache')->insert([
        'key' => "server_stats_$serverId",
        'value' => json_encode($stats),
        'expires' => time() + 300,
    ]);

    return $stats;
}
```

### 4. Response Time Targets

| Operation | Target |
|-----------|--------|
| API calls | < 200ms |
| Database queries | < 50ms |
| Page loads | < 500ms |
| Batch operations | < 5s |

## Output

Complete performance audit report with:
- Query analysis
- Cache recommendations
- Index suggestions
- Timing before/after
