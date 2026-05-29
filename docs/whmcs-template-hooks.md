# WHMCS Template Hooks

## Overview

Template hooks in WHMCS allow you to inject content at specific points in templates without modifying the core template files. They provide a powerful way to extend WHMCS functionality.

## Hook Points in Templates

### Client Area Hooks

```smarty
{hook point="ClientAreaPage"}
{hook point="ClientAreaPageHome"}
{hook point="ClientAreaPageServices"}
{hook point="ClientAreaPageDomains"}
{hook point="ClientAreaPageInvoices"}
{hook point="ClientAreaPageSupportTickets"}
{hook point="ClientAreaPageAddFunds"}
{hook point="ClientAreaPageCart"}
```

### Sidebar Hooks

```smarty
{hook point="ClientAreaSidebar"}
{hook point="ClientAreaSidebarHome"}
{hook point="ClientAreaSidebarServiceDetails"}
{hook point="ClientAreaSidebarDomainDetails"}
```

### Header/Footer Hooks

```smarty
{hook point="ClientAreaHeader"}
{hook point="ClientAreaFooter"}
{hook point="AdminAreaHeader"}
{hook point="AdminAreaFooter"}
```

### Page-Specific Hooks

```smarty
{hook point="ClientAreaPageViewInvoice"}
{hook point="ClientAreaPageViewTicket"}
{hook point="ClientAreaPageProductDetails"}
{hook point="ClientAreaPageDomainDetails"}
```

## Using Hook Points

### Basic Hook Usage

```smarty
{* In your template file *}
<div class="custom-content">
    {hook point="MyCustomHookPoint"}
</div>
```

### With Parameters

```smarty
{hook point="ProductSummary" product=$product}
```

## Creating Hook Files

### File Structure

```
templates/your-template/
    hooks/
        hook_ClientAreaPage.php
        hook_CustomFeature.php
```

### Hook File Format

```php
<?php
use WHMCS\View\Menu\Item as MenuItem;

add_hook('ClientAreaPage', 1, function(MenuItem $vars) {
    return [
        'custom_data' => 'value',
        'message' => 'Hello from hook'
    ];
});
```

### Template Integration Hook

```php
<?php
// File: templates/your-template/hooks/hook_CustomFeature.php

add_hook('ClientAreaPage', 1, function($vars) {
    return [
        'custom_variable' => 'value'
    ];
});
```

## Output Hooks

### Using {hook} in Templates

```smarty
{* Call a hook and display output *}
{hook point="CustomOutputHook"}
```

### Hook File Returning HTML

```php
<?php
add_hook('CustomOutputHook', 1, function($vars) {
    return '<div class="custom-module">Custom Content</div>';
});
```

### Hook Returning Data for Template

```php
<?php
add_hook('CustomDataHook', 1, function($vars) {
    return [
        'status' => 'active',
        'message' => 'Data loaded',
        'items' => [
            ['id' => 1, 'name' => 'Item 1'],
            ['id' => 2, 'name' => 'Item 2']
        ]
    ];
});
```

### Using Hook Data in Template

```smarty
{hook point="CustomDataHook" assign="customData"}

{if $customData.status}
    <div>{$customData.message}</div>
    {foreach $customData.items as $item}
        <li>{$item.name}</li>
    {/foreach}
{/if}
```

## Menu Hooks

### Client Area Menu

```php
<?php
add_hook('ClientAreaPrimarySidebar', 1, function(MenuItem $sidebar) {
    // Add new menu item
    $newItem = $sidebar->addChild('my-custom-item', [
        'label' => 'Custom Link',
        'uri' => 'custom.php',
        'icon' => 'fa-star'
    ]);
    
    // Add child item
    $newItem->addChild('sub-item', [
        'label' => 'Sub Item',
        'uri' => 'custom.php?action=sub'
    ]);
});
```

### Dynamic Menu Items

```php
<?php
add_hook('ClientAreaPrimarySidebar', 1, function(MenuItem $sidebar) {
    if (isset($_SESSION['uid'])) {
        $sidebar->addChild('user-quicklinks', [
            'label' => 'Quick Links',
            'icon' => 'fa-link',
            'order' => 5
        ])->addChild('new-ticket', [
            'label' => 'New Support Ticket',
            'uri' => 'supporttickets.php?action=open',
            'icon' => 'fa-ticket'
        ]);
    }
});
```

## CSS/JS Hooks

### Injecting Styles

```php
<?php
add_hook('ClientAreaHeaderOutput', 1, function($vars) {
    return '<link rel="stylesheet" href="templates/your-template/custom.css">';
});
```

### Injecting Scripts

```php
<?php
add_hook('ClientAreaFooterOutput', 1, function($vars) {
    return '<script src="templates/your-template/custom.js"></script>';
});
```

### Conditional Injection

```php
<?php
add_hook('ClientAreaHeaderOutput', 1, function($vars) {
    if ($vars['templatefile'] === 'productdetails') {
        return '<link rel="stylesheet" href="product-detail.css">';
    }
    return '';
});
```

## Page-Specific Hooks

### Invoice Page

```php
<?php
add_hook('ClientAreaPageViewInvoice', 1, function($vars) {
    return [
        'invoice_id' => $vars['invoice']->id,
        'show_qr_code' => true,
        'custom_message' => 'Thank you for your business!'
    ];
});
```

### Service Details

```php
<?php
add_hook('ClientAreaSidebarServiceDetails', 1, function(MenuItem $sidebar) {
    $service = Menu::context('service');
    
    if ($service) {
        $sidebar->addChild('service-monitor', [
            'label' => 'Uptime Monitor',
            'uri' => 'manage.php?service=' . $service->id,
            'icon' => 'fa-chart-line'
        ]);
    }
});
```

## Hook Priority

### Setting Priority

```php
<?php
// Lower number = higher priority (runs first)
add_hook('HookName', 1, function($vars) { /* ... */ });
add_hook('HookName', 5, function($vars) { /* ... */ });
add_hook('HookName', 10, function($vars) { /* ... */ });
```

### Multiple Hooks Same Point

```php
<?php
add_hook('ClientAreaPage', 1, function($vars) {
    // Runs first
});

add_hook('ClientAreaPage', 100, function($vars) {
    // Runs last
});
```

## Common Hook Points Reference

| Hook Point | Trigger | Typical Use |
|------------|---------|-------------|
| `ClientAreaPage` | Any client area page | Global modifications |
| `ClientAreaHeaderOutput` | In HTML head | CSS, meta tags |
| `ClientAreaFooterOutput` | Before </body> | JavaScript |
| `ClientAreaPrimarySidebar` | Main sidebar | Navigation items |
| `InvoiceDetailsPreTotal` | Invoice display | Add to invoice |
| `ServiceDetailsSidebar` | Service page sidebar | Quick actions |

## See Also

- [JavaScript Hooks](../whmcs-javascript-hooks.md)
- [Hook System Overview](../whmcs-hooks-reference.md)
- [Output Filters](../whmcs-template-filters.md)