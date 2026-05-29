---
name: whmcs-query-caching
description: Query cache for WHMCS
category: Performance & Monitoring
version: 1.0.0
---

# WHMCS Query Caching Skill

## Overview
This skill provides patterns for query result caching in WHMCS.

## Implementation Patterns

### Query Cache Manager
```php
<?php
/**
 * WHMCS Query Caching
 * Caches database query results
 */

namespace WHMCS\Module\Database;

class QueryCacheManager {
    private $cache;

    public function __construct() {
        $this->cache = new RedisCacheLayer();
    }

    /**
     * Execute cached query
     */
    public function cachedQuery(string $query, array $params = [], int $ttl = 300): array {
        $cacheKey = 'q_' . md5($query . serialize($params));

        $cached = $this->cache->get($cacheKey);
        if ($cached !== null) {
            return ['data' => $cached, 'cached' => true];
        }

        $result = \WHMCS\Database\Capsule::connection()->select($query, $params);

        $this->cache->set($cacheKey, $result, $ttl);

        return ['data' => $result, 'cached' => false];
    }

    /**
     * Invalidate query cache
     */
    public function invalidatePattern(string $pattern): int {
        // Pattern-based invalidation
        return 0;
    }
}
```

## Best Practices

1. **TTL Tuning**: Set appropriate cache TTLs
2. **Key Design**: Design clear cache keys
3. **Invalidation**: Implement proper invalidation
4. **Monitoring**: Track cache hit rates
5. **Size Limits**: Limit cached data size

## Related Skills

- whmcs-caching-strategies
- whmcs-database-indexing
- whmcs-slow-query
- whmcs-read-replica-config