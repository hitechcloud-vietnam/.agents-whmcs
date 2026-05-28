# WHMCS Cache Optimization Workflow

## Overview
This workflow provides strategies and implementation guidelines for optimizing caching in WHMCS to improve performance, reduce database load, and enhance user experience.

## Prerequisites
- WHMCS installation (v8.0+)
- PHP 8.0+ environment
- Redis, Memcached, or file-based caching configured
- SSH access to server
- Admin access to WHMCS

---

## Step 1: Cache Architecture Assessment

### 1.1 Evaluate Current Cache Configuration
```bash
# Check current cache settings in WHMCS configuration
cat /var/www/html/whmcs/configuration.php | grep -i cache

# Verify Redis/Memcached connection
redis-cli ping
memcached-tool localhost:11211 stats
```

### 1.2 Analyze Cache Hit Rates
```sql
-- Create cache analysis query
SELECT 
    cache_type,
    COUNT(*) as hit_count,
    AVG(TIMESTAMPDIFF(SECOND, created_at, last_accessed)) as avg_age,
    MAX(TIMESTAMPDIFF(SECOND, created_at, last_accessed)) as max_age
FROM tbl_cache_stats
WHERE created_at > DATE_SUB(NOW(), INTERVAL 7 DAY)
GROUP BY cache_type;
```

### 1.3 Current Cache Types to Review
- **Opcode Cache**: PHP compiled scripts
- **Object Cache**: WHMCS data objects (Redis/Memcached)
- **Database Query Cache**: Query results
- **Template Cache**: Smarty compiled templates
- **Asset Cache**: CSS, JS, images
- **API Response Cache**: Third-party API responses

---

## Step 2: Implement Multi-Layer Caching Strategy

### 2.1 Configure Redis Object Cache

Create `/var/www/html/whmcs/includes/cache/RedisCache.php`:
```php
<?php
/**
 * Redis Cache Driver for WHMCS
 */
namespace WHMCS\Cache;

class RedisCacheDriver implements CacheDriverInterface
{
    private $redis;
    private $prefix;
    private $ttl;

    public function __construct(array $config)
    {
        $this->redis = new \Redis();
        $this->redis->connect(
            $config['host'] ?? '127.0.0.1',
            $config['port'] ?? 6379
        );
        $this->prefix = $config['prefix'] ?? 'whmcs:';
        $this->ttl = $config['ttl'] ?? 3600;
    }

    public function get(string $key, $default = null)
    {
        $value = $this->redis->get($this->prefix . $key);
        return $value !== false ? unserialize($value) : $default;
    }

    public function set(string $key, $value, ?int $ttl = null): bool
    {
        $ttl = $ttl ?? $this->ttl;
        return $this->redis->setex(
            $this->prefix . $key,
            $ttl,
            serialize($value)
        );
    }

    public function delete(string $key): bool
    {
        return $this->redis->del($this->prefix . $key) > 0;
    }

    public function flush(): bool
    {
        $keys = $this->redis->keys($this->prefix . '*');
        if (!empty($keys)) {
            return $this->redis->del($keys) > 0;
        }
        return true;
    }

    public function remember(string $key, int $ttl, callable $callback)
    {
        $value = $this->get($key);
        if ($value === null) {
            $value = $callback();
            $this->set($key, $value, $ttl);
        }
        return $value;
    }
}
```

### 2.2 Configure WHMCS Cache Settings

Update `configuration.php`:
```php
$redis_cache = [
    'host' => '127.0.0.1',
    'port' => 6379,
    'password' => null,
    'database' => 0,
    'prefix' => 'whmcs_cache_',
    'ttl' => 3600
];

// Enable all cache drivers
$cc_cache = [
    'default' => 'redis',
    'drivers' => [
        'redis' => $redis_cache,
        'apcu' => ['prefix' => 'whmcs_apcu_'],
        'file' => ['path' => '/var/www/html/whmcs/data/cache/']
    ]
];
```

### 2.3 Implement Application-Level Caching

Create `/var/www/html/whmcs/includes/helpers/CacheManager.php`:
```php
<?php
namespace WHMCS\Helpers;

class CacheManager
{
    /**
     * Cache frequently accessed client data
     */
    public static function cacheClientData(int $clientId): array
    {
        $cache = \DI::make('cache');
        $cacheKey = "client_data_{$clientId}";

        return $cache->remember($cacheKey, 1800, function() use ($clientId) {
            $client = \WHMCS\User\Client::find($clientId);
            return [
                'id' => $client->id,
                'fullname' => $client->fullname,
                'email' => $client->email,
                'groupid' => $client->groupid,
                'status' => $client->status,
                'credit' => $client->credit,
                'currency' => $client->currency
            ];
        });
    }

    /**
     * Cache product/pricing data
     */
    public static function cacheProductPricing(int $productId): array
    {
        $cache = \DI::make('cache');
        $cacheKey = "product_pricing_{$productId}";

        return $cache->remember($cacheKey, 3600, function() use ($productId) {
            $product = \WHMCS\Product\Product::find($productId);
            return [
                'id' => $product->id,
                'name' => $product->name,
                'pricing' => $product->pricing()->get(),
                'configoptions' => $product->configOptions()->get()
            ];
        });
    }

    /**
     * Cache ticket counts for dashboard
     */
    public static function cacheTicketCounts(int $clientId): array
    {
        $cache = \DI::make('cache');
        $cacheKey = "ticket_counts_{$clientId}";

        return $cache->remember($cacheKey, 300, function() use ($clientId) {
            return [
                'open' => \WHMCS\Support\Ticket::where('userid', $clientId)
                    ->whereIn('status', ['Open', 'Answered'])->count(),
                'awaiting' => \WHMCS\Support\Ticket::where('userid', $clientId)
                    ->where('status', 'Awaiting Response')->count()
            ];
        });
    }

    /**
     * Invalidate all caches for a client
     */
    public static function invalidateClientCache(int $clientId): void
    {
        $cache = \DI::make('cache');
        $cache->delete("client_data_{$clientId}");
        $cache->delete("ticket_counts_{$clientId}");
    }

    /**
     * Warm cache for active clients
     */
    public static function warmActiveClientCache(int $limit = 100): void
    {
        $activeClients = \WHMCS\User\Client::where('status', 'Active')
            ->orderBy('lastlogin', 'desc')
            ->limit($limit)
            ->pluck('id');

        foreach ($activeClients as $clientId) {
            self::cacheClientData($clientId);
            self::cacheTicketCounts($clientId);
        }
    }
}
```

---

## Step 3: Implement Template Caching

### 3.1 Configure Smarty Caching
```php
// In configuration.php or hooks
add_hook('AdminAreaPageContext', 1, function($vars) {
    // Enable template caching for admin
    \WHMCS\View\Smarty\TemplateRenderer::setCaching(true);
    \WHMCS\View\Smarty\TemplateRenderer::setCacheLifetime(3600);
});

add_hook('ClientAreaPageContext', 1, function($vars) {
    // Configure client area template cache
    \WHMCS\View\Smarty\TemplateRenderer::setCaching(true);
    \WHMCS\View\Smarty\TemplateRenderer::setCacheLifetime(1800);
});
```

### 3.2 Create Hook for Dynamic Cache Invalidation
```php
<?php
// /var/www/html/whmcs/includes/hooks/cache_invalidation.php
use WHMCS\Service\Service;

add_hook('ServiceCreate', 1, function($vars) {
    $cache = \DI::make('cache');
    $cache->delete("client_data_{$vars['userid']}");
    $cache->delete("client_services_{$vars['userid']}");
});

add_hook('ServiceUpdate', 1, function($vars) {
    $service = Service::find($vars['serviceid']);
    if ($service) {
        $cache = \DI::make('cache');
        $cache->delete("client_data_{$service->userid}");
        $cache->delete("service_details_{$vars['serviceid']}");
    }
});

add_hook('InvoicePaid', 1, function($vars) {
    $invoice = \WHMCS\Billing\Invoice::find($vars['invoiceid']);
    if ($invoice) {
        $cache = \DI::make('cache');
        $cache->delete("client_data_{$invoice->userid}");
        $cache->delete("client_invoices_{$invoice->userid}");
    }
});

add_hook('TicketOpen', 1, function($vars) {
    $cache = \DI::make('cache');
    $cache->delete("ticket_counts_{$vars['userid']}");
});

add_hook('TicketReply', 1, function($vars) {
    $ticket = \WHMCS\Support\Ticket::find($vars['ticketid']);
    if ($ticket) {
        $cache = \DI::make('cache');
        $cache->delete("ticket_counts_{$ticket->userid}");
    }
});
```

---

## Step 4: Implement API Response Caching

### 4.1 Create API Response Cache Service
```php
<?php
// /var/www/html/whmcs/includes/APIResponseCache.php
namespace WHMCS\API;

class ResponseCache
{
    private static $instance = null;
    private $cache;
    private $enabled = true;

    private function __construct()
    {
        $this->cache = \DI::make('cache');
    }

    public static function getInstance(): self
    {
        if (self::$instance === null) {
            self::$instance = new self();
        }
        return self::$instance;
    }

    public function getCachedResponse(string $endpoint, array $params): ?array
    {
        if (!$this->enabled) {
            return null;
        }

        $cacheKey = $this->generateCacheKey($endpoint, $params);
        return $this->cache->get($cacheKey);
    }

    public function setCachedResponse(string $endpoint, array $params, array $response, int $ttl = 300): void
    {
        if (!$this->enabled) {
            return;
        }

        $cacheKey = $this->generateCacheKey($endpoint, $params);
        $this->cache->set($cacheKey, $response, $ttl);
    }

    private function generateCacheKey(string $endpoint, array $params): string
    {
        ksort($params);
        $hash = md5($endpoint . serialize($params));
        return "api_response:{$endpoint}:{$hash}";
    }

    public function invalidateEndpoint(string $endpoint): void
    {
        // This would require pattern matching in Redis
        // For now, use versioned cache keys
    }

    public function disable(): void
    {
        $this->enabled = false;
    }

    public function enable(): void
    {
        $this->enabled = true;
    }
}
```

### 4.2 Implement Cached API Calls
```php
<?php
// /var/www/html/whmcs/includes/hooks/api_cache_hook.php
use WHMCS\API\ResponseCache;

add_hook('ApiStart', 1, function($vars) {
    // Monitor API response times
    define('API_START_TIME', microtime(true));
});

add_hook('ApiAfterCall', 1, function($vars) {
    $endpoint = $vars['action'] ?? 'unknown';
    $responseTime = microtime(true) - API_START_TIME;

    // Cache GET requests with standard response times
    $cacheableEndpoints = ['GetClients', 'GetProducts', 'GetConfigurations'];

    if (in_array($endpoint, $cacheableEndpoints) && $responseTime < 0.5) {
        $responseCache = ResponseCache::getInstance();

        // Cache successful responses
        if ($vars['response']['result'] === 'success') {
            $responseCache->setCachedResponse(
                $endpoint,
                $vars['params'] ?? [],
                $vars['response'],
                300 // 5 minute cache
            );
        }
    }
});
```

---

## Step 5: Implement Database Query Caching

### 5.1 Create Query Cache Layer
```php
<?php
// /var/www/html/whmcs/includes/DatabaseQueryCache.php
namespace WHMCS\Database;

class QueryCache
{
    private $cache;
    private static $instance = null;

    public function __construct()
    {
        $this->cache = \DI::make('cache');
    }

    public static function getInstance(): self
    {
        if (self::$instance === null) {
            self::$instance = new self();
        }
        return self::$instance;
    }

    /**
     * Cache expensive aggregation queries
     */
    public function getMonthlyRevenue(int $year = null): array
    {
        $year = $year ?? date('Y');
        $cacheKey = "monthly_revenue_{$year}";

        return $this->cache->remember($cacheKey, 3600, function() use ($year) {
            $results = [];

            for ($month = 1; $month <= 12; $month++) {
                $revenue = \WHMCS\Database\Capsule::table('invoices')
                    ->where('status', 'Paid')
                    ->whereYear('datepaid', $year)
                    ->whereMonth('datepaid', $month)
                    ->sum('total');

                $results[$month] = (float) $revenue;
            }

            return $results;
        });
    }

    /**
     * Cache client count statistics
     */
    public function getClientStatistics(): array
    {
        $cacheKey = "client_statistics";

        return $this->cache->remember($cacheKey, 1800, function() {
            return [
                'total' => \WHMCS\User\Client::count(),
                'active' => \WHMCS\User\Client::where('status', 'Active')->count(),
                'inactive' => \WHMCS\User\Client::where('status', 'Inactive')->count(),
                'suspended' => \WHMCS\User\Client::where('status', 'Suspended')->count(),
                'closed' => \WHMCS\User\Client::where('status', 'Closed')->count(),
                'new_this_month' => \WHMCS\User\Client::whereDate('created_at', '>=', date('Y-m-01'))->count()
            ];
        });
    }

    /**
     * Cache service statistics
     */
    public function getServiceStatistics(): array
    {
        $cacheKey = "service_statistics";

        return $this->cache->remember($cacheKey, 1800, function() {
            return [
                'total' => \WHMCS\Service\Service::count(),
                'active' => \WHMCS\Service\Service::where('domainstatus', 'Active')->count(),
                'pending' => \WHMCS\Service\Service::where('domainstatus', 'Pending')->count(),
                'suspended' => \WHMCS\Service\Service::where('domainstatus', 'Suspended')->count(),
                'terminated' => \WHMCS\Service\Service::where('domainstatus', 'Terminated')->count()
            ];
        });
    }

    /**
     * Invalidate statistics cache
     */
    public function invalidateStatistics(): void
    {
        $this->cache->delete("client_statistics");
        $this->cache->delete("service_statistics");
    }
}
```

### 5.2 Register Hooks for Cache Invalidation
```php
<?php
// /var/www/html/whmcs/includes/hooks/stats_cache_hook.php
add_hook('ClientAdd', 1, function($vars) {
    \WHMCS\Database\QueryCache::getInstance()->invalidateStatistics();
});

add_hook('ServiceCreate', 1, function($vars) {
    \WHMCS\Database\QueryCache::getInstance()->invalidateStatistics();
});

add_hook('InvoicePaid', 1, function($vars) {
    // Invalidate monthly revenue cache
    $invoice = \WHMCS\Billing\Invoice::find($vars['invoiceid']);
    if ($invoice) {
        $year = date('Y', strtotime($invoice->datepaid));
        $cache = \DI::make('cache');
        $cache->delete("monthly_revenue_{$year}");
    }
});
```

---

## Step 6: Implement Opcode Caching

### 6.1 Configure PHP-FPM with OPcache
```ini
; /etc/php/8.0/fpm/php.ini or php.ini

; OPcache settings
opcache.enable=1
opcache.enable_cli=0
opcache.memory_consumption=256
opcache.interned_strings_buffer=16
opcache.max_accelerated_files=10000
opcache.revalidate_freq=0
opcache.fast_shutdown=1
opcache.enable_file_override=0

; JIT settings (PHP 8.0+)
opcache.jit_buffer_size=100M
opcache.jit=tracing
```

### 6.2 Create OPcache Management Script
```php
<?php
// /var/www/html/whmcs/includes/cli/clear_opcache.php
#!/usr/bin/env php
<?php
/**
 * CLI script to clear OPcache
 */

require_once __DIR__ . '/../../init.php';

if (!function_exists('opcache_get_status')) {
    echo "OPcache not available\n";
    exit(1);
}

// Get current status
$status = opcache_get_status(false);
echo "OPcache Status:\n";
echo "Memory: " . number_format($status['memory_usage']['used_memory'] / 1024 / 1024, 2) . " MB\n";
echo "Files: " . $status['opcache_statistics']['num_cached_scripts'] . "\n";
echo "Hits: " . number_format($status['opcache_statistics']['opcache_hit_rate'], 2) . "%\n\n";

// Clear cache if requested
if (isset($argv[1]) && $argv[1] === '--clear') {
    if (function_exists('opcache_reset')) {
        opcache_reset();
        echo "OPcache cleared successfully\n";
    }
}
```

---

## Step 7: Configure CDN for Static Assets

### 7.1 Create Asset Cache Helper
```php
<?php
// /var/www/html/whmcs/includes/helpers/AssetCache.php
namespace WHMCS\Helpers;

class AssetCache
{
    private static $cdnBaseUrl;

    public static function configure(string $cdnUrl): void
    {
        self::$cdnBaseUrl = rtrim($cdnUrl, '/');
    }

    public static function getAssetUrl(string $path): string
    {
        if (empty(self::$cdnBaseUrl)) {
            return $path;
        }

        $version = filemtime(__DIR__ . '/../../' . ltrim($path, '/'));
        $version = $version ?: time();

        return self::$cdnBaseUrl . $path . '?v=' . $version;
    }

    public static function css(string $file): string
    {
        return self::getAssetUrl('/assets/css/' . $file);
    }

    public static function js(string $file): string
    {
        return self::getAssetUrl('/assets/js/' . $file);
    }

    public static function image(string $file): string
    {
        return self::getAssetUrl('/assets/images/' . $file);
    }
}
```

### 7.2 Create Cache Headers Hook
```php
<?php
// /var/www/html/whmcs/includes/hooks/asset_cache_headers.php
add_hook('ClientAreaHeadOutput', 1, function($vars) {
    // Add cache headers for static assets
    // This should be handled by web server config
    return <<<HTML
<script>
    // Cache version for assets
    window.ASSET_CACHE_VERSION = '1.0.0';
</script>
HTML;
});
```

---

## Step 8: Cache Monitoring and Tuning

### 8.1 Create Cache Performance Dashboard
```php
<?php
// /var/www/html/whmcs/modules/widgets/CachePerformance.php
namespace WHMCS\Module\Widget;

class CachePerformance extends \WHMCS\Module\AbstractWidget
{
    protected $title = 'Cache Performance';
    protected $description = 'Monitor cache hit rates and performance';
    protected $author = 'System';
    protected $type = 'dashboard';

    public function getData(): array
    {
        $redis = new \Redis();
        $redis->connect('127.0.0.1', 6379);

        $info = $redis->info('stats');
        $keyCount = $redis->dbSize();

        $hits = $info['keyspace_hits'] ?? 0;
        $misses = $info['keyspace_misses'] ?? 1;
        $hitRate = ($hits / ($hits + $misses)) * 100;

        return [
            'hit_rate' => round($hitRate, 2),
            'total_keys' => $keyCount,
            'memory_used' => $info['used_memory_human'] ?? 'N/A',
            'connected_clients' => $info['connected_clients'] ?? 0,
            'total_commands' => $info['total_commands_processed'] ?? 0
        ];
    }

    public function generateOutput($data): string
    {
        return <<<HTML
<div class="widget-content">
    <div class="stat-row">
        <div class="stat-box">
            <div class="stat-value">{$data['hit_rate']}%</div>
            <div class="stat-label">Cache Hit Rate</div>
        </div>
        <div class="stat-box">
            <div class="stat-value">{$data['total_keys']}</div>
            <div class="stat-label">Cached Items</div>
        </div>
        <div class="stat-box">
            <div class="stat-value">{$data['memory_used']}</div>
            <div class="stat-label">Memory Usage</div>
        </div>
    </div>
</div>
HTML;
    }
}
```

### 8.2 Create Cache Warm-up Cron
```php
<?php
// /var/www/html/whmcs/includes/hooks/cache_warmup.php
// Add to WHMCS cron configuration
add_hook('DailyCronJob', 1, function() {
    $startTime = microtime(true);

    logActivity('Starting cache warm-up');

    // Warm client data cache for active clients
    $activeClients = \WHMCS\User\Client::where('status', 'Active')
        ->orderBy('lastlogin', 'desc')
        ->limit(500)
        ->pluck('id');

    $warmed = 0;
    foreach ($activeClients as $clientId) {
        \WHMCS\Helpers\CacheManager::cacheClientData($clientId);
        \WHMCS\Helpers\CacheManager::cacheTicketCounts($clientId);
        $warmed++;
    }

    // Warm product cache
    $products = \WHMCS\Product\Product::where('hidden', 0)->pluck('id');
    foreach ($products as $productId) {
        \WHMCS\Helpers\CacheManager::cacheProductPricing($productId);
    }

    $duration = round(microtime(true) - $startTime, 2);
    logActivity("Cache warm-up completed: {$warmed} clients warmed in {$duration}s");
});
```

---

## Best Practices

### Cache Key Naming Convention
```
[prefix]:[entity]:[id]:[variant]
Examples:
- whmcs:client:123:summary
- whmcs:product:456:pricing
- whmcs:invoice:789:details
```

### TTL Guidelines
| Cache Type | TTL | Reason |
|------------|-----|--------|
| Client Data | 30 min | Balance freshness vs performance |
| Product Pricing | 1 hour | Pricing doesn't change frequently |
| Statistics | 30 min | Aggregated data |
| API Responses | 5 min | Real-time requirements |
| Templates | 1 hour | Only on template changes |

### Cache Invalidation Strategy
1. **Event-based**: Hook into WHMCS events to invalidate related caches
2. **Time-based**: Use TTL for automatic expiration
3. **Manual**: Provide admin tools for cache clearing
4. **Version-based**: Include version in cache keys for controlled invalidation

---

## Verification Checklist

### Pre-Implementation
- [ ] Redis/Memcached server installed and configured
- [ ] PHP Redis extension enabled
- [ ] Network connectivity to cache server verified
- [ ] Backup of current configuration taken

### Implementation
- [ ] Cache driver implemented and registered
- [ ] Cache keys following naming convention
- [ ] TTL values configured appropriately
- [ ] Cache invalidation hooks registered

### Post-Implementation
- [ ] Cache hit rate > 80% for frequently accessed data
- [ ] Page load time improved by > 30%
- [ ] Database query count reduced
- [ ] No cache-related errors in logs
- [ ] Cache warm-up completing within 5 minutes

### Monitoring
- [ ] Cache performance widget active in admin
- [ ] Alerts configured for low hit rates
- [ ] Memory usage within bounds
- [ ] Cache eviction rate acceptable (< 5%)

---

## Troubleshooting

### Common Issues

**Redis Connection Failed**
```bash
# Verify Redis is running
systemctl status redis
redis-cli ping

# Check connection
redis-cli -h 127.0.0.1 -p 6379 ping
```

**Cache Not Invalidating**
- Check hook execution order
- Verify cache key format matches
- Enable debug logging in configuration.php

**High Memory Usage**
```php
// Set max memory in configuration
$redis_cache['maxmemory'] = '256mb';
$redis_cache['maxmemory_policy'] = 'allkeys-lru';
```

**Poor Hit Rate**
- Review cache key patterns
- Adjust TTL values
- Implement cache warming
- Check for unnecessary cache bypasses