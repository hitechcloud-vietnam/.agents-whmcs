# WHMCS Database Caching

## Skill Description
Implement database result caching for WHMCS modules to reduce database load, improve response times, and handle high-traffic scenarios efficiently.

## Prerequisites
- WHMCS 7.0+ installation
- PHP 7.4+
- Redis, Memcached, or file-based caching
- Understanding of cache invalidation strategies

## Step-by-Step Implementation

### 1. Database Cache
```php
<?php
// includes/database/DatabaseCache.php

namespace WHMCS\Module\YourModule\Database;

class DatabaseCache
{
    private string $driver;
    private ?\Redis $redis = null;
    private string $cacheDir;
    private array $config;

    public function __construct(array $config = [])
    {
        $this->config = $config;
        $this->driver = $config['driver'] ?? 'file';
        $this->cacheDir = $config['cache_dir'] ?? dirname(__DIR__) . '/cache/database/';

        if (!is_dir($this->cacheDir)) {
            mkdir($this->cacheDir, 0755, true);
        }

        $this->initializeDriver();
    }

    private function initializeDriver(): void
    {
        if ($this->driver === 'redis') {
            try {
                $this->redis = new \Redis();
                $this->redis->connect(
                    $this->config['redis_host'] ?? '127.0.0.1',
                    $this->config['redis_port'] ?? 6379
                );
            } catch (\Exception $e) {
                logActivity('Redis connection failed, falling back to file cache');
                $this->driver = 'file';
            }
        }
    }

    public function get(string $key, callable $callback = null, int $ttl = 300): mixed
    {
        $value = $this->retrieve($key);

        if ($value !== null) {
            return $value;
        }

        if ($callback !== null) {
            $value = $callback();
            $this->store($key, $value, $ttl);
            return $value;
        }

        return null;
    }

    public function remember(string $key, callable $callback, int $ttl = 300): mixed
    {
        return $this->get($key, $callback, $ttl);
    }

    public function rememberForever(string $key, callable $callback): mixed
    {
        return $this->get($key, $callback, 0);
    }

    public function put(string $key, mixed $value, int $ttl = 300): bool
    {
        return $this->store($key, $value, $ttl);
    }

    public function forget(string $key): bool
    {
        $cacheKey = $this->buildKey($key);

        if ($this->driver === 'redis') {
            return $this->redis->del($cacheKey) > 0;
        }

        $file = $this->cacheDir . $cacheKey . '.cache';
        return file_exists($file) && unlink($file);
    }

    public function flush(): bool
    {
        if ($this->driver === 'redis') {
            return $this->redis->flushDB();
        }

        $files = glob($this->cacheDir . '*.cache');
        foreach ($files as $file) {
            unlink($file);
        }

        return true;
    }

    public function has(string $key): bool
    {
        return $this->retrieve($key) !== null;
    }

    private function retrieve(string $key): mixed
    {
        $cacheKey = $this->buildKey($key);

        if ($this->driver === 'redis') {
            $value = $this->redis->get($cacheKey);
            return $value !== false ? unserialize($value) : null;
        }

        $file = $this->cacheDir . $cacheKey . '.cache';

        if (!file_exists($file)) {
            return null;
        }

        $data = unserialize(file_get_contents($file));

        // Check expiration
        if ($data['expires_at'] > 0 && $data['expires_at'] < time()) {
            unlink($file);
            return null;
        }

        return $data['value'];
    }

    private function store(string $key, mixed $value, int $ttl): bool
    {
        $cacheKey = $this->buildKey($key);
        $expiresAt = $ttl > 0 ? time() + $ttl : 0;

        if ($this->driver === 'redis') {
            return $this->redis->setex($cacheKey, $ttl, serialize([
                'value' => $value,
                'expires_at' => $expiresAt
            ]));
        }

        $data = [
            'value' => $value,
            'expires_at' => $expiresAt,
            'created_at' => time()
        ];

        $file = $this->cacheDir . $cacheKey . '.cache';
        return file_put_contents($file, serialize($data), LOCK_EX) !== false;
    }

    private function buildKey(string $key): string
    {
        return md5($key);
    }

    // Query caching methods
    public function query(string $sql, array $params = [], int $ttl = 300): array
    {
        $cacheKey = 'query:' . md5($sql . serialize($params));

        return $this->get($cacheKey, function () use ($sql, $params) {
            global $db;

            $db->query($sql, $params);
            return $db->fetchAll();
        }, $ttl);
    }

    public function queryOne(string $sql, array $params = [], int $ttl = 300): ?array
    {
        $cacheKey = 'query_one:' . md5($sql . serialize($params));

        $result = $this->get($cacheKey, function () use ($sql, $params) {
            global $db;

            $db->query($sql, $params);
            return $db->fetch();
        }, $ttl);

        return $result ?: null;
    }

    public function invalidateQueryCache(string $table): int
    {
        $pattern = 'query:*' . md5($table) . '*';
        return $this->deleteByPattern($pattern);
    }

    private function deleteByPattern(string $pattern): int
    {
        if ($this->driver === 'redis') {
            $keys = $this->redis->keys($this->buildKey($pattern));
            foreach ($keys as $key) {
                $this->redis->del($key);
            }
            return count($keys);
        }

        // File-based pattern matching
        $files = glob($this->cacheDir . '*.cache');
        $count = 0;

        foreach ($files as $file) {
            $data = unserialize(file_get_contents($file));
            if (strpos($data['value'] ?? '', $pattern) !== false) {
                unlink($file);
                $count++;
            }
        }

        return $count;
    }
}
```

### 2. Eloquent Cache Trait
```php
<?php
// includes/database/CachesQueries.php

namespace WHMCS\Module\YourModule\Traits;

trait CachesQueries
{
    private static ?DatabaseCache $queryCache = null;

    public static function getQueryCache(): DatabaseCache
    {
        if (self::$queryCache === null) {
            self::$queryCache = new DatabaseCache([
                'driver' => 'file',
                'cache_dir' => dirname(__DIR__) . '/cache/queries/'
            ]);
        }

        return self::$queryCache;
    }

    public static function setQueryCache(DatabaseCache $cache): void
    {
        self::$queryCache = $cache;
    }

    public static function cachedFind(int $id, int $ttl = 300): ?self
    {
        $cache = self::getQueryCache();
        $cacheKey = static::class . ':find:' . $id;

        $data = $cache->get($cacheKey, function () use ($id) {
            $instance = static::find($id);
            return $instance ? $instance->toArray() : null;
        }, $ttl);

        if ($data === null) {
            return null;
        }

        $instance = new static();
        foreach ($data as $key => $value) {
            $instance->$key = $value;
        }

        return $instance;
    }

    public static function cachedWhere(array $conditions, int $ttl = 300): array
    {
        $cache = self::getQueryCache();
        $cacheKey = static::class . ':where:' . md5(serialize($conditions));

        $ids = $cache->get($cacheKey, function () use ($conditions) {
            $results = static::where($conditions)->get();
            return array_column($results, 'id');
        }, $ttl);

        if (empty($ids)) {
            return [];
        }

        return static::findMany($ids);
    }

    public static function cachedAll(int $ttl = 300): array
    {
        $cache = self::getQueryCache();
        $cacheKey = static::class . ':all';

        $data = $cache->get($cacheKey, function () {
            $results = static::all();
            return $results->toArray();
        }, $ttl);

        $collection = [];
        foreach ($data as $item) {
            $instance = new static();
            foreach ($item as $key => $value) {
                $instance->$key = $value;
            }
            $collection[] = $instance;
        }

        return $collection;
    }

    public function invalidateCache(): void
    {
        $cache = self::getQueryCache();

        // Invalidate find cache
        $cache->forget(static::class . ':find:' . $this->id);

        // Invalidate all cache
        $cache->forget(static::class . ':all');

        // Invalidate table-related caches
        $cache->invalidateQueryCache($this->getTable());
    }

    public static function invalidateAllCache(): void
    {
        $cache = self::getQueryCache();
        $cache->flush();
    }
}
```

### 3. Cache Invalidation Manager
```php
<?php
// includes/database/CacheInvalidator.php

namespace WHMCS\Module\YourModule\Database;

class CacheInvalidator
{
    private DatabaseCache $cache;
    private array $relations = [];

    public function __construct(DatabaseCache $cache)
    {
        $this->cache = $cache;
    }

    public function registerRelation(string $model, array $relatedModels): void
    {
        $this->relations[$model] = $relatedModels;
    }

    public function invalidate(string $table, array $ids = []): void
    {
        // Invalidate table cache
        $this->cache->invalidateQueryCache($table);

        // Invalidate specific records
        foreach ($ids as $id) {
            $this->cache->forget("{$table}:{$id}");
        }

        // Invalidate related tables
        if (isset($this->relations[$table])) {
            foreach ($this->relations[$table] as $relatedTable) {
                $this->cache->invalidateQueryCache($relatedTable);
            }
        }

        // Log invalidation
        $this->logInvalidation($table, $ids);
    }

    public function invalidateUser(int $userId): void
    {
        $this->invalidate('tblclients', [$userId]);
        $this->invalidate('tblhosting', $this->getUserServices($userId));
        $this->invalidate('tblinvoices', $this->getUserInvoices($userId));
    }

    public function invalidateService(int $serviceId): void
    {
        $this->invalidate('tblhosting', [$serviceId]);

        $service = $this->getService($serviceId);
        if ($service) {
            $this->invalidateUser($service['userid']);
        }
    }

    public function invalidateInvoice(int $invoiceId): void
    {
        $this->invalidate('tblinvoices', [$invoiceId]);

        $invoice = $this->getInvoice($invoiceId);
        if ($invoice) {
            $this->invalidateUser($invoice['userid']);
        }
    }

    private function getUserServices(int $userId): array
    {
        global $db;

        $results = $db->select(
            "SELECT id FROM tblhosting WHERE userid = ?",
            [$userId]
        );

        return array_column($results, 'id');
    }

    private function getUserInvoices(int $userId): array
    {
        global $db;

        $results = $db->select(
            "SELECT id FROM tblinvoices WHERE userid = ?",
            [$userId]
        );

        return array_column($results, 'id');
    }

    private function getService(int $serviceId): ?array
    {
        global $db;

        $results = $db->select(
            "SELECT userid FROM tblhosting WHERE id = ?",
            [$serviceId]
        );

        return $results[0] ?? null;
    }

    private function getInvoice(int $invoiceId): ?array
    {
        global $db;

        $results = $db->select(
            "SELECT userid FROM tblinvoices WHERE id = ?",
            [$invoiceId]
        );

        return $results[0] ?? null;
    }

    private function logInvalidation(string $table, array $ids): void
    {
        global $db;

        $db->insert('mod_yourmodule_cache_log', [
            'table_name' => $table,
            'record_ids' => json_encode($ids),
            'created_at' => date('Y-m-d H:i:s')
        ]);
    }
}
```

### 4. Usage Examples
```php
<?php
// Example: Caching in a module service

namespace WHMCS\Module\YourModule\Services;

use WHMCS\Module\YourModule\Database\DatabaseCache;

class ClientService
{
    private DatabaseCache $cache;

    public function __construct()
    {
        $this->cache = new DatabaseCache([
            'driver' => 'file',
            'cache_dir' => dirname(__DIR__) . '/cache/clients/'
        ]);
    }

    public function getClient(int $clientId): ?array
    {
        $cacheKey = "client:{$clientId}";

        return $this->cache->remember($cacheKey, function () use ($clientId) {
            global $db;

            $result = $db->select(
                "SELECT * FROM tblclients WHERE id = ?",
                [$clientId]
            );

            return $result[0] ?? null;
        }, 600); // 10 minute cache
    }

    public function getClientServices(int $clientId): array
    {
        $cacheKey = "client:{$clientId}:services";

        return $this->cache->remember($cacheKey, function () use ($clientId) {
            global $db;

            return $db->select(
                "SELECT h.*, p.name as product_name
                 FROM tblhosting h
                 JOIN tblproducts p ON h.packageid = p.id
                 WHERE h.userid = ?
                 ORDER BY h.id DESC",
                [$clientId]
            );
        }, 300); // 5 minute cache
    }

    public function updateClient(int $clientId, array $data): bool
    {
        global $db;

        $result = $db->update('tblclients', $data, 'id = ?', [$clientId]);

        if ($result) {
            // Invalidate cache
            $this->cache->forget("client:{$clientId}");
            $this->cache->forget("client:{$clientId}:services");
        }

        return $result > 0;
    }

    public function getActiveClientsCount(): int
    {
        $cacheKey = 'stats:active_clients';

        return $this->cache->remember($cacheKey, function () {
            global $db;

            $result = $db->select(
                "SELECT COUNT(*) as cnt FROM tblclients WHERE status = 'Active'"
            );

            return (int) ($result[0]['cnt'] ?? 0);
        }, 3600); // 1 hour cache for statistics
    }
}
```

## Common Pitfalls and Solutions

| Pitfall | Solution |
|---------|----------|
| Stale data | Implement proper cache invalidation on updates |
| Cache stampede | Use locking to prevent simultaneous cache rebuilds |
| Memory pressure | Set reasonable TTLs and cleanup policies |
| Inconsistent caching | Use cache versioning |
| Cache explosion | Limit maximum cache entries |

## Security Considerations

1. **Don't cache sensitive data** - Exclude passwords, tokens, PII
2. **Encrypt cached data** - Consider encryption for sensitive cache entries
3. **Secure cache storage** - Set appropriate file permissions
4. **Clear cache on security events** - Password changes, etc.
5. **Validate cache keys** - Prevent cache poisoning

## Testing Checklist

- [ ] Test cache retrieval
- [ ] Test cache storage
- [ ] Test cache expiration
- [ ] Test cache invalidation
- [ ] Test query caching
- [ ] Test stampede prevention
- [ ] Test driver fallback
- [ ] Test cache statistics

## Reference Links

- [Cache Stampede Prevention](https://redis.io/topics/lru-cache)
- [Cache Invalidation Patterns](https://martinfowler.com/articles/patterns-of-distributed-systems/caching.html)
- [Laravel Cache](https://laravel.com/docs/cache)
