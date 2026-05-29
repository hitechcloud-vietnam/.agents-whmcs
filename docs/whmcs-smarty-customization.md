# WHMCS Smarty Customization

## Overview

Smarty is WHMCS's templating engine, powering both the admin and client area interfaces. Customizing Smarty templates allows you to modify the look, feel, and functionality of WHMCS without modifying core files. This guide covers advanced Smarty customization techniques.

## Technical Details

### Smarty Version and Configuration

WHMCS uses Smarty 3.x with custom modifications. Key configuration files:

- `/includes/ Smarty/CompiledTemplates/` - Compiled template cache
- `/templates_c/` - WHMCS template compilation directory
- `/configuration.php` - Smarty settings

### Template Directory Structure

```
/whmcs/templates/
├── six/                    # Default template
├── five/                   # Legacy template
├── your_custom_template/   # Custom template
│   ├── header.tpl
│   ├── footer.tpl
│   ├── clientlayout.tpl
│   └── ...
```

## Configuration Options

### 1. Smarty Configuration Settings

```php
// In configuration.php or hooks
$smarty = new Smarty();

// Debugging (development only)
$smarty->debugging = false;
$smarty->force_compile = false;  // Set true during development

// Caching
$smarty->caching = true;
$smarty->cache_lifetime = 3600;

// Template security
$smarty->security = true;
$smarty->secure_dir = ['templates/', 'templates_c/'];
```

### 2. Custom Template Variables

```php
<?php
// hooks/smarty_variables.php

add_hook('AdminAreaHeaderOutput', 1, function($vars) {
    global $smarty;

    // Assign custom variables
    $smarty->assign('custom_brand_name', 'Your Company');
    $smarty->assign('custom_support_email', 'support@example.com');
    $smarty->assign('custom_features', [
        'feature1' => true,
        'feature2' => false,
        'feature3' => true
    ]);

    // Assign system information
    $smarty->assign('php_version', PHP_VERSION);
    $smarty->assign('whmcs_version', App::getVersion());
});
```

### 3. Smarty Plugins and Modifiers

```php
<?php
// Create custom Smarty modifier
// Save as: /includes/Smarty/plugins/modifier.your_modifier.php

<?php
/**
 * Custom Smarty modifier - format currency with symbol
 */
function smarty_modifier_your_currency($amount, $symbol = '$') {
    return $symbol . number_format($amount, 2);
}

// Usage in template: {$amount|your_currency:'€'}
```

## Code Examples

### Custom Template Hook

```php
<?php
// hooks/custom_template.php

add_hook('AdminAreaHeaderOutput', 1, function($vars) {
    global $smarty;

    // Add navigation items
    $smarty->assign('custom_nav_items', [
        ['label' => 'Dashboard', 'url' => 'dashboard.php', 'icon' => 'home'],
        ['label' => 'Reports', 'url' => 'reports.php', 'icon' => 'chart'],
        ['label' => 'Settings', 'url' => 'settings.php', 'icon' => 'gear']
    ]);
});

add_hook('ClientAreaHeaderOutput', 1, function($vars) {
    global $smarty;

    // Custom client notifications
    $smarty->assign('client_notifications', [
        'maintenance' => false,
        'new_features' => true,
        'promo_code' => 'SAVE20'
    ]);
});
```

### Template Override Example

```smarty
{* custom/templates/six/header.tpl *}

{extends file="parent:header.tpl"}

{block name="head"}
    {$smarty.block.parent}
    <link rel="stylesheet" href="{$BASE_PATH}templates/six/css/custom.css">
    <script src="{$BASE_PATH}templates/six/js/custom.js"></script>
{/block}

{block name="nav"}
    {include file="parent:nav.tpl"}
    {* Add custom navigation *}
    <li class="custom-nav">
        <a href="custom-page.php">
            <i class="fa fa-star"></i> Custom Link
        </a>
    </li>
{/block}
```

### Conditional Template Logic

```smarty
{* Product detail template *}

{if $product->isFeatured()}
    <div class="featured-product-badge">
        <i class="fa fa-star"></i> Featured
    </div>
{/if}

{if $loggedin && $client->hasCustomProperty('vip_status')}
    <div class="vip-indicator">
        VIP Customer - Priority Support
    </div>
{/if}

{foreach $product->getAddons() as $addon}
    <div class="addon-option {if $addon->isRecommended()}recommended{/if}">
        <h4>{$addon->getName()}</h4>
        <p class="price">{$addon->getMonthlyPrice()|currency}</p>
        {if $addon->hasDiscount()}
            <span class="discount">Save {$addon->getDiscountPercent()}%</span>
        {/if}
    </div>
{/foreach}
```

### Dynamic Content Loading

```smarty
{* AJAX content template *}

<div class="dynamic-section" data-url="ajax/product-info.php"
     data-product-id="{$product->getId()}">
    <div class="loading-indicator">
        <i class="fa fa-spinner fa-spin"></i> Loading...
    </div>
</div>

<script>
$(document).ready(function() {
    $('.dynamic-section').each(function() {
        var $el = $(this);
        $.ajax({
            url: $el.data('url'),
            data: { id: $el.data('product-id') },
            success: function(response) {
                $el.html(response);
            }
        });
    });
});
</script>
```

## Troubleshooting Tips

### Common Smarty Issues

| Issue | Cause | Solution |
|-------|-------|----------|
| Template not updating | Cached version | Enable `$smarty->force_compile` or clear `/templates_c/` |
| Variable undefined | Wrong scope | Check hook priority and variable assignment |
| Syntax errors | PHP in template | Use `{php}` sparingly or convert to plugin |
| Block inheritance fails | Parent template missing | Verify template hierarchy |

### Debug Smarty Variables

```smarty
{* Debug template - add to any template *}

<pre>
Variables:
{foreach from=$smarty.template Object->tpl_vars key=name item=var}
    {$name}: {var->value|var_dump}
{/foreach}
</pre>

{* Or use captured debug *}
{capture assign=debug_output}
    Variable count: {$variables|count}
    Template: {$smarty.template.template}
{/capture}
```

### Template Development Workflow

```php
<?php
// Development mode hook - /hooks/development_mode.php

add_hook('AdminAreaHeaderOutput', 1, function($vars) {
    if (isset($_GET['dev_mode'])) {
        global $smarty;
        $smarty->force_compile = true;
        $smarty->debugging = true;
    }
});

// Clear compiled templates
add_hook('ClearCache', 1, function() {
    $dir = ROOTDIR . '/templates_c/';
    $files = glob($dir . '*.php');
    foreach ($files as $file) {
        unlink($file);
    }
    return 'Templates cleared';
});
```

### Performance Optimization

```php
<?php
// Production optimization hook

add_hook('AdminAreaPage', 1, function($vars) {
    global $smarty;

    // Disable debugging in production
    $smarty->debugging = false;
    $smarty->force_compile = false;

    // Enable caching for complex templates
    $smarty->caching = true;
    $smarty->cache_lifetime = 7200;
});
```

### Template Security Best Practices

```php
<?php
// Secure template configuration

$smarty->security = true;
$smarty->secure_dir = [
    ROOTDIR . '/templates/',
    ROOTDIR . '/templates_c/'
];

// Disable PHP in templates
$smarty->php_handling = Smarty::PHP_REMOVE;

// Whitelist allowed PHP functions if needed
$smarty->security_settings = [
    'php_handling' => Smarty::PHP_REMOVE,
    'allow_super_globals' => false,
    'allow_constants' => true,
    'allow_vars' => true
];
```