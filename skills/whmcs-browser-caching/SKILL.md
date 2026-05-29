---
name: whmcs-browser-caching
description: Cache-Control headers for WHMCS
category: Performance & Monitoring
version: 1.0.0
---

# WHMCS Browser Caching Skill

## Overview
This skill provides patterns for configuring browser caching in WHMCS.

## Implementation Patterns

### Browser Cache Manager
```php
<?php
/**
 * WHMCS Browser Caching
 * Manages browser cache headers
 */

namespace WHMCS\Module\Performance\Cache;

class BrowserCacheManager {
    /**
     * Set cache headers
     */
    public function setCacheHeaders(string $contentType, array $options = []): array {
        $ttl = $options['ttl'] ?? 86400; // 1 day default

        if ($options['immutable']) {
            $cacheControl = "public, max-age={$ttl}, immutable";
        } else {
            $cacheControl = "public, max-age={$ttl}";
        }

        if ($options['no_cache']) {
            $cacheControl = "no-cache, must-revalidate";
        }

        return [
            'Cache-Control' => $cacheControl,
            'Expires' => gmdate('D, d M Y H:i:s', time() + $ttl) . ' GMT',
            'ETag' => md5($contentType)
        ];
    }

    /**
     * Generate static asset caching
     */
    public function getStaticAssetHeaders(string $extension): array {
        $cacheRules = [
            'css' => ['ttl' => 604800, 'immutable' => true], // 1 week
            'js' => ['ttl' => 604800, 'immutable' => true],
            'png' => ['ttl' => 2592000, 'immutable' => true], // 1 month
            'jpg' => ['ttl' => 2592000, 'immutable' => true],
            'svg' => ['ttl' => 604800, 'immutable' => true],
            'woff2' => ['ttl' => 31536000, 'immutable' => true] // 1 year
        ];

        $rule = $cacheRules[$extension] ?? ['ttl' => 86400];

        return $this->setCacheHeaders($extension, $rule);
    }
}
```

## Best Practices

1. **Static Assets**: Long cache for static files
2. **Versioning**: Use content hashing for cache busting
3. **HTML**: Short or no cache for HTML
4. **Immutable**: Use immutable for versioned assets
5. **Compression**: Enable compression with caching

## Related Skills

- whmcs-caching-strategies
- whmcs-cdn-integration
- whmcs-gzip-compression
- whmcs-resource-optimization