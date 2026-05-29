# WHMCS API Response Caching

## Skill Description
Implement API response caching for WHMCS modules using various caching strategies to improve performance, reduce database load, and provide faster response times.

## Prerequisites
- WHMCS 7.0+ installation
- PHP 7.4+ with caching extension (Redis, Memcached, or file)
- Understanding of HTTP caching headers
- Basic performance optimization knowledge

## Step-by-Step Implementation

### 1. Cache Manager
```php
<?php
// includes/cache/CacheManager.php

namespace WHMCS\Module\YourModule\Cache;

class CacheManager
{
    private string $driver;
    private ?\Redis $redis = null;
    private ?\Memcached $memcached = null;
    private string $cacheDir;
    private array $config;

    public function __construct(array $config = [])
    {
        $this->config = $config;
        $this->driver = $config['driver'] ?? 'file';
        $this->cacheDir = $config['cache_dir'] ?? dirname(__DIR__, 3) . '/cache/api/';

        $this->initializeDriver();
    }

    private function initializeDriver(): void
    {
        if (!is_dir($this->cacheDir)) {
            mkdir($this->cacheDir, 0755, true);
        }

        switch ($this->driver) {
            case 'redis':
                $this->initializeRedis();
                break;
            case 'memcached':
                $this->initializeMemcached();
                break;
            case 'file':
            default:
                // File-based caching is default
                break;
        }
    }

    private function initializeRedis(): void
    {
        try {
            $this->redis = new \Redis();
            $this->redis->connect(
                $this->config['redis_host'] ?? '127.0.0.1',
                $this->config['redis_port'] ?? 6379
            );

            if (!empty($this->config['redis_password'])) {
                $this->redis->auth($this->config['redis_password']);
            }

            $this->redis->select($this->config['redis_db'] ?? 0);
        } catch (\Exception $e) {
            logActivity('Redis cache initialization failed: ' . $e->getMessage());
            $this->driver = 'file';
        }
    }

    private function initializeMemcached(): void
    {
        $this->memcached = new \Memcached();
        $this->memcached->addServer(
            $this->config['memcached_host'] ?? '127.0.0.1',
            $this->config['memcached_port'] ?? 11211
        );
    }

    public function get(string $key, callable $callback = null, int $ttl = 3600): mixed
    {
        $value = $this->getValue($key);

        if ($value !== null) {
            return $value;
        }

        if ($callback !== null) {
            $value = $callback();
            $this->set($key, $value, $ttl);
            return $value;
        }

        return null;
    }

    public function getValue(string $key): mixed
    {
        $cacheKey = $this->buildKey($key);

        switch ($this->driver) {
            case 'redis':
                return $this->getFromRedis($cacheKey);
            case 'memcached':
                return $this->getFromMemcached($cacheKey);
            case 'file':
            default:
                return $this->getFromFile($cacheKey);
        }
    }

    public function set(string $key, mixed $value, int $ttl = 3600): bool
    {
        $cacheKey = $this->buildKey($key);

        switch ($this->driver) {
            case 'redis':
                return $this->setToRedis($cacheKey, $value, $ttl);
            case 'memcached':
                return $this->setToMemcached($cacheKey, $value, $ttl);
            case 'file':
            default:
                return $this->setToFile($cacheKey, $value, $ttl);
        }
    }

    public function delete(string $key): bool
    {
        $cacheKey = $this->buildKey($key);

        switch ($this->driver) {
            case 'redis':
                return $this->redis->del($cacheKey) > 0;
            case 'memcached':
                return $this->memcached->delete($cacheKey);
            case 'file':
            default:
                $filePath = $this->cacheDir . $cacheKey . '.cache';
                return file_exists($filePath) && unlink($filePath);
        }
    }

    public function deletePattern(string $pattern): int
    {
        $count = 0;

        if ($this->driver === 'redis') {
            $keys = $this->redis->keys($this->buildKey($pattern));
            foreach ($keys as $key) {
                $this->redis->del($key);
                $count++;
            }
        } else {
            $files = glob($this->cacheDir . $this->buildKey($pattern) . '*.cache');
            foreach ($files as $file) {
                if (unlink($file)) {
                    $count++;
                }
            }
        }

        return $count;
    }

    public function clear(): bool
    {
        switch ($this->driver) {
            case 'redis':
                return $this->redis->flushDB();
            case 'memcached':
                return $this->memcached->flush();
            case 'file':
            default:
                $files = glob($this->cacheDir . '*.cache');
                foreach ($files as $file) {
                    unlink($file);
                }
                return true;
        }
    }

    public function exists(string $key): bool
    {
        return $this->getValue($key) !== null;
    }

    private function buildKey(string $key): string
    {
        return md5($key);
    }

    private function getFromRedis(string $key): mixed
    {
        $value = $this->redis->get($key);
        return $value !== false ? unserialize($value) : null;
    }

    private function getFromMemcached(string $key): mixed
    {
        $value = $this->memcached->get($key);
        return $value !== false ? $value : null;
    }

    private function getFromFile(string $key): mixed
    {
        $filePath = $this->cacheDir . $key . '.cache';

        if (!file_exists($filePath)) {
            return null;
        }

        $data = unserialize(file_get_contents($filePath));

        if ($data['expires_at'] < time()) {
            unlink($filePath);
            return null;
        }

        return $data['value'];
    }

    private function setToRedis(string $key, mixed $value, int $ttl): bool
    {
        return $this->redis->setex($key, $ttl, serialize($value));
    }

    private function setToMemcached(string $key, mixed $value, int $ttl): bool
    {
        return $this->memcached->set($key, $value, $ttl);
    }

    private function setToFile(string $key, mixed $value, int $ttl): bool
    {
        $filePath = $this->cacheDir . $key . '.cache';
        $data = [
            'value' => $value,
            'expires_at' => time() + $ttl,
            'created_at' => time()
        ];

        return file_put_contents($filePath, serialize($data), LOCK_EX) !== false;
    }
}
```

### 2. Response Cache Decorator
```php
<?php
// includes/cache/ResponseCache.php

namespace WHMCS\Module\YourModule\Cache;

class ResponseCache
{
    private CacheManager $cache;
    private array $defaultOptions = [
        'ttl' => 300, // 5 minutes
        'cache_empty' => false,
        'vary_by' => ['query', 'headers'],
        'key_prefix' => 'api_response'
    ];

    public function __construct(?CacheManager $cache = null)
    {
        $this->cache = $cache ?? new CacheManager();
    }

    public function cacheResponse(callable $callback, array $options = []): mixed
    {
        $options = array_merge($this->defaultOptions, $options);

        $cacheKey = $this->buildCacheKey($options);
        $cached = $this->cache->getValue($cacheKey);

        if ($cached !== null) {
            $this->addCacheHeaders($cached['ttl_remaining']);
            return $cached['data'];
        }

        // Execute callback
        $data = $callback();

        // Cache if not empty or cache_empty is true
        if ($data !== null || $options['cache_empty']) {
            $ttl = $options['ttl'];
            $this->cache->set($cacheKey, [
                'data' => $data,
                'cached_at' => time(),
                'ttl_remaining' => $ttl
            ], $ttl);

            $this->addCacheHeaders($ttl);
        }

        return $data;
    }

    public function cacheUserData(int $userId, callable $callback, int $ttl = 300): mixed
    {
        $cacheKey = "user:{$userId}";

        return $this->cache->get($cacheKey, $callback, $ttl);
    }

    public function cacheList(
        string $listType,
        array $filters,
        callable $callback,
        int $ttl = 300
    ): mixed {
        ksort($filters);
        $cacheKey = "list:{$listType}:" . md5(json_encode($filters));

        return $this->cache->get($cacheKey, $callback, $ttl);
    }

    public function invalidateUserCache(int $userId): void
    {
        $this->cache->deletePattern("user:{$userId}*");
    }

    public function invalidateListCache(string $listType): void
    {
        $this->cache->deletePattern("list:{$listType}:*");
    }

    public function invalidateAll(): void
    {
        $this->cache->clear();
    }

    private function buildCacheKey(array $options): string
    {
        $parts = [
            $options['key_prefix'] ?? 'api_response',
            $_SERVER['REQUEST_METHOD'] ?? 'GET',
            parse_url($_SERVER['REQUEST_URI'] ?? '/', PHP_URL_PATH)
        ];

        // Add query string hash if varying by query
        if (in_array('query', $options['vary_by'])) {
            $query = $_GET;
            unset($query['cache_bust']); // Remove cache bust param
            ksort($query);
            $parts[] = md5(json_encode($query));
        }

        return implode(':', $parts);
    }

    private function addCacheHeaders(int $ttl): void
    {
        header('Cache-Control: private, max-age=' . $ttl);
        header('X-Cache-TTL: ' . $ttl);
        header('X-Cache-Status: HIT');
    }
}
```

### 3. HTTP Cache Headers Trait
```php
<?php
// includes/cache/HttpCaching.php

namespace WHMCS\Module\YourModule\Traits;

trait HttpCaching
{
    private ?int $lastModified = null;
    private ?string $etag = null;
    private int $maxAge = 300;

    protected function setCacheHeaders(
        int $lastModified,
        ?string $etag = null,
        int $maxAge = 300
    ): void {
        $this->lastModified = $lastModified;
        $this->etag = $etag ?? $this->generateEtag($lastModified);
        $this->maxAge = $maxAge;

        header('Cache-Control: public, max-age=' . $this->maxAge);
        header('Last-Modified: ' . gmdate('D, d M Y H:i:s', $this->lastModified) . ' GMT');
        header('ETag: "' . $this->etag . '"');
    }

    protected function checkConditionalRequest(): ?array
    {
        $ifNoneMatch = $_SERVER['HTTP_IF_NONE_MATCH'] ?? null;
        $ifModifiedSince = $_SERVER['HTTP_IF_MODIFIED_SINCE'] ?? null;

        if ($ifNoneMatch) {
            // Remove quotes if present
            $ifNoneMatch = str_replace('"', '', $ifNoneMatch);

            if ($ifNoneMatch === $this->etag) {
                return ['status' => 304, 'reason' => 'ETag match'];
            }
        }

        if ($ifModifiedSince) {
            $modifiedSince = strtotime($ifModifiedSince);

            if ($modifiedSince && $this->lastModified && $modifiedSince >= $this->lastModified) {
                return ['status' => 304, 'reason' => 'Not modified since'];
            }
        }

        return null;
    }

    protected function sendNotModified(): void
    {
        http_response_code(304);
        header('Cache-Control: public, max-age=' . $this->maxAge);
        header('ETag: "' . $this->etag . '"');
        exit;
    }

    protected function setNoCache(): void
    {
        header('Cache-Control: no-cache, no-store, must-revalidate');
        header('Pragma: no-cache');
        header('Expires: 0');
    }

    protected function setPrivateCache(int $maxAge = 60): void
    {
        header('Cache-Control: private, max-age=' . $maxAge);
    }

    private function generateEtag(int $timestamp): string
    {
        return md5($timestamp . '-' . $_SERVER['REQUEST_URI'] ?? '');
    }
}
```

### 4. Usage Example
```php
<?php
// Example controller with caching

namespace WHMCS\Module\YourModule\Api\Controllers;

use WHMCS\Module\YourModule\Api\ApiController;
use WHMCS\Module\YourModule\Api\ApiResponse;
use WHMCS\Module\YourModule\Cache\ResponseCache;
use WHMCS\Module\YourModule\Traits\HttpCaching;

class InvoiceController extends ApiController
{
    use HttpCaching;

    private ResponseCache $cache;

    public function __construct()
    {
        parent::__construct();
        $this->cache = new ResponseCache();
    }

    public function index(): void
    {
        // Cache the list for 5 minutes
        $result = $this->cache->cacheResponse(function () {
            return $this->fetchInvoices();
        }, [
            'ttl' => 300,
            'key_prefix' => 'invoices:list',
            'vary_by' => ['query']
        ]);

        ApiResponse::success($result)->send();
    }

    public function show(): void
    {
        $invoiceId = $this->getRequiredParam('id');

        // Cache individual invoice for 10 minutes
        $result = $this->cache->cacheResponse(function () use ($invoiceId) {
            return $this->fetchInvoice($invoiceId);
        }, [
            'ttl' => 600,
            'key_prefix' => 'invoice:' . $invoiceId
        ]);

        if ($result === null) {
            ApiResponse::notFound('Invoice')->send();
            return;
        }

        ApiResponse::success($result)->send();
    }

    public function invalidateCache(): void
    {
        // Invalidate list cache when creating/updating invoices
        $this->cache->invalidateListCache('invoices');

        ApiResponse::success(['message' => 'Cache invalidated'])->send();
    }

    private function fetchInvoices(): array
    {
        // Database query here
        $query = \WHMCS\Billing\Invoice::query();
        $query->orderBy('created_at', 'desc');
        $query->limit(100);

        return $query->get()->toArray();
    }

    private function fetchInvoice(int $id): ?array
    {
        $invoice = \WHMCS\Billing\Invoice::find($id);
        return $invoice ? $invoice->toArray() : null;
    }
}
```

## Common Pitfalls and Solutions

| Pitfall | Solution |
|---------|----------|
| Stale data in cache | Implement cache invalidation on data changes |
| Cache stampede | Implement lock-based cache warming |
| Large cache keys | Use hash for long query strings |
| Memory pressure with file cache | Implement cache size limits and cleanup |
| Inconsistent cache across servers | Use Redis/Memcached for distributed caching |

## Security Considerations

1. **Never cache sensitive data** - Mark private data with Cache-Control: private
2. **Validate cache inputs** - Use hash for cache keys to prevent injection
3. **Set appropriate TTLs** - Don't cache indefinitely
4. **Clear cache on security events** - Clear all cache on password changes
5. **Encrypt cached sensitive data** - Consider encrypting sensitive cached data

## Testing Checklist

- [ ] Test cache hit scenario
- [ ] Test cache miss scenario
- [ ] Test cache invalidation
- [ ] Test cache TTL expiration
- [ ] Test conditional request (ETag)
- [ ] Test conditional request (Last-Modified)
- [ ] Test 304 Not Modified response
- [ ] Test no-cache headers
- [ ] Test cache key uniqueness
- [ ] Test cache warm-up

## Reference Links

- [HTTP Caching](https://developer.mozilla.org/en-US/docs/Web/HTTP/Caching)
- [Cache-Control Header](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Cache-Control)
- [RFC 7234 - HTTP/1.1 Caching](https://tools.ietf.org/html/rfc7234)
