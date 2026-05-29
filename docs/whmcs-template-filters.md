# WHMCS Template Filters

## Overview

Output filters in WHMCS/Smarty process template output before it's sent to the browser. They allow you to modify HTML, add content, or transform output globally.

## Registering Filters

### In PHP File

```php
<?php
// In includes/lib/template.php or custom module

$smarty = \WHMCS\View\Template::getSmarty();

$smarty->registerFilter("output", "myOutputFilter");
$smarty->registerFilter("prefilter", "myPrefilter");
$smarty->registerFilter("postfilter", "myPostfilter");
```

## Output Filters

### Basic Output Filter

```php
<?php
function addWatermark($output, $smarty)
{
    $watermark = '<div style="position:fixed;bottom:10px;right:10px;">My Watermark</div>';
    return $output . $watermark;
}
```

### Modify HTML Output

```php
<?php
function addCustomMeta($output, $smarty)
{
    $customMeta = '<meta name="custom" content="value">';
    
    // Add after <head> tag
    $output = preg_replace(
        '/<head[^>]*>/i',
        '$0' . $customMeta,
        $output,
        1
    );
    
    return $output;
}
```

### Add Analytics

```php
<?php
function injectAnalytics($output, $smarty)
{
    $trackingCode = '
    <script>
        // Your tracking code here
        gtag("config", "GA_TRACKING_ID");
    </script>';
    
    // Insert before </body>
    $output = str_replace(
        '</body>',
        $trackingCode . '</body>',
        $output
    );
    
    return $output;
}
```

## Prefilters

### Modify Template Source

```php
<?php
function convertShortcodes($templateSource, $smarty)
{
    // Convert [button]text[/button] to HTML
    $pattern = '/\[button\](.*?)\[\/button\]/i';
    $replacement = '<button class="btn">$1</button>';
    
    return preg_replace($pattern, $replacement, $templateSource);
}
```

### Add Debug Info

```php
<?php
function addDebugComments($templateSource, $smarty)
{
    if (defined('DEBUG_MODE') && DEBUG_MODE) {
        $debug = "<!-- Template: " . $smarty->template_resource . " -->";
        return $debug . $templateSource;
    }
    return $templateSource;
}
```

## Postfilters

### Process Compiled Templates

```php
<?php
function minifyOutput($compiled, $template, $smarty)
{
    // Simple minification
    $search = [
        '/\>[^\S]+/s',  // strip whitespace after tags
        '/[^\S]+\</s',  // strip whitespace before tags
        '/\s+/s',       // collapse whitespace
    ];
    
    $replace = ['>', '<', ' '];
    
    return preg_replace($search, $replace, $compiled);
}
```

## Common Use Cases

### Minification

```php
<?php
function minifyHtml($output, $smarty)
{
    $search = [
        '/\s{2,}/',           // Multiple whitespace
        '/\s*\n\s*/',         // Newlines
        '/\t/',               // Tabs
    ];
    
    $replace = [' ', '', ''];
    
    return preg_replace($search, $replace, $output);
}
```

### Lazy Loading Images

```php
<?php
function addLazyLoading($output, $smarty)
{
    // Add loading="lazy" to images
    $output = preg_replace(
        '/<img(?![^>]*loading=)([^>]*)>/i',
        '<img loading="lazy"$1>',
        $output
    );
    
    return $output;
}
```

### Add Schema Markup

```php
<?php
function addOrganizationSchema($output, $smarty)
{
    $schema = <<<JSON
<script type="application/ld+json">
{
    "@context": "https://schema.org",
    "@type": "Organization",
    "name": "Your Company",
    "url": "https://example.com"
}
</script>
JSON;
    
    return str_replace('</head>', $schema . '</head>', $output);
}
```

### Currency Formatting

```php
<?php
function formatCurrencyInHtml($output, $smarty)
{
    // Find prices and format them
    return preg_replace_callback(
        '/\$(\d+\.?\d*)/',
        function($matches) {
            return '$' . number_format($matches[1], 2);
        },
        $output
    );
}
```

## Filter Chaining

```php
<?php
// Register multiple filters
$smarty->registerFilter("output", "minifyHtml");
$smarty->registerFilter("output", "addAnalytics");
$smarty->registerFilter("output", "addLazyLoading");

// Filters run in order of registration
```

## Conditional Filters

```php
<?php
function conditionalAnalytics($output, $smarty)
{
    // Only add analytics on frontend
    if ($smarty->tpl_vars['inadmin'] ?? false) {
        return $output;
    }
    
    // Add tracking only on production
    if (defined('ENVIRONMENT') && ENVIRONMENT === 'production') {
        $analytics = '<script>...analytics code...</script>';
        $output = str_replace('</body>', $analytics . '</body>', $output);
    }
    
    return $output;
}
```

## WHMCS Integration

### Hook-Based Filter Registration

```php
<?php
add_hook('AfterOutputFilter', 1, function($vars) {
    // Add filters after WHMCS initialization
    $smarty = \WHMCS\View\Template::getSmarty();
    $smarty->registerFilter("output", "myFilter");
});
```

### Template-Specific Filters

```php
<?php
function applyTemplateFilters($output, $smarty)
{
    $template = $smarty->template_resource;
    
    if (strpos($template, 'checkout') !== false) {
        // Add checkout-specific modifications
    }
    
    if (strpos($template, 'invoice') !== false) {
        // Add invoice-specific modifications
    }
    
    return $output;
}
```

## Best Practices

1. **Performance**: Filters run on every page load - keep them efficient
2. **Caching**: Consider which filters need to run on cached pages
3. **Order**: Register filters in logical order
4. **Debugging**: Use output buffering for complex modifications

## See Also

- [Template Compiler](../whmcs-template-compiler.md)
- [Template Cache](../whmcs-template-cache.md)
- [Hook System](../whmcs-template-hooks.md)