---
name: whmcs-code-splitting
description: JS/CSS code splitting for WHMCS
category: Performance & Monitoring
version: 1.0.0
---

# WHMCS Code Splitting Skill

## Overview
This skill provides patterns for code splitting JavaScript and CSS in WHMCS.

## Implementation Patterns

### Code Splitter
```php
<?php
/**
 * WHMCS Code Splitting
 * Splits JS/CSS for performance
 */

namespace WHMCS\Module\Performance\Bundling;

class CodeSplitter {
    /**
     * Split JavaScript bundles
     */
    public function splitJS(array $entries): array {
        $chunks = [];

        foreach ($entries as $entry) {
            $chunks[$entry] = $this->analyzeDependencies($entry);
        }

        return $chunks;
    }

    /**
     * Generate critical CSS
     */
    public function extractCriticalCSS(string $html): string {
        // Extract CSS needed for above-the-fold content
        return "/* Critical CSS extracted */";
    }
}
```

## Best Practices

1. **Route-Based Splitting**: Split by route/pages
2. **Lazy Loading**: Load non-critical chunks on demand
3. **Vendor Splitting**: Separate vendor code
4. **Critical CSS**: Inline critical CSS
5. **Preload**: Preload critical chunks

## Related Skills

- whmcs-lazy-loading
- whmcs-prefetching
- whmcs-resource-optimization
- whmcs-caching-strategies