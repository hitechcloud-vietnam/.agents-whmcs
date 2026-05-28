# WHMCS Caching Strategies

**Version:** 8.x
**Updated:** 2026-05-28
**Related Skills:** `performance-optimization`, `database-indexing-guide`

## Overview

Effective caching is essential for WHMCS performance. This guide covers the built-in caching system, Redis/Memcached implementation, and custom caching strategies.

## WHMCS Cache System

### Available Cache Stores

```php
// Available cache adapters
$cacheStores = [
    'file'     => 'File-based caching (default)',
    'redis'    => 'Redis (recommended for production)',
    'memcached' => 'Memcached',
    'apc'      => 'APC (deprecated)',
    'apcu'     => 'APCu',
    'wincache' => 'WinCache (Windows only)',
];
```

### Basic Cache Usage

```php
// Get cache instance
$cache = WHMCS\Cache::store('redis');

// Or use file store
$cache = WHMCS\Cache::store('file');

// Basic operations
$cache->set('key', 'value', $ttl = 3600);  // Set with TTL
$cache->get('key');                          // Get value
$cache->has('key');                          // Check exists
$cache->forget('key');                       // Delete key
$cache->flush();                             // Clear all cache
```

### Complex Data Caching

```php
// Cache complex data structures
$data = [
    'users' => $users,
    'stats' => $stats,
    'timestamp' => time(),
];

$cache->set('dashboard_data', $data, 300); // 5 minute TTL

// Retrieve and check freshness
$cached = $cache->get('dashboard_data');

if ($cached && $cached['timestamp'] > time() - 300) {
    // Use cached data
    $data = $cached;
} else {
    // Refresh data
    $data = fetchFreshData();
    $cache->set('dashboard_data', $data, 300);
}
```

## Redis Configuration

### Redis Setup

```php
// configuration.php

// Redis connection settings
$redis_config = [
    'host' => '127.0.0.1',
    'port' => 6379,
    'database' => 0,
    'password' => null,
    'timeout' => 2.5,
    'read_timeout' => 2.5,
    'persistent' => false,
];
```

### Redis Configuration File

```php
// storage/config/redis.php

return [
    'default' => [
        'host' => env('REDIS_HOST', '127.0.0.1'),
        'port' => env('REDIS_PORT', 6379),
        'password' => env('REDIS_PASSWORD', null),
        'database' => env('REDIS_DB', 0),
    ],

    'cache' => [
        'host' => env('REDIS_HOST', '127.0.0.1'),
        'port' => env('REDIS_PORT', 6379),
        'database' => env('REDIS_CACHE_DB', 1),
        'prefix' => 'whmcs_cache_',
    ],

    'session' => [
        'host' => env('REDIS_HOST', '127.0.0.1'),
        'port' => env('REDIS_PORT', 6379),
        'database' => env('REDIS_SESSION_DB', 2),
    ],

    'locks' => [
        'host' => env('REDIS_HOST', '127.0.0.1'),
        'port' => env('REDIS_PORT', 6379),
        'database' => env('REDIS_LOCKS_DB', 3),
    ],
];
```

### Redis Security

```bash
# Redis configuration
# /etc/redis/redis.conf

# Bind to localhost only
bind 127.0.0.1

# Require password
requirepass your-redis-password

# Disable dangerous commands
rename-command FLUSHALL ""
rename-command FLUSHDB ""
rename-command CONFIG ""

# Max memory
maxmemory 256mb
maxmemory-policy allkeys-lru
```

## Custom Cache Implementation

### Query Result Caching

```php
// includes/hooks/cache_queries.php

/**
 * Cache expensive database queries
 */
class QueryCache
{
    protected $cache;
    protected $prefix = 'query_';

    public function __construct()
    {
        $this->cache = WHMCS\Cache::store('redis');
    }

    /**
     * Cache query results
     */
    public function remember(string $key, callable $callback, int $ttl = 3600)
    {
        $cacheKey = $this->prefix . $key;

        $result = $this->cache->get($cacheKey);

        if ($result !== null) {
            return $result;
        }

        $result = $callback();
        $this->cache->set($cacheKey, $result, $ttl);

        return $result;
    }

    /**
     * Invalidate query cache
     */
    public function invalidate(string $key): void
    {
        $this->cache->forget($this->prefix . $key);
    }

    /**
     * Cache client data
     */
    public function getClientData(int $clientId): array
    {
        return $this->remember("client_{$clientId}", function() use ($clientId) {
            return Capsule::table('tblclients')
                ->where('id', $clientId)
                ->first();
        }, 1800); // 30 minute TTL
    }

    /**
     * Cache service list
     */
    public function getClientServices(int $clientId): array
    {
        return $this->remember("client_{$clientId}_services", function() use ($clientId) {
            return Capsule::table('tblhosting')
                ->join('tblproducts', 'tblhosting.packageid', '=', 'tblproducts.id')
                ->where('tblhosting.userid', $clientId)
                ->where('tblhosting.domainstatus', 'Active')
                ->get();
        }, 900); // 15 minute TTL
    }

    /**
     * Cache system stats
     */
    public function getSystemStats(): array
    {
        return $this->remember('system_stats', function() {
            return [
                'total_clients' => Capsule::table('tblclients')->count(),
                'active_services' => Capsule::table('tblhosting')
                    ->where('domainstatus', 'Active')->count(),
                'monthly_revenue' => Capsule::table('tblinvoices')
                    ->where('status', 'Paid')
                    ->where('date', '>=', date('Y-m-01'))
                    ->sum('total'),
                'open_tickets' => Capsule::table('tbltickets')
                    ->whereIn('status', ['Open', 'Awaiting Reply'])->count(),
            ];
        }, 300); // 5 minute TTL
    }
}

// Initialize globally
$queryCache = new QueryCache();
```

### Template Fragment Caching

```php
// includes/hooks/template_cache.php

/**
 * Smarty cache modifier plugin
 */
function smarty_modifier_cache($content, $key, $ttl = 3600)
{
    $cache = WHMCS\Cache::store('redis');
    $cacheKey = 'template_' . md5($key);

    $cached = $cache->get($cacheKey);

    if ($cached !== null) {
        return $cached;
    }

    $cache->set($cacheKey, $content, $ttl);
    return $content;
}

// Usage in templates:
// {$expensive_content|cache:'unique_key':3600}
```

### API Response Caching

```php
// includes/hooks/api_cache.php

class ApiCache
{
    protected $cache;
    protected $prefix = 'api_';

    public function __construct()
    {
        $this->cache = WHMCS\Cache::store('redis');
    }

    /**
     * Cache external API responses
     */
    public function cacheApiCall(string $endpoint, array $params, $response, int $ttl = 300): void
    {
        $key = $this->buildKey($endpoint, $params);
        $this->cache->set($this->prefix . $key, $response, $ttl);
    }

    /**
     * Get cached API response
     */
    public function getCachedApiCall(string $endpoint, array $params)
    {
        $key = $this->buildKey($endpoint, $params);
        return $this->cache->get($this->prefix . $key);
    }

    /**
     * Build cache key from endpoint and params
     */
    protected function buildKey(string $endpoint, array $params): string
    {
        return md5($endpoint . serialize($params));
    }
}

// Usage in server module
class YourModule
{
    protected $apiCache;

    public function __construct()
    {
        $this->apiCache = new ApiCache();
    }

    public function getUsageStats(array $params): array
    {
        $cacheKey = "stats_{$params['serviceid']}_{$params['period']}";

        // Try cache first
        $cached = $this->apiCache->getCachedApiCall('GetUsage', [
            'service' => $params['serviceid'],
            'period' => $params['period'],
        ]);

        if ($cached !== null) {
            return $cached;
        }

        // Fetch from API
        $result = $this->api->getUsage($params);

        // Cache for 5 minutes
        $this->apiCache->cacheApiCall('GetUsage', $params, $result, 300);

        return $result;
    }
}
```

## Page Caching

### Full Page Caching

```php
// includes/hooks/page_cache.php

add_hook('PreTemplateRender', 1, function($vars) {
    // Only cache for non-logged-in users
    if (!empty($_SESSION['uid'])) {
        return;
    }

    // Don't cache admin area
    if (defined('ADMINAREA')) {
        return;
    }

    $cache = WHMCS\Cache::store('redis');
    $cacheKey = 'page_' . md5($_SERVER['REQUEST_URI']);

    // Check if page is in cache
    $cached = $cache->get($cacheKey);

    if ($cached !== null) {
        // Output cached page and exit
        echo $cached['html'];
        echo '<!-- Cached at: ' . $cached['time'] . ' -->';
        exit;
    }
});

add_hook('AfterApplicationOutput', 1, function($vars) {
    // Skip for logged-in users
    if (!empty($_SESSION['uid']) || defined('ADMINAREA')) {
        return;
    }

    $cache = WHMCS\Cache::store('redis');
    $cacheKey = 'page_' . md5($_SERVER['REQUEST_URI']);

    // Cache page for 5 minutes
    $cache->set($cacheKey, [
        'html' => $vars['output'],
        'time' => date('Y-m-d H:i:s'),
    ], 300);
});
```

### Fragment Caching in Templates

```smarty
{* Cache expensive template fragments *}
{cache key="featured-products" ttl=3600}
    <div class="featured-products">
        {foreach $featuredProducts as $product}
            <div class="product-card">
                <img src="{$product.image}" alt="{$product.name}">
                <h3>{$product.name}</h3>
                <p>{$product.description}</p>
            </div>
        {/foreach}
    </div>
{/cache}

{* Cache based on user context *}
{cache key="user-nav-{$loggedin}" ttl=1800}
    {if $loggedin}
        <div class="user-nav">
            <a href="{$WEB_ROOT}/clientarea.php">Dashboard</a>
            <a href="{$WEB_ROOT}/clientarea.php?action=account">Account</a>
        </div>
    {else}
        <div class="guest-nav">
            <a href="{$WEB_ROOT}/login.php">Login</a>
            <a href="{$WEB_ROOT}/register.php">Register</a>
        </div>
    {/if}
{/cache}
```

## Cache Invalidation

### Event-Based Invalidation

```php
// includes/hooks/cache_invalidation.php

// Invalidate client cache on update
add_hook('ClientEdit', 1, function($vars) {
    $cache = WHMCS\Cache::store('redis');
    $cache->forget('query_client_' . $vars['userid']);
    $cache->forget('query_client_' . $vars['userid'] . '_services');
});

// Invalidate on new order
add_hook('OrderPaid', 1, function($vars) {
    $cache = WHMCS\Cache::store('redis');
    $cache->forget('system_stats');

    // Get client ID from order
    $order = Capsule::table('tblorders')
        ->where('id', $vars['order_id'])
        ->first();

    if ($order) {
        $cache->forget('query_client_' . $order->userid . '_services');
    }
});

// Invalidate on payment
add_hook('InvoicePaid', 1, function($vars) {
    $cache = WHMCS\Cache::store('redis');
    $cache->forget('system_stats');

    $invoice = Capsule::table('tblinvoices')
        ->where('id', $vars['invoice_id'])
        ->first();

    if ($invoice) {
        $cache->forget('query_client_' . $invoice->userid);
    }
});

// Invalidate on service change
add_hook('ServiceProvision', 1, function($vars) {
    $cache = WHMCS\Cache::store('redis');

    $service = Capsule::table('tblhosting')
        ->where('id', $vars['serviceid'])
        ->first();

    if ($service) {
        $cache->forget('query_client_' . $service->userid . '_services');
    }
});
```

### Scheduled Cache Clearing

```php
// includes/hooks/scheduled_cache_clear.php

// Clear stale cache entries daily
add_hook('DailyCronJob', 1, function($vars) {
    $cache = WHMCS\Cache::store('redis');

    // Clear old page caches
    // Redis will handle TTL automatically

    // Clear expired query caches
    // Add manual cleanup if using file cache

    logActivity('Daily cache cleanup completed');
});

// Clear all cache on demand
function clearAllCache(): void
{
    // Clear WHMCS cache
    WHMCS\Cache::flush();

    // Clear file cache
    $cacheDir = ROOTDIR . '/storage/cache';
    if (is_dir($cacheDir)) {
        $files = glob($cacheDir . '/*');
        foreach ($files as $file) {
            if (is_file($file)) {
                unlink($file);
            }
        }
    }

    // Clear templates_c
    $templatesDir = ROOTDIR . '/templates_c';
    if (is_dir($templatesDir)) {
        $files = glob($templatesDir . '/*');
        foreach ($files as $file) {
            if (is_file($file)) {
                unlink($file);
            }
        }
    }

    logActivity('All cache cleared manually');
}
```

## Cache Monitoring

### Cache Statistics

```php
// includes/hooks/cache_stats.php

/**
 * Get cache statistics
 */
function getCacheStats(): array
{
    $cache = WHMCS\Cache::store('redis');

    // Get Redis info
    $redis = $cache->getRedis();
    $info = $redis->info();

    return [
        'used_memory' => $info['used_memory_human'],
        'connected_clients' => $info['connected_clients'],
        'total_keys' => $redis->dbSize(),
        'hit_rate' => calculateHitRate($info),
        'uptime_days' => round($info['uptime_in_days'], 1),
    ];
}

/**
 * Calculate cache hit rate
 */
function calculateHitRate(array $info): float
{
    $keyspaceHits = $info['keyspace_hits'] ?? 0;
    $keyspaceMisses = $info['keyspace_misses'] ?? 0;
    $total = $keyspaceHits + $keyspaceMisses;

    if ($total === 0) {
        return 0;
    }

    return round(($keyspaceHits / $total) * 100, 2);
}
```

## Best Practices

1. **Use Redis for Production**: Better performance than file cache
2. **Set Appropriate TTLs**: Balance freshness vs performance
3. **Invalidate on Updates**: Clear related cache when data changes
4. **Monitor Cache Size**: Prevent memory issues
5. **Use Cache Keys Wisely**: Include relevant identifiers
6. **Don't Over-Cache**: Cache expensive operations, not everything
7. **Handle Cache Misses**: Always have fallback to fresh data

## Related Documentation

- [Performance Optimization](performance-optimization.md)
- [Database Indexing Guide](database-indexing-guide.md)
- [Cron Events Reference](cron-events-reference.md)
