# WHMCS Advanced Caching

Complete guide to advanced caching strategies.

## Overview

Implement sophisticated caching for optimal performance.

## Cache Layer Architecture

### Multi-Tier Cache

```php
<?php
/**
 * Multi-tier cache implementation
 */
class MultiTierCache
{
    private array $tiers = [];
    
    public function __construct()
    {
        // Memory cache (Redis/Memcached)
        $this->tiers['memory'] = new RedisCache([
            'host' => REDIS_HOST,
            'port' => REDIS_PORT,
            'prefix' => 'whmcs:',
        ]);
        
        // File cache
        $this->tiers['file'] = new FileCache([
            'cache_dir' => CACHE_DIR,
        ]);
        
        // Database cache
        $this->tiers['database'] = new DatabaseCache();
    }
    
    /**
     * Get value from cache
     */
    public function get(string $key, $default = null)
    {
        // Check fastest tier first
        foreach ($this->tiers as $tier) {
            $value = $tier->get($key);
            if ($value !== null) {
                // Populate slower tiers
                $this->populateSlowerTiers($key, $value);
                return $value;
            }
        }
        
        return $default;
    }
    
    /**
     * Set value in all tiers
     */
    public function set(string $key, $value, int $ttl = 3600): void
    {
        foreach ($this->tiers as $tier) {
            $tier->set($key, $value, $ttl);
        }
    }
    
    /**
     * Delete from all tiers
     */
    public function delete(string $key): void
    {
        foreach ($this->tiers as $tier) {
            $tier->delete($key);
        }
    }
    
    /**
     * Populate slower tiers
     */
    private function populateSlowerTiers(string $key, $value): void
    {
        $tierOrder = ['memory', 'file', 'database'];
        $hitTier = null;
        
        foreach ($tierOrder as $tier) {
            if ($this->tiers[$tier]->get($key) !== null) {
                $hitTier = $tier;
                break;
            }
        }
        
        // Populate tiers after the hit tier
        $started = false;
        foreach ($tierOrder as $tier) {
            if ($tier === $hitTier) {
                $started = true;
                continue;
            }
            
            if ($started) {
                $this->tiers[$tier]->set($key, $value, 3600);
            }
        }
    }
}
```

## Redis Caching

### Redis Cache Client

```php
<?php
/**
 * Redis cache implementation
 */
class RedisCache
{
    private $redis;
    private string $prefix;
    
    public function __construct(array $config)
    {
        $this->redis = new Redis();
        $this->redis->connect($config['host'], $config['port']);
        $this->prefix = $config['prefix'] ?? 'cache:';
    }
    
    /**
     * Get value
     */
    public function get(string $key)
    {
        $value = $this->redis->get($this->prefix . $key);
        return $value !== false ? unserialize($value) : null;
    }
    
    /**
     * Set value with TTL
     */
    public function set(string $key, $value, int $ttl = 3600): bool
    {
        $serialized = serialize($value);
        return $this->redis->setex($this->prefix . $key, $ttl, $serialized);
    }
    
    /**
     * Get multiple keys
     */
    public function getMultiple(array $keys): array
    {
        $prefixed = array_map(fn($k) => $this->prefix . $k, $keys);
        $values = $this->redis->mGet($prefixed);
        
        $result = [];
        foreach ($keys as $i => $key) {
            $result[$key] = $values[$i] !== false 
                ? unserialize($values[$i]) 
                : null;
        }
        
        return $result;
    }
    
    /**
     * Set multiple values
     */
    public function setMultiple(array $items, int $ttl = 3600): bool
    {
        $this->redis->multi();
        
        foreach ($items as $key => $value) {
            $this->redis->setex($this->prefix . $key, $ttl, serialize($value));
        }
        
        return $this->redis->exec() !== false;
    }
    
    /**
     * Delete key
     */
    public function delete(string $key): bool
    {
        return $this->redis->del($this->prefix . $key) > 0;
    }
    
    /**
     * Delete by pattern
     */
    public function deletePattern(string $pattern): int
    {
        $keys = $this->redis->keys($this->prefix . $pattern);
        return count($keys) > 0 ? $this->redis->del($keys) : 0;
    }
    
    /**
     * Increment value
     */
    public function increment(string $key, int $value = 1): int
    {
        return $this->redis->incrBy($this->prefix . $key, $value);
    }
    
    /**
     * Get cache statistics
     */
    public function stats(): array
    {
        $info = $this->redis->info('memory');
        return [
            'used_memory' => $info['used_memory_human'] ?? 'unknown',
            'connected_clients' => $this->redis->info('clients')['connected_clients'] ?? 0,
        ];
    }
}
```

## Cache Warming

### Cache Warmer

```php
<?php
/**
 * Cache warming service
 */
class CacheWarmer
{
    private MultiTierCache $cache;
    
    public function __construct(MultiTierCache $cache)
    {
        $this->cache = $cache;
    }
    
    /**
     * Warm all caches
     */
    public function warmAll(): array
    {
        $results = [
            'products' => $this->warmProductCache(),
            'clients' => $this->warmClientCache(),
            'services' => $this->warmServiceCache(),
        ];
        
        logActivity('Cache warmed: ' . json_encode($results));
        
        return $results;
    }
    
    /**
     * Warm product cache
     */
    private function warmProductCache(): int
    {
        $products = Capsule::table('tblproducts')
            ->join('tblproductgroups', 'tblproducts.gid', '=', 'tblproductgroups.id')
            ->get();
        
        $count = 0;
        foreach ($products as $product) {
            $this->cache->set(
                "product:{$product->id}",
                (array)$product,
                7200
            );
            $count++;
        }
        
        return $count;
    }
    
    /**
     * Warm client cache (popular clients)
     */
    private function warmClientCache(): int
    {
        // Get top 100 most active clients
        $clients = Capsule::table('tblclients')
            ->selectRaw('tblclients.*, COUNT(tblorders.id) as order_count')
            ->leftJoin('tblorders', 'tblclients.id', '=', 'tblorders.userid')
            ->groupBy('tblclients.id')
            ->orderBy('order_count', 'desc')
            ->limit(100)
            ->get();
        
        $count = 0;
        foreach ($clients as $client) {
            $this->cache->set(
                "client:{$client->id}",
                [
                    'basic' => [
                        'id' => $client->id,
                        'email' => $client->email,
                        'name' => "{$client->firstname} {$client->lastname}",
                    ],
                    'services_count' => Capsule::table('tblhosting')
                        ->where('userid', $client->id)
                        ->where('domainstatus', 'Active')
                        ->count(),
                ],
                3600
            );
            $count++;
        }
        
        return $count;
    }
    
    /**
     * Warm service cache
     */
    private function warmServiceCache(): int
    {
        $services = Capsule::table('tblhosting')
            ->where('domainstatus', 'Active')
            ->limit(500)
            ->get();
        
        $count = 0;
        foreach ($services as $service) {
            $this->cache->set(
                "service:{$service->id}",
                [
                    'id' => $service->id,
                    'domain' => $service->domain,
                    'status' => $service->domainstatus,
                    'next_due' => $service->nextduedate,
                ],
                1800
            );
            $count++;
        }
        
        return $count;
    }
}
```

## Fragment Caching

### Template Fragment Cache

```php
<?php
/**
 * Fragment caching for templates
 */
class FragmentCache
{
    private RedisCache $cache;
    
    public function __construct(RedisCache $cache)
    {
        $this->cache = $cache;
    }
    
    /**
     * Cache template fragment
     */
    public function remember(string $key, int $ttl, callable $callback)
    {
        $cached = $this->cache->get("fragment:{$key}");
        
        if ($cached !== null) {
            return $cached;
        }
        
        $content = $callback();
        $this->cache->set("fragment:{$key}", $content, $ttl);
        
        return $content;
    }
    
    /**
     * Render cached fragment
     */
    public function render(string $key, int $ttl, string $template, array $data = []): string
    {
        return $this->remember($key, $ttl, function() use ($template, $data) {
            extract($data);
            ob_start();
            include $template;
            return ob_get_clean();
        });
    }
    
    /**
     * Invalidate fragment
     */
    public function invalidate(string $key): bool
    {
        return $this->cache->delete("fragment:{$key}");
    }
    
    /**
     * Invalidate multiple fragments
     */
    public function invalidatePattern(string $pattern): int
    {
        return $this->cache->deletePattern("fragment:{$pattern}");
    }
}

/**
 * Usage in templates
 */
function renderProductCard(array $product): string
{
    $cache = new FragmentCache(new RedisCache(['host' => REDIS_HOST, 'port' => REDIS_PORT]));
    
    return $cache->remember("product_card:{$product['id']}", 1800, function() use ($product) {
        return "<div class='product-card'>
            <h3>{$product['name']}</h3>
            <p>{$product['description']}</p>
            <span class='price'>{$product['price']}</span>
        </div>";
    });
}
```

## Cache Invalidation

### Smart Invalidation

```php
<?php
/**
 * Cache invalidation strategies
 */
class CacheInvalidator
{
    private MultiTierCache $cache;
    
    public function __construct(MultiTierCache $cache)
    {
        $this->cache = $cache;
    }
    
    /**
     * Invalidate on model update
     */
    public function onModelUpdate(string $model, int $id, array $changedFields): void
    {
        // Invalidate specific item
        $this->cache->delete("{$model}:{$id}");
        
        // Invalidate related collections
        switch ($model) {
            case 'client':
                $this->invalidateClientRelations($id);
                break;
            case 'product':
                $this->invalidateProductRelations($id);
                break;
            case 'service':
                $this->invalidateServiceRelations($id);
                break;
        }
        
        // Invalidate tag-based caches
        foreach ($changedFields as $field) {
            $this->cache->deletePattern("*:by_{$field}:*");
        }
    }
    
    /**
     * Invalidate client relations
     */
    private function invalidateClientRelations(int $clientId): void
    {
        $this->cache->deletePattern("client:{$clientId}");
        $this->cache->deletePattern("client:{$clientId}:*");
        
        // Invalidate related services
        $services = Capsule::table('tblhosting')
            ->where('userid', $clientId)
            ->pluck('id')
            ->toArray();
        
        foreach ($services as $serviceId) {
            $this->cache->delete("service:{$serviceId}");
        }
    }
    
    /**
     * Invalidate product relations
     */
    private function invalidateProductRelations(int $productId): void
    {
        $this->cache->delete("product:{$productId}");
        $this->cache->delete('products:all');
        $this->cache->deletePattern("product:*");
    }
    
    /**
     * Invalidate service relations
     */
    private function invalidateServiceRelations(int $serviceId): void
    {
        $this->cache->delete("service:{$serviceId}");
        
        // Invalidate client cache
        $service = Capsule::table('tblhosting')
            ->where('id', $serviceId)
            ->first();
        
        if ($service) {
            $this->cache->delete("client:{$service->userid}");
        }
    }
}
```

## Best Practices

1. **Use appropriate TTLs** - Different data needs different cache times
2. **Invalidate properly** - Clear cache when data changes
3. **Monitor hit rates** - Track cache effectiveness
4. **Warm critical caches** - Pre-populate on deployment
5. **Handle failures** - Graceful degradation when cache unavailable
6. **Memory management** - Evict old entries appropriately

## Related Documentation

- [whmcs-advanced-performance.md](whmcs-advanced-performance.md)
- [whmcs-advanced-database.md](whmcs-advanced-database.md)
