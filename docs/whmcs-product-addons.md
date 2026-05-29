# WHMCS Product Add-ons

## Overview

Product add-ons in WHMCS are optional enhancements that can be added to a primary product. Add-ons provide flexibility for clients to customize their services.

## Add-on Configuration

### Create Add-on

**Products > Add-ons > Create New Add-on**

```php
// Add-on configuration
[
    'name' => 'Daily Backups',
    'description' => 'Automatic daily backups',
    'billing_cycle' => 'monthly',
    'monthly_price' => 5.00,
    'require_product' => 1                 // Required product ID
]
```

### Add-on Options

```php
// Configurable add-ons
[
    'addon_id' => 1,
    'options' => [
        ['name' => '10GB Storage', 'price' => 5.00],
        ['name' => '50GB Storage', 'price' => 15.00],
        ['name' => '100GB Storage', 'price' => 25.00]
    ],
    'selection_type' => 'dropdown'        // dropdown, checkbox, radio
]
```

## Add-on Pricing

### Pricing Structure

```php
// Add-on pricing
[
    'monthly' => 5.00,
    'quarterly' => 14.00,
    'annually' => 50.00,
    'biennially' => 90.00
]
```

### Pro-Rated Add-ons

```php
// Pro-rate on add
[
    'prorate_allowed' => true,
    'prorate_method' => 'daily'
]
```

## Add-on Assignment

### During Order

```php
// Select add-ons with order
[
    'product_id' => 1,
    'addons' => [
        ['addon_id' => 1, 'selected' => true],
        ['addon_id' => 2, 'selected' => false]
    ]
]
```

### After Order

```php
// Add add-on to existing service
[
    'service_id' => 1,
    'addon_id' => 1,
    'billing_cycle' => 'monthly',
    'prorate' => true,
    'charge_now' => 2.50                 // Prorated amount
]
```

## Add-on Management

### Add to Service

```php
// Add addon to active service
[
    'service_id' => 1,
    'addon_id' => 1,
    'qty' => 1,
    'billing_cycle' => 'monthly',
    'create_invoice' => true,
    'send_email' => true
]
```

### Remove from Service

```php
// Remove addon
[
    'service_id' => 1,
    'addon_id' => 1,
    'credit_remaining' => true,
    'effective_date' => 'immediate'
]
```

## Module Integration

### Add-on Commands

```php
// Module handles addon provisioning
[
    'module' => 'cpanel',
    'action' => 'create_addon',
    'service_id' => 1,
    'addon_name' => 'backup_10gb'
]
```

## Add-on Display

### Order Form

```php
// Display add-ons in order
[
    'show_with_product' => true,
    'show_description' => true,
    'show_pricing' => true,
    'allow_multiple' => true
]
```

### Client Area

```php
// Show in client service
[
    'service_id' => 1,
    'addons' => [
        ['name' => 'Daily Backups', 'price' => 5.00, 'status' => 'Active']
    ]
]
```

## Add-on Billing

### Separate Invoices

```php
// Add-on on own invoice
[
    'addon_id' => 1,
    'billing' => 'combined',            // combined, separate
    'separate_invoice' => true
]
```

### Combined with Service

```php
// Included in service invoice
[
    'addon_id' => 1,
    'billing' => 'combined',
    'line_item' => 'Daily Backups - $5.00'
]
```

## Add-on Upgrades

### Upgrade Add-on

```php
// Change addon option
[
    'service_id' => 1,
    'addon_id' => 1,
    'from_option' => '10gb',
    'to_option' => '50gb',
    'prorate_charge' => 3.33
]
```

## API Functions

```php
// Add addon to service
$result = localAPI('AddAddon', [
    'serviceid' => 1,
    'addonid' => 1
]);

// Remove addon
$result = localAPI('RemoveAddon', [
    'serviceid' => 1,
    'addonid' => 1
]);

// Get available addons
$result = localAPI('GetAddons', [
    'serviceid' => 1
]);
```

## Hooks

```php
// Hook: AddonAdded
add_hook('AddonAdded', 1, function($vars) {
    // $vars['serviceid']
    // $vars['addonid']
    // Provision addon, notify, etc.
});

// Hook: AddonRemoved
add_hook('AddonRemoved', 1, function($vars) {
    // $vars['serviceid']
    // $vars['addonid']
    // Remove addon, credit, etc.
});
```

## Best Practices

1. **Clear descriptions**: Explain add-on benefits
2. **Reasonable pricing**: Don't overcharge
3. **Logical bundles**: Offer related add-ons
4. **Easy to add/remove**: Simple management
5. **Module integration**: Sync with server

## Related Documentation

- [Product Bundles](./whmcs-product-bundles.md)
- [Configurable Options](./whmcs-product-configurable.md)
- [Service Modification](./whmcs-service-modification.md)
- [Product Configuration](./whmcs-product-configuration.md)