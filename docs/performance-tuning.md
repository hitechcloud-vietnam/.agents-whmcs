# WHMCS Performance Tuning Guide
# Version: 1.0 | Updated: 2026-05-28

## Purpose

Optimize WHMCS performance for high-traffic environments.

## PHP Optimization

### OPcache Configuration
```ini
; php.ini
opcache.enable=1
opcache.memory_consumption=256
opcache.interned_strings_buffer=16
opcache.max_accelerated_files=10000
opcache.revalidate_freq=0
opcache.validate_timestamps=0
opcache.save_comments=1
```

### PHP-FPM Configuration
```ini
; www.conf
pm = dynamic
pm.max_children = 50
pm.start_servers = 10
pm.min_spare_servers = 5
pm.max_spare_servers = 20
pm.max_requests = 500
```

## WHMCS Caching

### Database Caching
```php
<?php
// Enable query caching
Capsule::connection()->enableQueryCache();

// Configure cache TTL
Capsule::connection()->setQueryCacheDuration(300);
```

### Application Caching
```php
<?php
// Use WHMCS caching for expensive operations
use WHMCS\Session\Storage as SessionStorage;

$cache = \App::getRuntimeStorage();

// Cache service data
$services = $cache->getOrSet('user_services_' . $userId, function() use ($userId) {
    return Capsule::table('tblhosting')
        ->where('userid', $userId)
        ->get()
        ->toArray();
}, 1800); // 30 minutes TTL
```

## Template Caching

### Smarty Configuration
```php
<?php
// configuration.php additions

$templates.compile_check = false; // Disable for production
$templates.cache = true; // Enable template caching
$templates.compile_id = 'v2'; // Version for cache busting
```

## Database Optimization

### Connection Pooling
```php
<?php
// Use persistent connections
Capsule::connection()->setAttribute(PDO::ATTR_PERSISTENT, true);
```

### Query Optimization
```php
<?php
// Bad
foreach ($users as $user) {
    $services = Capsule::table('tblhosting')
        ->where('userid', $user->id)
        ->get(); // N+1 query problem
}

// Good - Use JOIN
$users = Capsule::table('tblclients')
    ->join('tblhosting', 'tblclients.id', '=', 'tblhosting.userid')
    ->select('tblclients.*', 'tblhosting.domain', 'tblhosting.id as service_id')
    ->where('tblclients.id', $userId)
    ->get();
```

## Asset Optimization

### Minification
```php
<?php
// Add to hooks
add_hook('AdminAreaHeaderOutput', 1, function($vars) {
    return '<link rel="stylesheet" href="https://whmcs.example.com/assets/css/admin.min.css">';
});

add_hook('ClientAreaHeaderOutput', 1, function($vars) {
    return '<link rel="stylesheet" href="https://whmcs.example.com/assets/css/client.min.css">';
});
```

### CDN Integration
```php
<?php
// Replace local URLs with CDN
function cdnUrl(string $path): string {
    $cdnBase = 'https://cdn.whmcs.example.com';

    if (defined('CDN_ENABLED') && CDN_ENABLED) {
        return $cdnBase . $path;
    }

    return $path;
}
```

## Session Optimization

```php
<?php
// Use Redis for sessions
ini_set('session.save_handler', 'redis');
ini_set('session.save_path', 'tcp://127.0.0.1:6379?database=0');
```

## Monitoring

### Performance Metrics
```php
<?php
function getPerformanceMetrics(): array {
    return [
        'php_memory' => memory_get_usage(true) / 1024 / 1024,
        'php_peak_memory' => memory_get_peak_usage(true) / 1024 / 1024,
        'query_count' => Capsule::getQueryCount(),
        'query_time' => Capsule::getTotalQueryTime(),
        'page_time' => microtime(true) - $_SERVER['REQUEST_TIME_FLOAT'],
    ];
}
```

---

**Related Skills:**
- whmcs-performance-optimization
- whmcs-database-design
- whmcs-redis-cache