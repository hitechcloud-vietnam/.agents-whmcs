# WHMCS API Rate Limiter Storage

## Skill Description
Implement flexible rate limiting storage backends for WHMCS modules using Redis, Memcached, or file-based storage with support for distributed rate limiting.

## Prerequisites
- WHMCS 7.0+ installation
- PHP 7.4+ with Redis extension (optional)
- Redis or Memcached server (optional)
- Understanding of rate limiting algorithms

## Step-by-Step Implementation

### 1. Rate Limiter Storage Interface
```php
<?php
// includes/rate-limiter/storage/RateLimiterStorageInterface.php

namespace WHMCS\Module\YourModule\RateLimiter\Storage;

interface RateLimiterStorageInterface
{
    /**
     * Increment the counter for a key and return the new value
     */
    public function increment(string $key, int $ttl): int;

    /**
     * Get the current count for a key
     */
    public function get(string $key): int;

    /**
     * Check if the key exists
     */
    public function exists(string $key): bool;

    /**
     * Reset a key
     */
    public function reset(string $key): void;

    /**
     * Get the TTL for a key
     */
    public function getTtl(string $key): int;
}
```

### 2. Redis Storage
```php
<?php
// includes/rate-limiter/storage/RedisStorage.php

namespace WHMCS\Module\YourModule\RateLimiter\Storage;

class RedisStorage implements RateLimiterStorageInterface
{
    private \Redis $redis;
    private string $prefix;
    private bool $connected = false;

    public function __construct(array $config = [])
    {
        $this->prefix = $config['prefix'] ?? 'ratelimit:';

        $host = $config['host'] ?? '127.0.0.1';
        $port = $config['port'] ?? 6379;
        $password = $config['password'] ?? null;
        $database = $config['database'] ?? 0;

        try {
            $this->redis = new \Redis();
            $this->redis->connect($host, $port, 5);

            if ($password) {
                $this->redis->auth($password);
            }

            $this->redis->select($database);
            $this->connected = true;
        } catch (\Exception $e) {
            logActivity('Redis connection failed: ' . $e->getMessage());
            $this->connected = false;
        }
    }

    public function increment(string $key, int $ttl): int
    {
        if (!$this->connected) {
            return 0;
        }

        $fullKey = $this->prefix . $key;

        // Use MULTI/EXEC for atomic operation
        $this->redis->multi();
        $this->redis->incr($fullKey);
        $this->redis->ttl($fullKey);

        $results = $this->redis->exec();
        $count = $results[0] ?? 0;
        $currentTtl = $results[1] ?? -1;

        // Set expiry if this is a new key
        if ($currentTtl === -1 || $currentTtl === -2) {
            $this->redis->expire($fullKey, $ttl);
        }

        return (int) $count;
    }

    public function get(string $key): int
    {
        if (!$this->connected) {
            return 0;
        }

        $fullKey = $this->prefix . $key;
        $value = $this->redis->get($fullKey);

        return (int) ($value ?? 0);
    }

    public function exists(string $key): bool
    {
        if (!$this->connected) {
            return false;
        }

        return (bool) $this->redis->exists($this->prefix . $key);
    }

    public function reset(string $key): void
    {
        if (!$this->connected) {
            return;
        }

        $this->redis->del($this->prefix . $key);
    }

    public function getTtl(string $key): int
    {
        if (!$this->connected) {
            return -1;
        }

        return $this->redis->ttl($this->prefix . $key);
    }

    public function isConnected(): bool
    {
        return $this->connected;
    }

    public function getRateLimitInfo(string $key): array
    {
        $fullKey = $this->prefix . $key;
        $count = $this->get($key);
        $ttl = $this->getTtl($key);

        return [
            'key' => $key,
            'count' => $count,
            'ttl' => $ttl,
            'reset_at' => $ttl > 0 ? time() + $ttl : null
        ];
    }
}
```

### 3. Memcached Storage
```php
<?php
// includes/rate-limiter/storage/MemcachedStorage.php

namespace WHMCS\Module\YourModule\RateLimiter\Storage;

class MemcachedStorage implements RateLimiterStorageInterface
{
    private \Memcached $memcached;
    private string $prefix;
    private array $localCache = [];
    private bool $connected = false;

    public function __construct(array $config = [])
    {
        $this->prefix = $config['prefix'] ?? 'ratelimit:';

        $servers = $config['servers'] ?? [
            [$config['host'] ?? '127.0.0.1', $config['port'] ?? 11211]
        ];

        $this->memcached = new \Memcached();
        $this->memcached->addServers($servers);

        // Set options
        $this->memcached->setOption(\Memcached::OPT_BINARY_PROTOCOL, true);
        $this->memcached->setOption(\Memcached::OPT_TCP_NODELAY, true);

        // Test connection
        $this->memcached->getStats();
        $this->connected = $this->memcached->getResultCode() === \Memcached::RES_SUCCESS;
    }

    public function increment(string $key, int $ttl): int
    {
        if (!$this->connected) {
            return 0;
        }

        $fullKey = $this->prefix . $key;

        // Try to increment existing key
        $result = $this->memcached->increment($fullKey, 1);

        if ($result === false) {
            // Key doesn't exist, create it
            $this->memcached->set($fullKey, 1, $ttl);
            return 1;
        }

        return $result;
    }

    public function get(string $key): int
    {
        if (!$this->connected) {
            return $this->localCache[$this->prefix . $key] ?? 0;
        }

        $value = $this->memcached->get($this->prefix . $key);
        return (int) ($value ?? 0);
    }

    public function exists(string $key): bool
    {
        if (!$this->connected) {
            return isset($this->localCache[$this->prefix . $key]);
        }

        $this->memcached->get($this->prefix . $key);
        return $this->memcached->getResultCode() !== \Memcached::RES_NOTFOUND;
    }

    public function reset(string $key): void
    {
        if (!$this->connected) {
            unset($this->localCache[$this->prefix . $key]);
            return;
        }

        $this->memcached->delete($this->prefix . $key);
    }

    public function getTtl(string $key): int
    {
        // Memcached doesn't support TTL queries directly
        // Return a default value
        return -1;
    }
}
```

### 4. File-Based Storage
```php
<?php
// includes/rate-limiter/storage/FileStorage.php

namespace WHMCS\Module\YourModule\RateLimiter\Storage;

class FileStorage implements RateLimiterStorageInterface
{
    private string $cacheDir;
    private string $prefix;
    private array $locks = [];

    public function __construct(array $config = [])
    {
        $this->prefix = $config['prefix'] ?? 'ratelimit_';
        $this->cacheDir = $config['cache_dir'] ?? dirname(__DIR__, 3) . '/cache/rate_limiter/';

        if (!is_dir($this->cacheDir)) {
            mkdir($this->cacheDir, 0755, true);
        }
    }

    private function getFilePath(string $key): string
    {
        $hash = md5($key);
        return $this->cacheDir . $this->prefix . $hash . '.json';
    }

    public function increment(string $key, int $ttl): int
    {
        $filePath = $this->getFilePath($key);
        $lockPath = $filePath . '.lock';

        $this->acquireLock($lockPath);

        try {
            $data = $this->readData($filePath);
            $now = time();

            // Check if data has expired
            if ($data && isset($data['expires_at']) && $data['expires_at'] < $now) {
                $data = ['count' => 0, 'created_at' => $now, 'expires_at' => $now + $ttl];
            }

            if (!$data) {
                $data = ['count' => 0, 'created_at' => $now, 'expires_at' => $now + $ttl];
            }

            $data['count']++;
            $this->writeData($filePath, $data);

            return $data['count'];

        } finally {
            $this->releaseLock($lockPath);
        }
    }

    public function get(string $key): int
    {
        $filePath = $this->getFilePath($key);
        $data = $this->readData($filePath);

        if (!$data) {
            return 0;
        }

        // Check expiration
        if (isset($data['expires_at']) && $data['expires_at'] < time()) {
            $this->reset($key);
            return 0;
        }

        return $data['count'] ?? 0;
    }

    public function exists(string $key): bool
    {
        $filePath = $this->getFilePath($key);

        if (!file_exists($filePath)) {
            return false;
        }

        $data = $this->readData($filePath);

        if (!$data || !isset($data['expires_at'])) {
            return false;
        }

        return $data['expires_at'] >= time();
    }

    public function reset(string $key): void
    {
        $filePath = $this->getFilePath($key);

        if (file_exists($filePath)) {
            unlink($filePath);
        }
    }

    public function getTtl(string $key): int
    {
        $filePath = $this->getFilePath($key);
        $data = $this->readData($filePath);

        if (!$data || !isset($data['expires_at'])) {
            return -1;
        }

        $remaining = $data['expires_at'] - time();
        return max(0, $remaining);
    }

    private function readData(string $filePath): ?array
    {
        if (!file_exists($filePath)) {
            return null;
        }

        $content = file_get_contents($filePath);

        if ($content === false) {
            return null;
        }

        $data = json_decode($content, true);

        return is_array($data) ? $data : null;
    }

    private function writeData(string $filePath, array $data): void
    {
        file_put_contents($filePath, json_encode($data), LOCK_EX);
    }

    private function acquireLock(string $lockPath): void
    {
        $fp = fopen($lockPath, 'c');
        if (flock($fp, LOCK_EX)) {
            $this->locks[$lockPath] = $fp;
        }
    }

    private function releaseLock(string $lockPath): void
    {
        if (isset($this->locks[$lockPath])) {
            flock($this->locks[$lockPath], LOCK_UN);
            fclose($this->locks[$lockPath]);
            unset($this->locks[$lockPath]);

            if (file_exists($lockPath)) {
                unlink($lockPath);
            }
        }
    }

    public function cleanup(): int
    {
        $count = 0;
        $files = glob($this->cacheDir . $this->prefix . '*.json');

        foreach ($files as $file) {
            $data = $this->readData($file);

            if ($data && isset($data['expires_at']) && $data['expires_at'] < time()) {
                unlink($file);
                $count++;
            }
        }

        return $count;
    }
}
```

### 5. Rate Limiter Factory
```php
<?php
// includes/rate-limiter/RateLimiterFactory.php

namespace WHMCS\Module\YourModule\RateLimiter;

use WHMCS\Module\YourModule\RateLimiter\Storage\RedisStorage;
use WHMCS\Module\YourModule\RateLimiter\Storage\MemcachedStorage;
use WHMCS\Module\YourModule\RateLimiter\Storage\FileStorage;

class RateLimiterFactory
{
    public static function create(?string $type = null, array $config = []): RateLimiter
    {
        $type = $type ?? self::detectStorageType();

        $storage = match ($type) {
            'redis' => new RedisStorage($config),
            'memcached' => new MemcachedStorage($config),
            default => new FileStorage($config)
        };

        return new RateLimiter($storage, $config);
    }

    private static function detectStorageType(): string
    {
        $config = self::getConfig();

        if (!empty($config['redis_host'])) {
            return 'redis';
        }

        if (!empty($config['memcached_servers'])) {
            return 'memcached';
        }

        return 'file';
    }

    private static function getConfig(): array
    {
        // Load from module configuration
        $settings = getWHMCSModuleConfig('yourmodule');

        return [
            'redis_host' => $settings['redis_host'] ?? null,
            'redis_port' => $settings['redis_port'] ?? 6379,
            'redis_password' => $settings['redis_password'] ?? null,
            'memcached_servers' => $settings['memcached_servers'] ?? null,
            'cache_dir' => $settings['cache_dir'] ?? null
        ];
    }
}
```

### 6. Updated Rate Limiter with Storage
```php
<?php
// includes/rate-limiter/RateLimiter.php

namespace WHMCS\Module\YourModule\RateLimiter;

use WHMCS\Module\YourModule\RateLimiter\Storage\RateLimiterStorageInterface;

class RateLimiter
{
    private RateLimiterStorageInterface $storage;
    private int $maxRequests;
    private int $windowSeconds;
    private string $keyPrefix;

    public function __construct(
        RateLimiterStorageInterface $storage,
        array $config = []
    ) {
        $this->storage = $storage;
        $this->maxRequests = $config['max_requests'] ?? 60;
        $this->windowSeconds = $config['window_seconds'] ?? 60;
        $this->keyPrefix = $config['key_prefix'] ?? 'default';
    }

    public function attempt(string $identifier): bool
    {
        $key = $this->buildKey($identifier);
        $count = $this->storage->increment($key, $this->windowSeconds);

        return $count <= $this->maxRequests;
    }

    public function getRemainingAttempts(string $identifier): int
    {
        $key = $this->buildKey($identifier);
        $count = $this->storage->get($key);

        return max(0, $this->maxRequests - $count);
    }

    public function getRetryAfter(string $identifier): int
    {
        $key = $this->buildKey($identifier);
        $ttl = $this->storage->getTtl($key);

        return max(0, $ttl);
    }

    public function reset(string $identifier): void
    {
        $key = $this->buildKey($identifier);
        $this->storage->reset($key);
    }

    public function getHeaders(string $identifier): array
    {
        $remaining = $this->getRemainingAttempts($identifier);
        $retryAfter = $this->getRetryAfter($identifier);

        return [
            'X-RateLimit-Limit' => $this->maxRequests,
            'X-RateLimit-Remaining' => $remaining,
            'X-RateLimit-Reset' => time() + $retryAfter,
            'Retry-After' => $retryAfter
        ];
    }

    private function buildKey(string $identifier): string
    {
        return sprintf('%s:%s', $this->keyPrefix, $identifier);
    }
}
```

## Common Pitfalls and Solutions

| Pitfall | Solution |
|---------|----------|
| Redis connection failures | Implement fallback to file storage |
| Race conditions in file storage | Use file locking for atomic operations |
| Distributed rate limiting inconsistency | Use Redis with proper key distribution |
| Cache stampede | Implement lock-based cache warming |
| Storage connection timeouts | Set connection timeouts and retries |

## Security Considerations

1. **Secure Redis connection** - Use password authentication for Redis
2. **File permissions** - Set restrictive permissions on cache files
3. **Rate limiter bypass** - Ensure storage backend is consistent
4. **Key collision** - Use unique key prefixes per module
5. **Cleanup old data** - Implement automatic cleanup of expired entries

## Testing Checklist

- [ ] Test rate limiting with Redis storage
- [ ] Test rate limiting with Memcached storage
- [ ] Test rate limiting with file storage
- [ ] Test fallback when primary storage fails
- [ ] Test concurrent request handling
- [ ] Test rate limit reset functionality
- [ ] Test expired entries cleanup
- [ ] Test key collision prevention
- [ ] Test TTL propagation
- [ ] Test distributed rate limiting across servers

## Reference Links

- [Redis Rate Limiting Commands](https://redis.io/commands/incr/)
- [Memcached Protocol](https://github.com/memcached/memcached/blob/master/doc/protocol.txt)
- [PHP File Locking](https://www.php.net/manual/en/function.flock.php)
- [Sliding Window Rate Limiting](https://engineering.classdojo.com/sliding-window-rate-limiter/)
