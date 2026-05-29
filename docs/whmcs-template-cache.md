# WHMCS Template Caching

## Overview

Template caching in WHMCS improves performance by storing compiled templates and rendered output. Understanding caching helps balance performance with freshness of content.

## Caching Modes

### Compiled Templates Only

Templates are compiled to PHP but not cached.

```php
<?php
$smarty->setCaching(\Smarty\Smarty::CACHING_OFF);
$smarty->setForceCompile(true);
```

### Cached Templates

Both compiled templates and output are cached.

```php
<?php
$smarty->setCaching(\Smarty\Smarty::CACHING_LIFETIME_CURRENT);
$smarty->setCacheLifetime(3600); // 1 hour
```

## Cache Lifetime

### Global Setting

```php
<?php
$smarty->setCacheLifetime(3600); // 1 hour in seconds
```

### Per-Template Setting

```php
<?php
// In template file
{cache ttl=600}
    {$expensive_query_result}
{/cache}
```

### Lifetime from Variables

```php
<?php
$smarty->setCacheLifetime($userCacheLifetime);
```

## Cache Groups

### Using cache_id

```php
<?php
// Different cache for each user
$cache_id = $userId;
$smarty->display('template.tpl', $cache_id);

// Different cache for each page
$cache_id = $pageName . '_' . $categoryId;
$smarty->display('template.tpl', $cache_id);
```

### Cache with Multiple Identifiers

```php
<?php
// Combine multiple factors
$cache_id = $pageType . '|' . $categoryId . '|' . $countryCode;
$smarty->display('template.tpl', $cache_id);
```

## isCached

### Check if Cached

```php
<?php
if ($smarty->isCached('template.tpl', $cacheId)) {
    // Use cached version
    $smarty->display('template.tpl', $cacheId);
} else {
    // Fetch fresh data
    $data = fetchData();
    $smarty->assign('data', $data);
    $smarty->display('template.tpl', $cacheId);
}
```

### Conditional Loading

```php
<?php
$smarty->setCaching(\Smarty\Smarty::CACHING_LIFETIME_CURRENT);

if (!$smarty->isCached('product.tpl', $productId)) {
    // Expensive database query
    $product = Product::find($productId);
    $smarty->assign('product', $product);
}

$smarty->display('product.tpl', $productId);
```

## Clear Cache

### Clear Single Template

```php
<?php
$smarty->clearCache('template.tpl');
$smarty->clearCache('template.tpl', $cacheId);
```

### Clear All Cache

```php
<?php
$smarty->clearAllCache();
```

### Clear by Lifetime

```php
<?php
// Clear cache older than 1 hour
$smarty->clearCache(null, null, null, 3600);
```

## Cache Handler

### Custom Cache Backend

```php
<?php
use WHMCS\Session\Cache as SessionCache;

// Using WHMCS built-in cache
$cache = SessionCache::getInstance();

// Set cache
$cache->set('key', $value, 3600);

// Get cache
$value = $cache->get('key');

// Check if exists
if ($cache->has('key')) {
    // ...
}
```

### Redis/Memcached Handler

```php
<?php
// In configuration
$smarty->setCacheHandler('memcached', [
    'host' => 'localhost',
    'port' => 11211,
    'priority' => 0
]);
```

## Excluding Content from Cache

### nocache Modifier

```smarty
{cache ttl=3600}
    Static content here: {$static_var}
    
    User-specific content: {$user->name nocache}
{/cache}
```

### Dynamic Blocks

```smarty
{cached var="section1"}
    Static content
{/cached}

<div class="dynamic">
    {$always_fresh_data nocache}
</div>
```

## Caching Strategies

### Strategy 1: Full Page Cache

```php
<?php
// Cache entire page for anonymous users
if (!isset($_SESSION['uid'])) {
    $smarty->setCaching(\Smarty\Smarty::CACHING_LIFETIME_SAVED);
    $smarty->setCacheLifetime(300); // 5 minutes
    
    $cacheId = md5($_SERVER['REQUEST_URI']);
    
    if ($smarty->isCached('page.tpl', $cacheId)) {
        $smarty->display('page.tpl', $cacheId);
        exit;
    }
}
```

### Strategy 2: Fragment Cache

```php
<?php
// Cache expensive queries
if (!$smarty->isCached('product_list.tpl', $categoryId)) {
    $products = Product::where('category', $categoryId)->get();
    $smarty->assign('products', $products);
}
$smarty->display('product_list.tpl', $categoryId);
```

### Strategy 3: User-Specific Cache

```php
<?php
// Cache per user preferences
$cacheId = 'user_' . ($userId ?? 'guest') . '_' . $template;

$smarty->display('dashboard.tpl', $cacheId);
```

## Performance Tips

### Do

- Enable caching in production
- Use appropriate cache lifetimes
- Clear cache on data updates
- Use cache groups for related content

### Don't

- Cache user-specific data without proper IDs
- Set extremely long cache lifetimes without versioning
- Cache authenticated user pages

## WHMCS Cache Control

### Clear WHMCS Cache

```php
<?php
// Via API or Admin
run_hook('ClearCache');

// Manual clearing
$whmcs = App::self();
$whmcs->getConfig('_clearCache')();
```

### Hook to Clear Template Cache

```php
<?php
add_hook('ProductEdit', 1, function($params) {
    $smarty = \WHMCS\View\Template::getSmarty();
    $smarty->clearCache('product.tpl', $params['pid']);
});
```

## Debugging Cache Issues

### Check Cache Status

```php
<?php
if ($smarty->isCached('template.tpl', $cacheId)) {
    echo "<!-- Cached version -->";
} else {
    echo "<!-- Fresh render -->";
}
```

### Development Mode

```php
<?php
// Always fresh in development
if (defined('DEV_MODE') && DEV_MODE) {
    $smarty->setForceCompile(true);
    $smarty->setCaching(\Smarty\Smarty::CACHING_OFF);
}
```

## See Also

- [Template Compiler](../whmcs-template-compiler.md)
- [Template Filters](../whmcs-template-filters.md)
- [Smarty Caching](https://www.smarty.net/docs/en/caching.tpl)