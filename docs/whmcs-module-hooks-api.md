# WHMCS Module Hooks API

Complete reference for module-based hook implementation in WHMCS.

## Overview

Modules can define hooks that get triggered during WHMCS operations.

## Hook Implementation

### Module Hooks

```php
<?php
/**
 * Module Hooks
 * 
 * Define hooks within your module that can be dynamically enabled/disabled
 */

function yourmodule_registerHooks(): array
{
    return [
        'ClientAreaPrimaryNavbar' => [
            'function' => 'yourmodule_clientNavbarHook',
            'priority' => 100,
        ],
        'ClientAreaFooterOutput' => [
            'function' => 'yourmodule_footerHook',
            'priority' => 50,
        ],
        'InvoiceCreationPreMerge' => [
            'function' => 'yourmodule_invoiceHook',
            'priority' => 10,
        ],
    ];
}

/**
 * Client navbar hook
 */
function yourmodule_clientNavbarHook(array $params): array
{
    if (!$_SESSION['uid']) {
        return [];
    }
    
    return [
        'yourmodule' => [
            'name' => 'Your Module',
            'label' => 'Your Module',
            'uri' => 'addon.php?module=yourmodule',
        ],
    ];
}

/**
 * Footer hook for analytics
 */
function yourmodule_footerHook(array $params): string
{
    return '<script>console.log("Module footer hook")</script>';
}

/**
 * Invoice hook for modifications
 */
function yourmodule_invoiceHook(array $params): array
{
    // Modify invoice data before creation
    $params['items'][] = [
        'description' => 'Module added item',
        'amount' => 10.00,
    ];
    
    return $params;
}
```

## Hook Registration

```php
/**
 * Register module hooks dynamically
 */
function yourmodule_registerModuleHooks(): void
{
    $hooks = yourmodule_registerHooks();
    
    foreach ($hooks as $hookPoint => $hookConfig) {
        add_hook(
            $hookPoint,
            $hookConfig['priority'],
            $hookConfig['function']
        );
    }
}

// Call during module activation
yourmodule_registerModuleHooks();
```

## Common Hook Points

| Hook Point | Description |
|------------|-------------|
| ClientAreaPrimaryNavbar | Add client area menu items |
| ClientAreaFooterOutput | Add footer content |
| AdminAreaHeaderOutput | Add admin header content |
| InvoiceCreationPreMerge | Modify invoice before creation |
| AfterCronJob | After cron job execution |
| ServiceProvision | After service provisioning |
| TicketReply | After ticket reply |

## Related Documentation

- [whmcs-advanced-events.md](../advanced/whmcs-advanced-events.md)