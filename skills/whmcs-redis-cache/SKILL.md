# WHMCS Redis Cache Integration Skill
# Version: 1.0 | Updated: 2026-05-28

## Purpose

Guide for implementing Redis caching in WHMCS modules.

## When to Use

- High-performance caching
- Session storage
- Distributed caching

## Redis Patterns

```php
<?php
class RedisCache {
    private Redis $redis;
    private string $prefix = 'whmcs_module_';

    public function __construct() {
        $this->redis = new Redis();
        $this->redis->connect(
            env('REDIS_HOST', '127.0.0.1'),
            env('REDIS_PORT', 6379)
        );
    }

    public function get(string $key, callable $callback = null, int $ttl = 300) {
        $value = $this->redis->get($this->prefix . $key);

        if ($value !== null) {
            return json_decode($value, true);
        }

        if ($callback) {
            $value = $callback();
            $this->set($key, $value, $ttl);
            return $value;
        }

        return null;
    }

    public function set(string $key, $value, int $ttl = 300): void {
        $this->redis->setex(
            $this->prefix . $key,
            $ttl,
            json_encode($value)
        );
    }

    public function delete(string $key): void {
        $this->redis->del($this->prefix . $key);
    }

    public function flush(): void {
        $keys = $this->redis->keys($this->prefix . '*');
        foreach ($keys as $key) {
            $this->redis->del($key);
        }
    }

    public function increment(string $key, int $value = 1): int {
        return $this->redis->incrby($this->prefix . $key, $value);
    }

    public function getMultiple(array $keys): array {
        $fullKeys = array_map(fn($k) => $this->prefix . $k, $keys);
        $values = $this->redis->mget($fullKeys);

 $result = [];
        foreach ($keys as $i => $key) {
            $result[$key] = $values[$i] ? json_decode($values[$i], true) : null;
        }

        return $result;
    }
}
```

### Dependency Check
```php
private function checkRedisExtension(): bool {
    return extension_loaded('redis');
}
```

---

**Related Skills:**
- whmcs-cache-manager
- whmcs-performance-optimization
