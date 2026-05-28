# Caching Strategies for WHMCS

Caching is essential for optimizing WHMCS performance. This guide covers comprehensive caching strategies and implementation patterns.

## Cache Layer Architecture

### Cache Manager Interface

```php
<?php
/**
 * Cache manager interface
 */
interface CacheManagerInterface
{
    public function get(string $key, $default = null);
    public function set(string $key, $value, int $ttl = 3600): bool;
    public function delete(string $key): bool;
    public function has(string $key): bool;
    public function flush(): bool;
    public function remember(string $key, int $ttl, callable $callback);
}
```

### Multi-Tier Cache Implementation

```php
<?php
/**
 * Multi-tier cache manager
 */
class MultiTierCacheManager implements CacheManagerInterface
{
    private array $stores = [];
    private string $defaultNamespace = 'module';
    private int $defaultTtl = 3600;

    public function __construct(array $config = [])
    {
        // Initialize cache stores by priority
        $this->stores['memory'] = new MemoryCacheStore();
        $this->stores['redis'] = new RedisCacheStore($config['redis'] ?? []);
        $this->stores['database'] = new DatabaseCacheStore();
    }

    /**
     * Get value from cache (checks all tiers)
     */
    public function get(string $key, $default = null)
    {
        $fullKey = $this->prepareKey($key);

        // Check memory first (fastest)
        if ($this->stores['memory']->has($fullKey)) {
            return $this->stores['memory']->get($fullKey);
        }

        // Check Redis
        if ($this->stores['redis']->has($fullKey)) {
            $value = $this->stores['redis']->get($fullKey);
            // Promote to memory
            $this->stores['memory']->set($fullKey, $value);
            return $value;
        }

        // Check database
        if ($this->stores['database']->has($fullKey)) {
            $value = $this->stores['database']->get($fullKey);
            // Promote to Redis and memory
            $this->stores['redis']->set($fullKey, $value);
            $this->stores['memory']->set($fullKey, $value);
            return $value;
        }

        return $default;
    }

    /**
     * Set value in all tiers
     */
    public function set(string $key, $value, int $ttl = 3600): bool
    {
        $fullKey = $this->prepareKey($key);

        // Set in all tiers
        $memory = $this->stores['memory']->set($fullKey, $value, $ttl);
        $redis = $this->stores['redis']->set($fullKey, $value, $ttl);
        $db = $this->stores['database']->set($fullKey, $value, $ttl);

        return $memory && $redis && $db;
    }

    /**
     * Delete from all tiers
     */
    public function delete(string $key): bool
    {
        $fullKey = $this->prepareKey($key);

        $this->stores['memory']->delete($fullKey);
        $this->stores['redis']->delete($fullKey);
        $this->stores['database']->delete($fullKey);

        return true;
    }

    /**
     * Check if key exists
     */
    public function has(string $key): bool
    {
        $fullKey = $this->prepareKey($key);

        return $this->stores['memory']->has($fullKey) ||
               $this->stores['redis']->has($fullKey) ||
               $this->stores['database']->has($fullKey);
    }

    /**
     * Flush all cache
     */
    public function flush(): bool
    {
        foreach ($this->stores as $store) {
            $store->flush();
        }
        return true;
    }

    /**
     * Remember pattern - cache or compute
     */
    public function remember(string $key, int $ttl, callable $callback)
    {
        $value = $this->get($key);

        if ($value !== null) {
            return $value;
        }

        $value = $callback();
        $this->set($key, $value, $ttl);

        return $value;
    }

    /**
     * Cache tags for invalidation
     */
    public function tags(array $tags): TaggedCache
    {
        return new TaggedCache($this, $tags);
    }

    /**
     * Invalidate by tags
     */
    public function invalidateTags(array $tags): void
    {
        $taggedItems = Capsule::table('mod_cache_tags')
            ->whereIn('tag', $tags)
            ->get();

        foreach ($taggedItems as $item) {
            $this->delete($item->cache_key);
        }

        // Clean up tag records
        Capsule::table('mod_cache_tags')
            ->whereIn('tag', $tags)
            ->delete();
    }

    private function prepareKey(string $key): string
    {
        return $this->defaultNamespace . ':' . $key;
    }
}
```

## Cache Stores

### Memory Cache Store

```php
<?php
/**
 * In-memory cache store for request-level caching
 */
class MemoryCacheStore
{
    private static $cache = [];
    private static $expiry = [];

    public function get(string $key, $default = null)
    {
        if ($this->isExpired($key)) {
            $this->delete($key);
            return $default;
        }

        return self::$cache[$key] ?? $default;
    }

    public function set(string $key, $value, int $ttl = 3600): bool
    {
        self::$cache[$key] = $value;
        self::$expiry[$key] = time() + $ttl;
        return true;
    }

    public function has(string $key): bool
    {
        return isset(self::$cache[$key]) && !$this->isExpired($key);
    }

    public function delete(string $key): bool
    {
        unset(self::$cache[$key], self::$expiry[$key]);
        return true;
    }

    public function flush(): bool
    {
        self::$cache = [];
        self::$expiry = [];
        return true;
    }

    private function isExpired(string $key): bool
    {
        if (!isset(self::$expiry[$key])) {
            return true;
        }
        return time() >= self::$expiry[$key];
    }
}
```

### Redis Cache Store

```php
<?php
/**
 * Redis cache store
 */
class RedisCacheStore
{
    private $redis;
    private string $prefix = 'cache:';
    private int $defaultTtl = 3600;

    public function __construct(array $config)
    {
        $this->redis = new Redis();
        $this->redis->connect(
            $config['host'] ?? '127.0.0.1',
            $config['port'] ?? 6379
        );

        if (!empty($config['password'])) {
            $this->redis->auth($config['password']);
        }

        if (!empty($config['prefix'])) {
            $this->prefix = $config['prefix'];
        }
    }

    public function get(string $key, $default = null)
    {
        $value = $this->redis->get($this->prefix . $key);

        if ($value === false) {
            return $default;
        }

        $data = @unserialize($value);
        return $data !== false ? $data : $value;
    }

    public function set(string $key, $value, int $ttl = 3600): bool
    {
        $serialized = is_string($value) ? $value : serialize($value);

        if ($ttl > 0) {
            return $this->redis->setex($this->prefix . $key, $ttl, $serialized);
        }

        return $this->redis->set($this->prefix . $key, $serialized);
    }

    public function has(string $key): bool
    {
        return (bool) $this->redis->exists($this->prefix . $key);
    }

    public function delete(string $key): bool
    {
        return (bool) $this->redis->del($this->prefix . $key);
    }

    public function flush(): bool
    {
        $keys = $this->redis->keys($this->prefix . '*');

        if (!empty($keys)) {
            $this->redis->del($keys);
        }

        return true;
    }

    public function increment(string $key, int $value = 1): int
    {
        return $this->redis->incrby($this->prefix . $key, $value);
    }

    public function decrement(string $key, int $value = 1): int
    {
        return $this->redis->decrby($this->prefix . $key, $value);
    }

    public function remember(string $key, int $ttl, callable $callback)
    {
        $value = $this->get($key);

        if ($value !== null) {
            return $value;
        }

        $value = $callback();
        $this->set($key, $value, $ttl);

        return $value;
    }
}
```

### Database Cache Store

```php
<?php
/**
 * Database-based cache store
 */
class DatabaseCacheStore
{
    private string $table = 'mod_cache';
    private int $defaultTtl = 3600;

    public function get(string $key, $default = null)
    {
        $record = Capsule::table($this->table)
            ->where('cache_key', $key)
            ->where('expires_at', '>', date('Y-m-d H:i:s'))
            ->first();

        if (!$record) {
            return $default;
        }

        $data = @unserialize($record->cache_value);
        return $data !== false ? $data : $record->cache_value;
    }

    public function set(string $key, $value, int $ttl = 3600): bool
    {
        $expiresAt = date('Y-m-d H:i:s', time() + $ttl);
        $serialized = is_string($value) ? $value : serialize($value);

        Capsule::table($this->table)->updateOrInsert(
            ['cache_key' => $key],
            [
                'cache_value' => $serialized,
                'expires_at' => $expiresAt,
                'updated_at' => date('Y-m-d H:i:s'),
            ]
        );

        return true;
    }

    public function has(string $key): bool
    {
        return Capsule::table($this->table)
            ->where('cache_key', $key)
            ->where('expires_at', '>', date('Y-m-d H:i:s'))
            ->exists();
    }

    public function delete(string $key): bool
    {
        return Capsule::table($this->table)
            ->where('cache_key', $key)
            ->delete() > 0;
    }

    public function flush(): bool
    {
        return Capsule::table($this->table)->delete() >= 0;
    }

    public function prune(): int
    {
        return Capsule::table($this->table)
            ->where('expires_at', '<=', date('Y-m-d H:i:s'))
            ->delete();
    }
}
```

## Caching Patterns

### Query Result Caching

```php
<?php
/**
 * Cached query builder
 */
class CachedQueryBuilder
{
    private $cache;
    private string $baseQuery;
    private array $bindings = [];
    private int $ttl = 3600;
    private ?string $cacheKey = null;

    public function __construct(CacheManagerInterface $cache, string $query)
    {
        $this->cache = $cache;
        $this->baseQuery = $query;
    }

    /**
     * Add binding
     */
    public function binding(string $key, $value): self
    {
        $this->bindings[$key] = $value;
        return $this;
    }

    /**
     * Set cache TTL
     */
    public function ttl(int $seconds): self
    {
        $this->ttl = $seconds;
        return $this;
    }

    /**
     * Set custom cache key
     */
    public function cacheKey(string $key): self
    {
        $this->cacheKey = $key;
        return $this;
    }

    /**
     * Execute query with caching
     */
    public function get(): array
    {
        $key = $this->cacheKey ?? $this->generateCacheKey();

        return $this->cache->remember($key, $this->ttl, function () {
            return $this->executeQuery();
        });
    }

    /**
     * Invalidate cache for this query
     */
    public function invalidate(): bool
    {
        $key = $this->cacheKey ?? $this->generateCacheKey();
        return $this->cache->delete($key);
    }

    private function generateCacheKey(): string
    {
        return md5($this->baseQuery . json_encode($this->bindings));
    }

    private function executeQuery(): array
    {
        // Execute the query with bindings
        return Capsule::select($this->baseQuery, array_values($this->bindings));
    }
}

// Usage example
$clients = (new CachedQueryBuilder($cache, "
    SELECT * FROM tblclients
    WHERE status = ?
    ORDER BY id DESC
"))
->binding('status', 'Active')
->ttl(300)
->get();
```

### Entity Caching

```php
<?php
/**
 * Cached entity repository
 */
class CachedEntityRepository
{
    private $cache;
    private string $entityClass;
    private int $ttl = 3600;

    public function __construct(CacheManagerInterface $cache, string $entityClass)
    {
        $this->cache = $cache;
        $this->entityClass = $entityClass;
    }

    /**
     * Find by ID with caching
     */
    public function find(int $id): ?object
    {
        $key = "entity:{$this->entityClass}:{$id}";

        return $this->cache->remember($key, $this->ttl, function () use ($id) {
            return Capsule::table($this->getTable())
                ->where('id', $id)
                ->first();
        });
    }

    /**
     * Find by ID or fail
     */
    public function findOrFail(int $id): object
    {
        $entity = $this->find($id);

        if (!$entity) {
            throw new EntityNotFoundException(
                "Entity {$this->entityClass} with ID {$id} not found"
            );
        }

        return $entity;
    }

    /**
     * Find all with caching
     */
    public function findAll(array $filters = [], int $limit = 100): array
    {
        $key = "entities:{$this->entityClass}:" . md5(json_encode($filters)) . ":{$limit}";

        return $this->cache->remember($key, $this->ttl, function () use ($filters, $limit) {
            $query = Capsule::table($this->getTable());

            foreach ($filters as $column => $value) {
                $query->where($column, $value);
            }

            return $query->limit($limit)->get();
        });
    }

    /**
     * Invalidate entity cache
     */
    public function invalidate(int $id): bool
    {
        $key = "entity:{$this->entityClass}:{$id}";
        return $this->cache->delete($key);
    }

    /**
     * Invalidate all entities of this type
     */
    public function invalidateAll(): bool
    {
        // Use tag-based invalidation
        $this->cache->invalidateTags([$this->entityClass]);
        return true;
    }

    private function getTable(): string
    {
        // Convert entity class to table name
        $className = (new ReflectionClass($this->entityClass))->getShortName();
        return 'mod_' . strtolower($className);
    }
}
```

### View Fragment Caching

```php
<?php
/**
 * View fragment cache helper
 */
class FragmentCache
{
    private $cache;
    private string $key;
    private int $ttl;

    public function __construct(CacheManagerInterface $cache, string $key, int $ttl = 3600)
    {
        $this->cache = $cache;
        $this->key = $key;
        $this->ttl = $ttl;
    }

    /**
     * Remember fragment
     */
    public function remember(callable $callback): string
    {
        return $this->cache->remember($this->key, $this->ttl, $callback);
    }

    /**
     * Check if cached
     */
    public function has(): bool
    {
        return $this->cache->has($this->key);
    }

    /**
     * Get cached content
     */
    public function get(): ?string
    {
        return $this->cache->get($this->key);
    }

    /**
     * Invalidate
     */
    public function invalidate(): bool
    {
        return $this->cache->delete($this->key);
    }
}

// Smarty plugin for fragment caching
function smarty_block_cache($params, $content, Smarty_Internal_Template $template, &$repeat)
{
    if ($repeat) {
        return;
    }

    $key = $params['key'] ?? 'fragment_' . md5($template->getTemplateId() . $content);
    $ttl = $params['ttl'] ?? 3600;

    $cache = new FragmentCache(App::make(CacheManagerInterface::class), $key, $ttl);

    if ($cache->has()) {
        return $cache->get();
    }

    $cachedContent = $cache->remember(function () use ($content) {
        return $content;
    });

    return $cachedContent;
}
```

## Cache Invalidation Strategies

### Event-Based Invalidation

```php
<?php
/**
 * Event-driven cache invalidation
 */
class CacheInvalidationManager
{
    private $cache;

    public function __construct(CacheManagerInterface $cache)
    {
        $this->cache = $cache;
    }

    /**
     * Register cache keys for an entity
     */
    public function registerKeys(string $entityType, int $entityId, array $keys): void
    {
        foreach ($keys as $key) {
            Capsule::table('mod_cache_keys')->updateOrInsert(
                ['cache_key' => $key],
                [
                    'entity_type' => $entityType,
                    'entity_id' => $entityId,
                ]
            );
        }
    }

    /**
     * Invalidate all keys for an entity
     */
    public function invalidateEntity(string $entityType, int $entityId): void
    {
        $keys = Capsule::table('mod_cache_keys')
            ->where('entity_type', $entityType)
            ->where('entity_id', $entityId)
            ->pluck('cache_key');

        foreach ($keys as $key) {
            $this->cache->delete($key);
        }

        // Clean up registry
        Capsule::table('mod_cache_keys')
            ->where('entity_type', $entityType)
            ->where('entity_id', $entityId)
            ->delete();
    }

    /**
     * Invalidate by pattern
     */
    public function invalidatePattern(string $pattern): void
    {
        // Implementation depends on cache backend
    }
}

// Hook for automatic invalidation
add_hook('AfterModuleCronJob', 1, function ($vars) {
    $cache = new CacheInvalidationManager(App::make(CacheManagerInterface::class));
    $cache->invalidateEntity('module_cron', $vars['module_id']);
});
```

## Cache Monitoring

### Cache Statistics

```php
<?php
/**
 * Cache statistics collector
 */
class CacheStatistics
{
    private $cache;

    public function __construct(CacheManagerInterface $cache)
    {
        $this->cache = $cache;
    }

    /**
     * Track cache hit
     */
    public function recordHit(string $key): void
    {
        Capsule::table('mod_cache_stats')->insert([
            'cache_key' => $key,
            'type' => 'hit',
            'created_at' => date('Y-m-d H:i:s'),
        ]);
    }

    /**
     * Track cache miss
     */
    public function recordMiss(string $key): void
    {
        Capsule::table('mod_cache_stats')->insert([
            'cache_key' => $key,
            'type' => 'miss',
            'created_at' => date('Y-m-d H:i:s'),
        ]);
    }

    /**
     * Get hit rate for period
     */
    public function getHitRate(DateTime $since): float
    {
        $stats = Capsule::table('mod_cache_stats')
            ->where('created_at', '>=', $since->format('Y-m-d H:i:s'))
            ->selectRaw("
                SUM(CASE WHEN type = 'hit' THEN 1 ELSE 0 END) as hits,
                SUM(CASE WHEN type = 'miss' THEN 1 ELSE 0 END) as misses
            ")
            ->first();

        $total = ($stats->hits ?? 0) + ($stats->misses ?? 0);

        if ($total === 0) {
            return 0.0;
        }

        return ($stats->hits ?? 0) / $total * 100;
    }

    /**
     * Get top cache keys
     */
    public function getTopKeys(int $limit = 20): array
    {
        return Capsule::table('mod_cache_stats')
            ->selectRaw('cache_key, COUNT(*) as accesses')
            ->groupBy('cache_key')
            ->orderByRaw('accesses DESC')
            ->limit($limit)
            ->get();
    }
}
```

## Best Practices Summary

| Practice | Description |
|----------|-------------|
| Use multi-tier caching | Memory > Redis > Database |
| Set appropriate TTLs | Vary by data volatility |
| Invalidate on updates | Use event-driven invalidation |
| Monitor hit rates | Track and optimize cache efficiency |
| Use cache versioning | Handle schema changes |
| Prune expired entries | Regular cleanup jobs |
| Compress large values | Save memory/bandwidth |
| Use namespacing | Prevent key collisions |

## Related Patterns

- [Performance Optimization](./performance-optimization.md) - Caching for performance
- [Repository Pattern](./repository-pattern.md) - Data access caching
- [Module Cache Guide](./module-cache-guide.md) - Module-specific caching