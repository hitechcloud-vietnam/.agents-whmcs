# WHMCS Module Cache Guide

**Version:** 8.0 | **Updated:** 2026-05-28

## Overview

This guide covers caching strategies for WHMCS modules, including WHMCS cache API usage, custom cache implementations, cache invalidation, and performance optimization through caching.

---

## WHMCS Cache API

### Using WHMCS\Module\Cache

```php
/**
 * Get module cache instance
 * 
 * @param string $moduleName Module name
 * @return WHMCS\Module\Cache
 */
$cache = WHMCS\Module\Cache::getInstance('YourModule');

// Store item
$cache->store('key', $data, $ttl = 3600);

// Retrieve item
$data = $cache->retrieve('key');

// Check if exists
$exists = $cache->exists('key');

// Remove single item
$cache->forget('key');

// Clear all module cache
$cache->clear();
```

### Cache with Default Values

```php
/**
 * Get cached data or fetch and cache
 * 
 * @param string $key Cache key
 * @param callable $fetch Callback to fetch data
 * @param int $ttl Time to live in seconds
 * @return mixed
 */
function getCachedOrFetch($key, $fetch, $ttl = 3600)
{
    $cache = WHMCS\Module\Cache::getInstance('YourModule');
    
    $data = $cache->retrieve($key);
    
    if ($data !== null) {
        return $data;
    }
    
    // Fetch fresh data
    $data = $fetch();
    
    // Cache the result
    $cache->store($key, $data, $ttl);
    
    return $data;
}

// Usage
$result = getCachedOrFetch('client_data_' . $clientId, function() use ($clientId) {
    return WHMCS\Database\Capsule::table('tblclients')
        ->where('id', $clientId)
        ->first();
});
```

## Cache Layer Implementation

### Cache Service Class

```php
<?php
/**
 * Module Cache Service
 */

namespace YourModule\Cache;

class CacheService
{
    private $cachePrefix = 'your_module_';
    private $ttl;
    
    public function __construct($ttl = 3600)
    {
        $this->ttl = $ttl;
    }
    
    /**
     * Get cache instance
     */
    private function cache()
    {
        return WHMCS\Module\Cache::getInstance('YourModule');
    }
    
    /**
     * Build cache key
     */
    public function buildKey($key)
    {
        return $this->cachePrefix . $key;
    }
    
    /**
     * Get client data
     */
    public function getClientData($clientId)
    {
        return $this->cache()->retrieve($this->buildKey("client_{$clientId}"));
    }
    
    /**
     * Store client data
     */
    public function setClientData($clientId, $data, $ttl = null)
    {
        $this->cache()->store(
            $this->buildKey("client_{$clientId}"),
            $data,
            $ttl ?? $this->ttl
        );
    }
    
    /**
     * Clear client cache
     */
    public function clearClientCache($clientId)
    {
        $this->cache()->forget($this->buildKey("client_{$clientId}"));
    }
    
    /**
     * Clear all module cache
     */
    public function clearAll()
    {
        $this->cache()->clear();
    }
}
```

## Cache Patterns

### Lazy Cache

```php
/**
 * Lazy loading cache pattern
 */
class LazyCache
{
    private $data = [];
    private $loaded = false;
    private $loader;
    
    public function __construct(callable $loader, $ttl = 3600)
    {
        $this->loader = $loader;
        $this->ttl = $ttl;
    }
    
    public function get($key)
    {
        if (!isset($this->data[$key])) {
            if (!$this->loaded) {
                $this->loadAll();
                $this->loaded = true;
            }
            
            if (!isset($this->data[$key])) {
                $this->data[$key] = call_user_func($this->loader, $key);
            }
        }
        
        return $this->data[$key];
    }
    
    private function loadAll()
    {
        // Load commonly needed data in bulk
        // This reduces database calls
    }
}
```

### Cache-Aside Pattern

```php
/**
 * Cache-aside pattern
 */
function getData($clientId)
{
    $cacheKey = "client_{$clientId}";
    $cache = WHMCS\Module\Cache::getInstance('YourModule');
    
    // Step 1: Try cache first
    $cached = $cache->retrieve($cacheKey);
    if ($cached !== null) {
        return $cached;
    }
    
    // Step 2: Load from database
    $data = WHMCS\Database\Capsule::table('mod_your_table')
        ->where('client_id', $clientId)
        ->first();
    
    if ($data === null) {
        return null;
    }
    
    // Step 3: Store in cache
    $cache->store($cacheKey, $data, 3600);
    
    return $data;
}
```

### Read-Through Cache

```php
/**
 * Read-through cache
 */
function getClientStats($clientId)
{
    $cache = WHMCS\Module\Cache::getInstance('YourModule');
    $cacheKey = "stats_{$clientId}";
    
    // Check cache
    $cached = $cache->retrieve($cacheKey);
    if ($cached !== null) {
        $cached['from_cache'] = true;
        return $cached;
    }
    
    // Need to calculate
    $stats = calculateClientStats($clientId);
    
    // Store for future requests
    $cache->store($cacheKey, $stats, 3600);
    
    $stats['from_cache'] = false;
    return $stats;
}

function calculateClientStats($clientId)
{
    return [
        'total_orders' => WHMCS\Database\Capsule::table('tblorders')
            ->where('userid', $clientId)
            ->count(),
        'total_spent'  => WHMCS\Database\Capsule::table('tblinvoices')
            ->where('userid', $clientId)
            ->where('status', 'Paid')
            ->sum('total'),
    ];
}
```

## Cache Invalidation

### Automatic Expiration

```php
/**
 * Set cache with expiration
 */
$cache->store('data_key', $data, 3600); // 1 hour
$cache->store('short_lived', $data, 60); // 1 minute
$cache->store('long_lived', $data, 86400); // 1 day
```

### Event-Based Invalidation

```php
/**
 * Invalidate cache on events
 */
function your_module_registerHooks()
{
    return [
        'ClientEdit' => 'invalidateClientCache',
        'ClientDelete' => 'invalidateClientCache',
        'InvoicePaid' => 'invalidateClientCache',
    ];
}

function invalidateClientCache($vars)
{
    $clientId = $vars['userid'] ?? $vars['user_id'] ?? null;
    
    if (!$clientId) {
        return;
    }
    
    $cache = WHMCS\Module\Cache::getInstance('YourModule');
    $cache->forget("client_{$clientId}");
    $cache->forget("stats_{$clientId}");
    $cache->forget("data_{$clientId}");
}
```

### Pattern-Based Invalidation

```php
/**
 * Invalidate by pattern
 */
function your_module_cacheInvalidate($clientId)
{
    $cache = WHMCS\Module\Cache::getInstance('YourModule');
    
    // Invalidate all keys for this client
    $cache->forget("client_{$clientId}");
    $cache->forget("stats_{$clientId}");
    $cache->forget("orders_{$clientId}");
    $cache->forget("invoices_{$clientId}");
}

/**
 * Full cache clear
 */
function your_module_clearAllCache()
{
    $cache = WHMCS\Module\Cache::getInstance('YourModule');
    $cache->clear();
    
    logActivity('Your module: Cache cleared');
}
```

## Database Query Caching

### Caching Expensive Queries

```php
/**
 * Cache database results
 */
function getCachedQuery($queryKey, $callback, $ttl = 300)
{
    $cache = WHMCS\Module\Cache::getInstance('YourModule');
    $cacheKey = "query_{$queryKey}";
    
    $cached = $cache->retrieve($cacheKey);
    if ($cached !== null) {
        return $cached;
    }
    
    $result = $callback();
    
    $cache->store($cacheKey, $result, $ttl);
    
    return $result;
}

/**
 * Count cache
 */
function getCachedCount($status, $ttl = 300)
{
    return getCachedQuery("count_{$status}", function() use ($status) {
        return WHMCS\Database\Capsule::table('mod_your_table')
            ->where('status', $status)
            ->count();
    }, $ttl);
}

/**
 * List cache
 */
function getCachedList($status, $limit, $ttl = 300)
{
    return getCachedQuery("list_{$status}_{$limit}", function() use ($status, $limit) {
        return WHMCS\Database\Capsule::table('mod_your_table')
            ->where('status', $status)
            ->orderBy('created_at', 'desc')
            ->limit($limit)
            ->get()
            ->toArray();
    }, $ttl);
}
```

## API Response Caching

```php
/**
 * Cache API responses
 */
function cacheApiCall($endpoint, $params, $ttl = 300)
{
    $cache = WHMCS\Module\Cache::getInstance('YourModule');
    $cacheKey = 'api_' . md5($endpoint . json_encode($params));
    
    // Check cache
    $cached = $cache->retrieve($cacheKey);
    if ($cached !== null) {
        return array_merge($cached, ['_cached' => true]);
    }
    
    // Make API call
    $response = makeApiCall($endpoint, $params);
    
    // Cache response
    if ($response['success']) {
        $cache->store($cacheKey, $response, $ttl);
    }
    
    return array_merge($response, ['_cached' => false]);
}

/**
 * Refresh API cache
 */
function refreshApiCache($endpoint, $params = [])
{
    $cache = WHMCS\Module\Cache::getInstance('YourModule');
    $cacheKey = 'api_' . md5($endpoint . json_encode($params));
    
    // Clear old cache
    $cache->forget($cacheKey);
    
    // Make fresh API call
    return makeApiCall($endpoint, $params);
}
```

## Session Cache

```php
/**
 * Use session for request-scoped cache
 */
function getSessionCache($key, $default = null)
{
    $cacheKey = 'your_module_' . $key;
    
    if (isset($_SESSION[$cacheKey])) {
        return $_SESSION[$cacheKey];
    }
    
    return $default;
}

function setSessionCache($key, $value)
{
    $_SESSION['your_module_' . $key] = $value;
}

function clearSessionCache()
{
    foreach ($_SESSION as $key => $value) {
        if (strpos($key, 'your_module_') === 0) {
            unset($_SESSION[$key]);
        }
    }
}
```

## Cache Performance Tips

### Cache Guidelines

| Data Type | Recommended TTL | Invalidation |
|-----------|-----------------|---------------|
| User preferences | 1 day | On edit |
| Dashboard stats | 5 minutes | Event-based |
| API responses | 5-15 minutes | Time-based |
| Configuration | 1 hour | On save |
| Expensive queries | 15-30 minutes | Event-based |
| Session data | Request | On logout |

### Cache Monitoring

```php
/**
 * Log cache performance
 */
function logCacheMetrics($operation, $key, $duration, $cached)
{
    logModuleCall('your_module', 'cache', [
        'operation'  => $operation,
        'key'        => $key,
        'duration_ms'=> $duration * 1000,
        'from_cache' => $cached,
    ]);
}
```

---

## Related Skills and Workflows

- `module-performance-best-practices` - Performance with caching
- `module-database-patterns` - Database caching
- `caching-strategies` - WHMCS caching overview
- `performance-optimization` - Optimization techniques
