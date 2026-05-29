# WHMCS Performance Debugging Workflow

## Overview
This workflow guides you through debugging performance issues in WHMCS modules.

## Prerequisites
- Query logging enabled
- Profiling tools
- Monitoring

## Step-by-Step Guide

### Step 1: Enable Query Logging
```php
// In configuration.php
define('DB_DEBUG', true);
```

### Step 2: Profile Slow Queries
```php
// Add to module code
$start = microtime(true);

// Your code
$results = \WHMCS\Database\Capsule::table('tblclients')
    ->where('status', 'Active')
    ->get();

$duration = microtime(true) - $start;

if ($duration > 1.0) {
    \Log::warning("Slow query detected", [
        'duration' => $duration,
        'query' => 'tblclients query',
    ]);
}
```

### Step 3: Use EXPLAIN
```sql
EXPLAIN SELECT * FROM tblclients WHERE email LIKE '%@example.com';
```

### Step 4: Monitor with Blackfire
```bash
# Install Blackfire agent
curl -sS https://installer.blackfire.io/ | bash

# Profile a request
blackfire curl https://whmcs.com/admin/index.php
```

### Step 5: Common Performance Fixes
```php
// Add indexes
\WHMCS\Database\Capsule::schema()->table('mod_yourmodule', function ($table) {
    $table->index(['client_id', 'created_at']);
});

// Use eager loading
$services = \WHMCS\Module\YourModule\Model::with(['client', 'product'])
    ->where('status', 'Active')
    ->get();

// Cache results
$cacheKey = 'client_stats_' . $clientId;
$stats = \Cache::remember($cacheKey, 3600, function() use ($clientId) {
    return computeStats($clientId);
});
```

## Performance Debugging Checklist

### Profiling
- [ ] Slow queries identified
- [ ] Bottlenecks located
- [ ] Memory usage checked
- [ ] CPU time measured

### Optimization
- [ ] Queries optimized
- [ ] Indexes added
- [ ] Caching implemented
- [ ] Code refactored

### Verification
- [ ] Performance improved
- [ ] No regressions
- [ ] Benchmarks improved
