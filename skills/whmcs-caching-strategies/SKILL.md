---
name: whmcs-caching-strategies
description: Multi-layer caching for WHMCS
category: Performance & Monitoring
version: 1.0.0
---

# WHMCS Caching Strategies Skill

## Overview
This skill provides patterns and implementations for multi-layer caching in WHMCS, including application caching, CDN, and database query caching.

## Implementation Patterns

### Cache Manager
```php
<?php
/**
 * WHMCS Multi-Layer Caching
 * Manages caching across layers
 */

namespace WHMCS\Module\Performance\Cache;

class CacheManager {
    private $db;
    private $layers = [];

    public function __construct() {
        $this->db = \WHMCS\Database\Capsule::connection();
        $this->initializeLayers();
    }

    private function initializeLayers(): void {
        $this->layers = [
            'memory' => new MemoryCacheLayer(),
            'redis' => new RedisCacheLayer(),
            'apc' => new APCuCacheLayer(),
            'file' => new FileCacheLayer()
        ];
    }

    /**
     * Set cache value
     */
    public function set(string $key, $value, array $options = []): void {
        $ttl = $options['ttl'] ?? 3600;
        $layers = $options['layers'] ?? ['redis'];

        foreach ($layers as $layer) {
            if (isset($this->layers[$layer])) {
                $this->layers[$layer]->set($key, $value, $ttl);
            }
        }

        // Log cache operation
        $this->logOperation('set', $key, $layers);
    }

    /**
     * Get cache value
     */
    public function get(string $key, array $options = []): ?mixed {
        $layers = $options['layers'] ?? ['redis', 'memory', 'apc'];

        foreach ($layers as $layer) {
            if (isset($this->layers[$layer])) {
                $value = $this->layers[$layer]->get($key);
                if ($value !== null) {
                    $this->logOperation('hit', $key, [$layer]);
                    return $value;
                }
            }
        }

        $this->logOperation('miss', $key, $layers);
        return null;
    }

    /**
     * Invalidate cache
     */
    public function invalidate(string $key, array $options = []): void {
        $layers = $options['layers'] ?? ['redis', 'memory', 'apc'];

        foreach ($layers as $layer) {
            if (isset($this->layers[$layer])) {
                $this->layers[$layer]->delete($key);
            }
        }
    }

    /**
     * Cache database query
     */
    public function cacheQuery(string $query, array $params = [], int $ttl = 300): array {
        $cacheKey = 'query_' . md5($query . serialize($params));

        $cached = $this->get($cacheKey);
        if ($cached !== null) {
            return $cached;
        }

        $result = $this->db->select($query, $params);

        $this->set($cacheKey, $result, ['ttl' => $ttl]);

        return $result;
    }

    /**
     * Warm cache
     */
    public function warmCache(array $keys): void {
        foreach ($keys as $key) {
            if ($data = $this->fetchForWarming($key)) {
                $this->set($key, $data, ['ttl' => 7200]);
            }
        }
    }
}

/**
 * Cache Layer Interfaces
 */
interface CacheLayerInterface {
    public function get(string $key): mixed;
    public function set(string $key, $value, int $ttl): void;
    public function delete(string $key): void;
}

class RedisCacheLayer implements CacheLayerInterface {
    private $redis;

    public function __construct() {
        $this->redis = new Redis();
        $this->redis->connect('127.0.0.1', 6379);
    }

    public function get(string $key): mixed {
        $value = $this->redis->get($key);
        return $value ? unserialize($value) : null;
    }

    public function set(string $key, $value, int $ttl): void {
        $this->redis->setex($key, $ttl, serialize($value));
    }

    public function delete(string $key): void {
        $this->redis->del($key);
    }
}
```

## Database Schema
```sql
CREATE TABLE `mod_cache_stats` (
  `id` INT AUTO_INCREMENT PRIMARY KEY,
  `cache_key` VARCHAR(255) NOT NULL,
  `layer` VARCHAR(50) NOT NULL,
  `operation` ENUM('set', 'get', 'hit', 'miss', 'invalidate') NOT NULL,
  `created_at` DATETIME NOT NULL,
  INDEX `idx_key` (`cache_key`)
);
```

## Cache Layers

| Layer | TTL | Use Case |
|-------|-----|----------|
| Memory | 30s | Hot data |
| Redis | 5min | Session, API cache |
| APCu | 1min | Opcode cache |
| File | 10min | Large data |

## Best Practices

1. **Cache Warming**: Pre-warm critical caches
2. **Invalidation Strategy**: Use cache tags for invalidation
3. **TTL Balance**: Balance freshness vs performance
4. **Layer Coordination**: Use consistent keys
5. **Monitoring**: Track hit/miss ratios

## Related Skills

- whmcs-redis-cache
- whmcs-database-caching
- whmcs-cdn-integration
- whmcs-browser-caching