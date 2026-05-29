---
name: whmcs-resource-optimization
description: Resource minification for WHMCS
category: Performance & Monitoring
version: 1.0.0
---

# WHMCS Resource Optimization Skill

## Overview
This skill provides patterns for minifying and optimizing resources in WHMCS.

## Implementation Patterns

### Resource Optimizer
```php
<?php
/**
 * WHMCS Resource Optimization
 * Minifies and optimizes resources
 */

namespace WHMCS\Module\Performance\Optimization;

class ResourceOptimizer {
    /**
     * Minify CSS
     */
    public function minifyCSS(string $css): string {
        // Remove comments
        $css = preg_replace('/\/\*[\s\S]*?\*\//', '', $css);
        // Remove whitespace
        $css = preg_replace('/\s+/', ' ', $css);
        // Remove trailing semicolons before closing braces
        $css = preg_replace('/;\s*}/', '}', $css);

        return trim($css);
    }

    /**
     * Minify JavaScript
     */
    public function minifyJS(string $js): string {
        // Basic minification
        $js = preg_replace('/\/\*[\s\S]*?\*\//', '', $js);
        $js = preg_replace('/\/\/.*$/m', '', $js);
        $js = preg_replace('/\s+/', ' ', $js);
        $js = preg_replace('/\s*([{};,:])\s*/', '$1', $js);

        return trim($js);
    }
}
```

## Best Practices

1. **Minification**: Remove unnecessary characters
2. **Uglification**: Shorten variable names
3. **Tree Shaking**: Remove unused code
4. **Compression**: Enable gzip/brotli
5. **Caching**: Cache minified resources

## Related Skills

- whmcs-gzip-compression
- whmcs-code-splitting
- whmcs-caching-strategies
- whmcs-browser-caching