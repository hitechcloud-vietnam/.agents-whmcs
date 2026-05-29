# WHMCS Cache Layer

## Overview

The cache layer improves performance by storing frequently accessed data in memory.

## Configuration

```php
<?php
// config/cache.php or early in execution
use WHMCS\Session\Session;
use Illuminate\Support\Facades\Cache;

$whmcsCache = new \WHMCS\Cache\CacheManager([
    'default' => 'file', // file, redis, memcached
    'stores' => [
        'file' => [
            'driver' => 'file',
            'path' => ROOTDIR . '/data/cache',
            'expire' => 60,
        ],
        'redis' => [
            'driver' => 'redis',
            'host' => '127.0.0.1',
            'port' => 6379,
            'password' => null,
            'database' => 0,
        ],
    ],
    'prefix' => 'whmcs_',
]);
```

## Basic Cache Operations

### Store Data

```php
<?php
use Illuminate\Support\Facades\Cache;

// Simple store
Cache::put('key', 'value', 3600); // 1 hour

// Store forever
Cache::forever('key', 'value');

// Conditional store (only if not exists)
Cache::putIfAbsent('key', 'value', 3600);

// Multiple items
Cache::putMany([
    'key1' => 'value1',
    'key2' => 'value2',
], 3600);
```

### Retrieve Data

```php
<?php
// Simple get
$value = Cache::get('key');

// Default value if not found
$value = Cache::get('key', 'default');

// Check exists
if (Cache::has('key')) {
    // Key exists
}

// Get and delete
$value = Cache::pull('key'); // Returns value and removes key
```

### Delete Data

```php
<?php
// Delete single key
Cache::forget('key');

// Delete multiple
Cache::forget(['key1', 'key2', 'key3']);

// Flush all cache
Cache::flush();
```

## Cache Tags

```php
<?php
// Store with tags
Cache::tags(['clients', 'active'])->put('client_1', $data, 3600);
Cache::tags(['clients', 'premium'])->put('client_2', $data, 3600);

// Get tagged items
$clients = Cache::tags(['clients'])->get('client_1');

// Flush by tag
Cache::tags(['clients'])->flush();

// Flush multiple tags
Cache::tags(['clients', 'premium'])->flush();
```

## Cache Helpers

```php
<?php
// Remember pattern - cache or compute
$clients = Cache::remember('active_clients', 3600, function () {
    return Capsule::table('tblclients')
        ->where('status', 'Active')
        ->get();
});

// Remember with tags
$clients = Cache::tags(['clients'])->remember('active', 3600, function () {
    return Client::active()->get();
});

// Forget and refresh
function refreshClientCache(int $clientId): void
{
    Cache::forget('client_' . $clientId);
    Cache::tags(['clients'])->flush();
}
```

## WHMCS-Specific Caching

### Product Cache

```php
<?php
function getProductWithCache(int $productId): ?object
{
    return Cache::remember("product_{$productId}", 3600, function () use ($productId) {
        return Capsule::table('tblproducts')
            ->where('id', $productId)
            ->first();
    });
}

function invalidateProductCache(int $productId): void
{
    Cache::forget("product_{$productId}");
}
```

### Client Cache

```php
<?php
function getClientWithCache(int $clientId): ?object
{
    return Cache::remember("client_{$clientId}", 1800, function () use ($clientId) {
        return Capsule::table('tblclients')
            ->where('id', $clientId)
            ->first();
    });
}

function getClientServicesWithCache(int $clientId): array
{
    return Cache::tags(['client_services', 'client_' . $clientId])
        ->remember('services', 1800, function () use ($clientId) {
            return Capsule::table('tblhosting')
                ->where('userid', $clientId)
                ->get();
        });
}
```

## Distributed Cache

```php
<?php
class DistributedCache
{
    private \Illuminate\Contracts\Cache\Store $store;
    
    public function __construct()
    {
        $this->store = Cache::store('redis')->getStore();
    }
    
    public function lock(string $key, int $seconds = 10): bool
    {
        $lock = Cache::lock($key, $seconds);
        
        if ($lock->get()) {
            return true;
        }
        
        return false;
    }
    
    public function release(string $key): void
    {
        Cache::lock($key)->release();
    }
    
    public function atomicIncrement(string $key, int $value = 1): int
    {
        return Cache::increment($key, $value);
    }
}

// Usage
$cache = new DistributedCache();

// Acquire lock
if ($cache->lock('processing_order_' . $orderId)) {
    try {
        processOrder($orderId);
    } finally {
        $cache->release('processing_order_' . $orderId);
    }
}
```

## Cache Warming

```php
<?php
function warmProductCache(): void
{
    $products = Capsule::table('tblproducts')->get();
    
    foreach ($products as $product) {
        Cache::put("product_{$product->id}", $product, 3600);
    }
    
    // Also warm related caches
    $categories = Capsule::table('tblproductgroups')->get();
    foreach ($categories as $category) {
        $categoryProducts = Capsule::table('tblproducts')
            ->where('gid', $category->id)
            ->get();
        
        Cache::put("category_{$category->id}_products", $categoryProducts, 3600);
    }
}
```

## Best Practices

1. **Set appropriate TTL** - Balance freshness and performance
2. **Use tags** - Invalidate related caches efficiently
3. **Handle cache misses** - Always have fallback
4. **Monitor cache size** - Prevent memory issues
5. **Use locking** - Prevent cache stampedes

## Related Documentation

- [WHMCS Session Storage](/docs/whmcs-session-storage.md)
- [WHMCS File Storage](/docs/whmcs-file-storage.md)