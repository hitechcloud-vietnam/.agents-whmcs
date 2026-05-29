# WHMCS Template Hooks Workflow

## Purpose
Understand and implement WHMCS template hook system for customizing client area.

## Prerequisites
- WHMCS 7.x or higher
- Basic PHP and Smarty knowledge
- File access to WHMCS installation

## Available Template Hook Points

### Header/Footer Hooks
| Hook Name | Description |
|-----------|-------------|
| ClientAreaHeadOutput | HTML head section |
| ClientAreaHeaderOutput | After header |
| ClientAreaFooterOutput | Before footer |
| ClientAreaPageBodyClass | Add body classes |

### Content Hooks
| Hook Name | Description |
|-----------|-------------|
| ClientAreaHomepage | Homepage content |
| ClientAreaProductDetails | Product details page |
| ClientAreaSupportSubmitTicket | Ticket submission |
| ClientAreaCartCheckoutComplete | After checkout |

## Step-by-Step Process

### Step 1: Create Hook Directory
```
1. Navigate to /whmcs/includes/hooks/
2. Create custom hook files (e.g., template_customizations.php)
```

### Step 2: Header Output Hook
```php
<?php
/**
 * Add custom CSS/JS to head section
 */
add_hook('ClientAreaHeadOutput', 1, function($vars) {
    $customCss = '<style>
        .custom-header {
            background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
        }
        .custom-banner {
            padding: 40px 0;
            margin-bottom: 30px;
        }
    </style>';
    
    $customJs = '<script>
        document.addEventListener("DOMContentLoaded", function() {
            console.log("Custom template loaded");
        });
    </script>';
    
    return $customCss . $customJs;
});
```

### Step 3: Homepage Hook
```php
<?php
/**
 * Add content to homepage
 */
add_hook('ClientAreaHomepage', 1, function($vars) {
    return [
        'customSection' => [
            'title' => 'Welcome Message',
            'content' => '<div class="alert alert-info">Special offer for new customers!</div>',
            'order' => 10
        ]
    ];
});
```

### Step 4: Sidebar Hook
```php
<?php
/**
 * Add sidebar widgets
 */
add_hook('ClientAreaSidebarSecondarySidebar', 1, function($vars) {
    return [
        'templatefile' => 'custom_sidebar_widget',
        'vars' => [
            'message' => 'Custom sidebar content',
            'links' => [
                ['label' => 'Link 1', 'url' => 'link1.php'],
                ['label' => 'Link 2', 'url' => 'link2.php']
            ]
        ]
    ];
});
```

### Step 5: Product Details Hook
```php
<?php
/**
 * Add content to product details page
 */
add_hook('ClientAreaProductDetails', 1, function($vars) {
    $product = $vars['product'];
    
    return [
        [
            'title' => 'Custom Product Info',
            'content' => '<div class="custom-product-info">
                <p>SKU: ' . htmlspecialchars($product['sku'] ?? 'N/A') . '</p>
                <p>Stock: In Stock</p>
            </div>'
        ]
    ];
});
```

### Step 6: Cart Hook
```php
<?php
/**
 * Modify cart checkout
 */
add_hook('ClientAreaCartCheckoutComplete', 1, function($vars) {
    logActivity('Checkout completed - Order ID: ' . $vars['orderId']);
    
    // Send custom notification
    sendCustomNotification($vars);
});
```

### Step 7: Register Custom Template Hooks
```php
<?php
/**
 * Register custom hook points for templates
 */
add_hook('ClientAreaPageBodyClass', 1, function($vars) {
    return [
        'custom-class',
        'theme-' . $vars['template'],
        'page-' . $vars['filename']
    ];
});
```

## Template Usage

### Display Hook Output in Template
```smarty
{* In your template file *}
{if $hookresults.ClientAreaHomepage}
    {foreach from=$hookresults.ClientAreaHomepage item=hook}
        <div class="hook-content">
            {$hook.content}
        </div>
    {/foreach}
{/if}
```

### Sidebar Hook Display
```smarty
{if $sidebar}
    <div class="sidebar">
        {$sidebar}
    </div>
{/if}
```

## Advanced Hook Examples

### Multiple Hooks in One File
```php
<?php

// Hook 1: Head Output
add_hook('ClientAreaHeadOutput', 1, function($vars) {
    return '<link rel="stylesheet" href="custom.css">';
});

// Hook 2: Footer
add_hook('ClientAreaFooterOutput', 1, function($vars) {
    return '<script src="custom.js"></script>';
});

// Hook 3: Homepage
add_hook('ClientAreaHomepage', 1, function($vars) {
    return ['content' => '<p>Custom homepage content</p>'];
});
```

### Conditional Hooks
```php
<?php

add_hook('ClientAreaHeadOutput', 1, function($vars) {
    // Only on homepage
    if ($vars['filename'] === 'index') {
        return '<meta name="custom" content="homepage">';
    }
    
    // Only for logged-in users
    if ($vars['loggedin']) {
        return '<script>console.log("Welcome back!");</script>';
    }
    
    return '';
});
```

## Debugging Hooks

### Enable Debug Logging
Add to `/whmcs/configuration.php`:
```php
$debug = true;
```

### Check Hook Output
```php
<?php

add_hook('ClientAreaHeadOutput', 1, function($vars) {
    file_put_contents('/tmp/hook_debug.log', print_r($vars, true));
    return '';
});
```

## Best Practices
- Create separate files for different hook purposes
- Use meaningful file names (e.g., analytics_hooks.php)
- Keep hooks simple and focused
- Return arrays for structured data
- Handle errors gracefully
- Document your hooks
- Test thoroughly before production
