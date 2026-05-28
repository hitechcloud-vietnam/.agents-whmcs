# WHMCS Cache Manager Skill
# Version: 1.0 | Updated: 2026-05-28

## Purpose

Guide for implementing caching strategies in WHMCS modules.

## When to Use

- Optimizing module performance
- Reducing API calls
- Implementing temporary storage

## Cache Patterns

```php
<?php
class CacheManager {
    private string $prefix = 'mod_{module}_';

    public function get(string $key, callable $callback = null, int $ttl = 3600) {
        $cacheKey = $this->prefix . $key;
        $cached = Capsule::table('mod_cache')
            ->where('cache_key', $cacheKey)
            ->where('expires_at', '>', date('Y-m-d H:i:s'))
            ->first();

        if ($cached) {
            return json_decode($cached->cache_value, true);
        }

        if ($callback) {
            $value = $callback();
            $this->set($key, $value, $ttl);
            return $value;
        }

        return null;
    }

    public function set(string $key, $value, int $ttl = 3600): void {
        Capsule::table('mod_cache')->updateOrInsert(
            ['cache_key' => $this->prefix . $key],
            [
                'cache_value' => json_encode($value),
                'expires_at' => date('Y-m-d H:i:s', strtotime("+{$ttl} seconds")),
            ]
        );
    }

    public function delete(string $key): void {
        Capsule::table('mod_cache')
            ->where('cache_key', $this->prefix . $key)
            ->delete();
    }

    public function clear(): void {
        Capsule::table('mod_cache')
            ->where('cache_key', 'like', $this->prefix . '%')
            ->delete();
    }
}
```

---

**Related Skills:**
- whmcs-performance-optimization
- whmcs-database-design
- whmcs-cron-automation
