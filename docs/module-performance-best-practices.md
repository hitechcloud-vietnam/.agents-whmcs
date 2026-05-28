# WHMCS Module Performance Best Practices

**Version:** 8.0 | **Updated:** 2026-05-28

## Overview

This guide covers performance optimization techniques for WHMCS modules, ensuring efficient execution, minimal database load, and optimized API interactions.

---

## Database Optimization

### Use Query Builder

```php
/**
 * Use WHMCS Query Builder for better performance
 */

// Eager loading relationships
$invoices = WHMCS\Database\Capsule::table('tblinvoices')
    ->where('userid', $clientId)
    ->whereIn('status', ['Paid', 'Unpaid'])
    ->orderBy('duedate', 'desc')
    ->limit(10)
    ->get();

// Join for combined queries
$results = WHMCS\Database\Capsule::table('tblinvoices')
    ->join('tblclients', 'tblinvoices.userid', '=', 'tblclients.id')
    ->select('tblinvoices.*', 'tblclients.firstname', 'tblclients.lastname')
    ->where('tblinvoices.status', 'Unpaid')
    ->get();
```

### Use Indexes

```php
/**
 * Create index when adding module tables
 */
$indexSql = "CREATE INDEX idx_module_client_status 
             ON `mod_your_table` (`client_id`, `status`, `created_at`)";

// Composite index for common queries
$compositeIndex = "CREATE INDEX idx_status_date 
                   ON `mod_your_table` (`status`, `created_at`)";
```

### Batch Operations

```php
/**
 * Batch insert for multiple records
 */
function batchInsertRecords($records)
{
    $insertSql = "INSERT INTO `mod_your_table` 
                  (`client_id`, `data`, `created_at`) VALUES ";
    
    $values = [];
    $bindings = [];
    
    foreach ($records as $record) {
        $values[] = '(?, ?, NOW())';
        $bindings[] = $record['client_id'];
        $bindings[] = json_encode($record['data']);
    }
    
    $insertSql .= implode(', ', $values);
    
    WHMCS\Database\Capsule::connection()->prepare($insertSql)->execute($bindings);
}
```

## Caching Strategies

### Result Caching

```php
/**
 * Cache API responses
 */
function getCacheOptions($ttl = 3600)
{
    return [
        'cache_ttl' => $ttl,
        'cache_key' => 'your_module_' . md5($uniqueKey),
    ];
}

function fetchWithCache($apiCall, $cacheKey, $ttl = 3600)
{
    $cache = WHMCS\Module\Cache::getInstance('YourModule');
    
    if ($cached = $cache->retrieve($cacheKey)) {
        return $cached;
    }
    
    $result = $apiCall();
    
    $cache->store($cacheKey, $result, $ttl);
    
    return $result;
}

/**
 * Invalidate cache on update
 */
function onDataUpdate($clientId)
{
    $cache = WHMCS\Module\Cache::getInstance('YourModule');
    
    // Invalidate specific key
    $cache->forget('client_data_' . $clientId);
    
    // Or invalidate pattern
    $cache->forgetByPattern('client_*');
}
```

### Database Query Caching

```php
/**
 * Cache expensive queries
 */
function getCachedClientCount($status = 'active')
{
    $cacheKey = 'client_count_' . $status;
    
    $cached = WHMCS\Application\Support\Helpers\Cache::get($cacheKey);
    if ($cached !== null) {
        return $cached;
    }
    
    $count = WHMCS\Database\Capsule::table('tblclients')
        ->where('status', $status)
        ->count();
    
    // Cache for 5 minutes
    WHMCS\Application\Support\Helpers\Cache::put($cacheKey, $count, 300);
    
    return $count;
}
```

## Lazy Loading

```php
/**
 * Lazy load expensive data
 */
class YourModuleDataService
{
    private $data = null;
    private $clientId;
    
    public function __construct($clientId)
    {
        $this->clientId = $clientId;
    }
    
    public function getData($force = false)
    {
        if ($this->data === null || $force) {
            // Only fetch when actually needed
            $this->data = $this->fetchDataFromDatabase();
        }
        
        return $this->data;
    }
    
    private function fetchDataFromDatabase()
    {
        return WHMCS\Database\Capsule::table('mod_your_table')
            ->where('client_id', $this->clientId)
            ->first();
    }
}
```

## API Call Optimization

### Request Batching

```php
/**
 * Batch multiple API requests
 */
function batchApiRequests($requests, $batchSize = 10)
{
    $results = [];
    $chunks = array_chunk($requests, $batchSize);
    
    foreach ($chunks as $chunk) {
        // Build batch request
        $batchPayload = array_map(function($req) {
            return [
                'id'      => $req['id'],
                'method'  => $req['method'],
                'params'  => $req['params'],
            ];
        }, $chunk);
        
        $response = apiPost('https://api.example.com/batch', $batchPayload);
        
        foreach ($response['results'] as $result) {
            $results[$result['id']] = $result;
        }
    }
    
    return $results;
}
```

### Parallel Requests

```php
/**
 * Execute parallel API requests
 */
function parallelApiCalls($endpoints)
{
    $multiHandle = curl_multi_init();
    $handles = [];
    
    foreach ($endpoints as $id => $endpoint) {
        $handles[$id] = curl_init($endpoint);
        curl_setopt_array($handles[$id], [
            CURLOPT_RETURNTRANSFER => true,
            CURLOPT_TIMEOUT       => 30,
        ]);
        curl_multi_add_handle($multiHandle, $handles[$id]);
    }
    
    // Execute all requests in parallel
    $running = null;
    do {
        curl_multi_exec($multiHandle, $running);
        curl_multi_select($multiHandle);
    } while ($running > 0);
    
    // Collect results
    $results = [];
    foreach ($handles as $id => $handle) {
        $results[$id] = curl_multi_getcontent($handle);
        curl_multi_remove_handle($multiHandle, $handle);
        curl_close($handle);
    }
    
    curl_multi_close($multiHandle);
    
    return $results;
}
```

## Memory Optimization

### Process in Chunks

```php
/**
 * Process large datasets in chunks
 */
function processLargeDataset($clientIds, $batchSize = 100)
{
    $totalProcessed = 0;
    
    foreach (array_chunk($clientIds, $batchSize) as $batch) {
        $records = WHMCS\Database\Capsule::table('mod_your_table')
            ->whereIn('client_id', $batch)
            ->get();
        
        foreach ($records as $record) {
            processRecord($record);
            $totalProcessed++;
        }
        
        // Clear memory between batches
        gc_collect_cycles();
    }
    
    return $totalProcessed;
}
```

### Use Generators

```php
/**
 * Generator for memory-efficient iteration
 */
function iterateLargeDataset()
{
    $query = WHMCS\Database\Capsule::table('mod_your_table')
        ->where('status', 'pending')
        ->orderBy('id');
    
    // Don't use ->get(), iterate instead
    foreach ($query->cursor() as $row) {
        yield $row;
    }
}

/**
 * Process using generator
 */
function processWithGenerator()
{
    foreach (iterateLargeDataset() as $row) {
        // Process one row at a time
        processRow($row);
    }
}
```

## Hook Performance

### Efficient Hook Implementation

```php
/**
 * Register hook with priority control
 */
function your_module_registerHooks()
{
    return [
        // Higher priority = runs first
        'ClientAdd' => [
            'handler'     => 'onClientAdd',
            'priority'    => 100,
        ],
        // Lower priority for less critical operations
        'AfterCronJob' => [
            'handler'     => 'onAfterCronJob',
            'priority'    => 1000,
        ],
    ];
}

/**
 * Keep hooks lightweight
 */
function onClientAdd($vars)
{
    // DON'T: Make heavy API calls in hooks
    // heavyExternalApiCall();
    
    // DO: Queue for later processing
    your_module_queueAction('sync', $vars['userid']);
}
```

### Queue Heavy Operations

```php
/**
 * Queue operations for background processing
 */
function queueModuleAction($action, $data)
{
    WHMCS\Database\Capsule::table('mod_your_queue')->insert([
        'action'    => $action,
        'data'      => json_encode($data),
        'status'    => 'pending',
        'created_at'=> date('Y-m-d H:i:s'),
    ]);
}

/**
 * Process queue via cron
 */
function processQueue($limit = 100)
{
    $pending = WHMCS\Database\Capsule::table('mod_your_queue')
        ->where('status', 'pending')
        ->orderBy('created_at')
        ->limit($limit)
        ->get();
    
    foreach ($pending as $job) {
        try {
            processJob($job);
            WHMCS\Database\Capsule::table('mod_your_queue')
                ->where('id', $job->id)
                ->update(['status' => 'completed']);
        } catch (\Exception $e) {
            WHMCS\Database\Capsule::table('mod_your_queue')
                ->where('id', $job->id)
                ->update([
                    'status'   => 'failed',
                    'error'    => $e->getMessage(),
                    'attempts' => $job->attempts + 1,
                ]);
        }
    }
}
```

## Response Time Targets

| Operation | Target | Maximum |
|-----------|--------|---------|
| Database Query | < 50ms | 200ms |
| API Call | < 200ms | 500ms |
| Page Render | < 100ms | 300ms |
| Hook Execution | < 20ms | 100ms |
| Full Module Load | < 150ms | 500ms |

## Performance Checklist

- [ ] Use query builder instead of raw SQL
- [ ] Add adequate indexes to module tables
- [ ] Cache API responses
- [ ] Implement lazy loading
- [ ] Batch database operations
- [ ] Process large datasets in chunks
- [ ] Use generators for memory efficiency
- [ ] Queue heavy operations
- [ ] Monitor page execution time
- [ ] Profile slow queries

---

## Related Skills and Workflows

- `module-cache-guide` - Caching implementation
- `module-database-patterns` - Database optimization
- `module-cron-reference` - Background processing via cron
- `performance-optimization` - Overall performance tips
- `database-indexing-guide` - Index creation strategy
