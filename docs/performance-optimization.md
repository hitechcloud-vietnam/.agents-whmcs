# WHMCS Performance Optimization

**Version:** 8.x
**Updated:** 2026-05-28
**Related Skills:** `caching-strategies`, `database-indexing-guide`, `security-best-practices`

## Overview

Optimizing WHMCS performance involves multiple layers: server configuration, database optimization, caching implementation, and code-level improvements.

## Server-Level Optimization

### PHP Configuration

```ini
; php.ini optimization

; Opcache settings (critical for performance)
opcache.enable=1
opcache.enable_cli=0
opcache.memory_consumption=256
opcache.max_accelerated_files=10000
opcache.revalidate_freq=0
opcache.validate_timestamps=0
opcache.save_comments=1
opcache.fast_shutdown=1

; Realpath cache
realpath_cache_size=4096K
realpath_cache_ttl=600

; Memory limits
memory_limit = 512M
max_execution_time = 120

; Session optimization
session.save_handler = redis
session.save_path = "tcp://127.0.0.1:6379?database=1"
session.gc_maxlifetime = 86400
```

### OPcache Configuration

```php
// includes/additional_config.php

// OPcache optimization
if (function_exists('opcache_get_configuration')) {
    opcache_set('opcache.enable', 1);
    opcache_set('opcache.memory_consumption', 256);
    opcache_set('opcache.max_accelerated_files', 10000);
    opcache_set('opcache.revalidate_freq', 0);
    opcache_set('opcache.validate_timestamps', 0);
}
```

## Database Optimization

### Query Optimization

```php
// Optimized database queries

// Instead of loading all records and filtering in PHP
$clients = Capsule::table('tblclients')
    ->where('status', 'Active')
    ->where('domain', 'LIKE', '%example.com%')
    ->select(['id', 'firstname', 'lastname', 'email'])
    ->limit(100)
    ->offset(0)
    ->get();

// Use indexes for better performance
// See database-indexing-guide.md

// Batch processing for large datasets
function processInBatches(callable $callback, $query, $batchSize = 1000)
{
    $offset = 0;

    do {
        $batch = (clone $query)
            ->offset($offset)
            ->limit($batchSize)
            ->get();

        $count = count($batch);

        if ($count > 0) {
            $callback($batch);
        }

        $offset += $batchSize;

    } while ($count === $batchSize);
}

// Usage
processInBatches(
    function($clients) {
        foreach ($clients as $client) {
            // Process each client
        }
    },
    Capsule::table('tblclients')->where('status', 'Active')
);
```

### Connection Pooling

```php
// configuration.php - Multiple DB connections strategy

$mysql_config = [
    'host' => 'localhost',
    'port' => 3306,
    'database' => 'whmcs',
    'username' => 'whmcs_user',
    'password' => 'secure_password',
    'charset' => 'utf8mb4',
    'strict' => true,
    'engine' => 'InnoDB',

    // Connection pooling
    'options' => [
        \PDO::ATTR_PERSISTENT => true,
        \PDO::ATTR_EMULATE_PREPARES => false,
        \PDO::MYSQL_ATTR_INIT_COMMAND => "SET sql_mode=''",
    ],
];
```

## Caching Implementation

### WHMCS Cache Configuration

```php
// configuration.php

// Redis cache configuration
$redis_cache = [
    'host' => '127.0.0.1',
    'port' => 6379,
    'database' => 0,
    'password' => null,
    'timeout' => 2.5,
];

// Use file cache as fallback
$file_cache = [
    'path' => ROOTDIR . '/storage/cache',
    'ttl' => 3600,
];
```

### Custom Cache Usage

```php
// includes/hooks/performance_optimization.php

// Cache expensive queries
function getCachedClientStats(int $clientId, int $ttl = 3600): array
{
    $cache = \WHMCS\Cache::store('redis');
    $cacheKey = "client_stats_{$clientId}";

    $data = $cache->get($cacheKey);

    if ($data === null) {
        // Fetch from database
        $data = [
            'total_services' => Capsule::table('tblhosting')
                ->where('userid', $clientId)
                ->where('domainstatus', 'Active')
                ->count(),
            'total_orders' => Capsule::table('tblorders')
                ->where('userid', $clientId)
                ->count(),
            'total_spent' => Capsule::table('tblinvoices')
                ->where('userid', $clientId)
                ->where('status', 'Paid')
                ->sum('total'),
        ];

        $cache->set($cacheKey, $data, $ttl);
    }

    return $data;
}

// Cache API responses
function cacheApiResponse(string $endpoint, array $params, $data, int $ttl = 300): void
{
    $cache = \WHMCS\Cache::store('redis');
    $cacheKey = "api_" . md5($endpoint . serialize($params));

    $cache->set($cacheKey, $data, $ttl);
}
```

### Full Page Caching

```php
// includes/hooks/page_cache.php

add_hook('PreTemplateRender', 1, function($vars) {
    // Only cache for non-logged-in users
    if (!empty($_SESSION['uid'])) {
        return;
    }

    $cache = \WHMCS\Cache::store('file');
    $cacheKey = 'page_' . md5($_SERVER['REQUEST_URI']);

    // Try to serve from cache
    $cached = $cache->get($cacheKey);

    if ($cached !== null) {
        echo $cached;
        exit;
    }
});

add_hook('AfterApplicationOutput', 1, function($vars) {
    // Store rendered page in cache
    $cache = \WHMCS\Cache::store('file');
    $cacheKey = 'page_' . md5($_SERVER['REQUEST_URI']);

    if (empty($_SESSION['uid'])) {
        $cache->set($cacheKey, $vars['output'], 300); // 5 minute TTL
    }
});
```

## Template Optimization

### Reduce Smarty Overhead

```smarty
{* Disable template checking for production *}
{config_load file='global' assign='config'}

{* Cache expensive operations *}
{assign var='current_time' value=$smarty.now}

{* Use registerPlugin for custom functions *}

{* Avoid repeated function calls *}
{assign var='user_logged_in' value=$loggedin}

{if $user_logged_in}
    {* Logged in content *}
{/if}

{* Optimize loops *}
{foreach $items as $item name="items_loop"}
    {if $smarty.foreach.items_loop.index < 10}
        {* Only render first 10 items *}
        {$item.name}
    {/if}
{/foreach}
```

### Asset Optimization

```php
// includes/hooks/asset_optimization.php

add_hook('ClientAreaHeaderOutput', 1, function($vars) {
    $output = '';

    // Minify CSS inline
    $output .= '<style>
        /* Critical CSS here */
        body{margin:0;font-family:sans-serif}
    </style>';

    // Async non-critical CSS
    $output .= '<link rel="preload" href="/templates/your-theme/assets/css/main.min.css" as="style" onload="this.onload=null;this.rel=\'stylesheet\'">';
    $output .= '<noscript><link rel="stylesheet" href="/templates/your-theme/assets/css/main.min.css"></noscript>';

    return $output;
});

add_hook('ClientAreaFooterOutput', 1, function($vars) {
    // Defer JavaScript loading
    return '<script defer src="/templates/your-theme/assets/js/main.min.js"></script>';
});
```

## Image Optimization

```php
// includes/hooks/image_optimization.php

class ImageOptimizer
{
    public static function optimize(string $imagePath): bool
    {
        if (!extension_loaded('imagick')) {
            return false;
        }

        try {
            $image = new Imagick($imagePath);

            // Strip metadata
            $image->stripImage();

            // Optimize
            $image->setImageCompressionQuality(85);
            $image->setInterlaceScheme(Imagick::INTERLACE_PLANE);
            $image->setSamplingFactors(['2x2', '1x1', '1x1']);

            // Convert to progressive JPEG
            $image->setFormat('jpg');
            $image->setInterlaceScheme(Imagick::INTERLACE_PLANE);

            $image->writeImage($imagePath);
            $image->destroy();

            return true;
        } catch (\Exception $e) {
            return false;
        }
    }
}

// Lazy load images
add_hook('ClientAreaHeaderOutput', 1, function($vars) {
    return '<script>
        document.addEventListener("DOMContentLoaded", function() {
            var lazyImages = document.querySelectorAll("img[data-src]");
            var observer = new IntersectionObserver(function(entries) {
                entries.forEach(function(entry) {
                    if (entry.isIntersecting) {
                        entry.target.src = entry.target.dataset.src;
                        observer.unobserve(entry.target);
                    }
                });
            });
            lazyImages.forEach(function(img) { observer.observe(img); });
        });
    </script>';
});
```

## Module Performance

### Module Command Optimization

```php
// Optimize module commands

class OptimizedServerModule
{
    protected $cache;

    public function __construct()
    {
        $this->cache = \WHMCS\Cache::store('redis');
    }

    // Cache API calls to remote server
    public function getAccountInfo(array $params): array
    {
        $cacheKey = "server_account_{$params['serviceid']}";

        $cached = $this->cache->get($cacheKey);
        if ($cached !== null) {
            return $cached;
        }

        // Fetch from server
        $result = $this->callApi('GetAccountInfo', $params);

        // Cache for 5 minutes
        $this->cache->set($cacheKey, $result, 300);

        return $result;
    }

    // Batch operations when possible
    public function batchUpdate(array $services): array
    {
        $batch = [];
        foreach ($services as $service) {
            $batch[] = $this->prepareUpdateParams($service);
        }

        return $this->callApi('BatchUpdate', $batch);
    }
}
```

## Monitoring Performance

### Performance Logging

```php
// includes/hooks/perf_monitoring.php

add_hook('PreTemplateRender', 1, function($vars) {
    if (!defined('PERF_START')) {
        define('PERF_START', microtime(true));
    }
});

add_hook('AfterApplicationOutput', 1, function($vars) {
    $duration = microtime(true) - PERF_START;

    // Log slow pages
    if ($duration > 2.0) {
        logActivity("Slow page detected: {$_SERVER['REQUEST_URI']} - {$duration}s");

        Capsule::table('tblperformance_log')->insert([
            'timestamp' => date('Y-m-d H:i:s'),
            'url' => $_SERVER['REQUEST_URI'],
            'duration' => $duration,
            'memory_usage' => memory_get_peak_usage(true),
            'query_count' => Capsule::connection()->getQueryLog() ? count(Capsule::connection()->getQueryLog()) : 0,
        ]);
    }
});

// Log database queries
add_hook('DatabaseQueryExecuted', 1, function($vars) {
    if ($vars['time'] > 0.1) { // Log slow queries (>100ms)
        logActivity("Slow query: {$vars['query']} - {$vars['time']}s");
    }
});
```

### Performance Dashboard

```php
// admin/custom/perf_dashboard.php

add_hook('AdminHomepage', 1, function($vars) {
    // Get performance stats
    $stats = Capsule::table('tblperformance_log')
        ->selectRaw('AVG(duration) as avg_duration, COUNT(*) as total')
        ->where('timestamp', '>=', date('Y-m-d H:i:s', strtotime('-24 hours')))
        ->first();

    $slowPages = Capsule::table('tblperformance_log')
        ->orderBy('duration', 'desc')
        ->limit(5)
        ->get();

    return [
        'displayFunction' => 'renderPerfDashboard',
        'vars' => [
            'stats' => $stats,
            'slowPages' => $slowPages,
        ],
    ];
});

function renderPerfDashboard(array $vars): void
{
    echo '<div class="admin-widget">';
    echo '<h4>Performance</h4>';
    echo '<p>Avg Load: ' . number_format($vars['stats']->avg_duration, 3) . 's</p>';
    echo '<p>Requests: ' . $vars['stats']->total . '</p>';
    echo '<ul>';
    foreach ($vars['slowPages'] as $page) {
        echo '<li>' . htmlspecialchars($page->url) . ' (' . number_format($page->duration, 2) . 's)</li>';
    }
    echo '</ul>';
    echo '</div>';
}
```

## Quick Wins Checklist

1. Enable OPcache with proper configuration
2. Use Redis for session and cache storage
3. Enable MySQL query cache
4. Use CDN for static assets
5. Minify and combine CSS/JS files
6. Optimize images (WebP, lazy loading)
7. Enable GZIP compression
8. Use HTTP/2 for multiplexing
9. Implement database query caching
10. Monitor and optimize slow queries

## Related Documentation

- [Caching Strategies](caching-strategies.md)
- [Database Indexing Guide](database-indexing-guide.md)
- [Security Best Practices](security-best-practices.md)
