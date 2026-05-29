---
name: whmcs-gzip-compression
description: Gzip/Brotli compression for WHMCS
category: Performance & Monitoring
version: 1.0.0
---

# WHMCS Gzip/Brotli Compression Skill

## Overview
This skill provides patterns and implementations for configuring compression in WHMCS to reduce bandwidth and improve load times.

## Implementation Patterns

### Compression Manager
```php
<?php
/**
 * WHMCS Compression Configuration
 * Manages gzip/brotli compression
 */

namespace WHMCS\Module\Performance\Compression;

class CompressionManager {
    /**
     * Enable compression
     */
    public function enableCompression(array $params): array {
        $config = [
            'gzip_enabled' => $params['gzip'] ?? true,
            'brotli_enabled' => $params['brotli'] ?? true,
            'compression_level' => $params['level'] ?? 6, // 1-9 for gzip, 1-11 for brotli
            'min_size' => $params['min_size'] ?? 500 // bytes
        ];

        // Apply nginx configuration
        $nginxConfig = $this->generateNginxConfig($config);
        file_put_contents('/etc/nginx/conf.d/whmcs-compression.conf', $nginxConfig);

        // Apply PHP configuration
        ini_set('zlib.output_compression', 'On');
        ini_set('zlib.output_compression_level', $params['level'] ?? 6);

        return [
            'success' => true,
            'gzip_enabled' => $config['gzip_enabled'],
            'brotli_enabled' => $config['brotli_enabled']
        ];
    }

    /**
     * Generate nginx compression config
     */
    private function generateNginxConfig(array $config): string {
        $gzipConfig = $config['gzip_enabled'] ? 'on' : 'off';
        $brotliConfig = $config['brotli_enabled'] ? 'on' : 'off';

        return <<<NGINX
# Compression Configuration
gzip {$gzipConfig};
gzip_vary on;
gzip_proxied any;
gzip_comp_level {$config['compression_level']};
gzip_min_length {$config['min_size']};
gzip_types
    text/plain
    text/css
    text/javascript
    application/javascript
    application/json
    application/xml
    application/xml+rss
    image/svg+xml;

# Brotli (if available)
brotli {$brotliConfig};
brotli_comp_level {$config['compression_level']};
brotli_types
    text/plain
    text/css
    text/javascript
    application/javascript
    application/json;
NGINX;
    }
}
```

## Best Practices

1. **Appropriate Levels**: Level 6 is usually optimal
2. **Content Types**: Only compress text-based content
3. **Min Size**: Don't compress tiny files
4. **CDN Integration**: Enable compression at CDN edge
5. **Monitor Ratio**: Track compression ratio

## Related Skills

- whmcs-cdn-integration
- whmcs-resource-optimization
- whmcs-browser-caching
- whmcs-http2-push