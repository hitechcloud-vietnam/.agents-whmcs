# WHMCS Product Configurable Options

## Overview

Configurable options in WHMCS allow clients to customize products by selecting from predefined choices during order or modifying existing services.

## Option Types

### Dropdown

```php
// Single selection dropdown
[
    'type' => 'dropdown',
    'name' => 'Storage Size',
    'options' => [
        ['value' => '10gb', 'name' => '10GB', 'price' => 0],
        ['value' => '50gb', 'name' => '50GB', 'price' => 5.00],
        ['value' => '100gb', 'name' => '100GB', 'price' => 10.00]
    ]
]
```

### Checkbox

```php
// Yes/No option
[
    'type' => 'checkbox',
    'name' => 'Daily Backups',
    'description' => 'Automatic daily backups',
    'price' => 5.00,
    'checked_by_default' => false
]
```

### Quantity

```php
// Numeric quantity
[
    'type' => 'quantity',
    'name' => 'Additional Emails',
    'min' => 0,
    'max' => 100,
    'step' => 1,
    'price_per_unit' => 1.00
]
```

### Text Field

```php
// Free text input
[
    'type' => 'text',
    'name' => 'Domain Prefix',
    'placeholder' => 'www',
    'max_length' => 50,
    'price' => 0
]
```

### Radio Buttons

```php
// Single selection radio
[
    'type' => 'radio',
    'name' => 'Control Panel',
    'options' => [
        ['value' => 'cpanel', 'name' => 'cPanel', 'price' => 0],
        ['value' => 'plesk', 'name' => 'Plesk', 'price' => 2.00]
    ]
]
```

## Option Configuration

### Create Option Group

**Products > Configurable Options > Create New Group**

```php
// Option group
[
    'name' => 'Hosting Options',
    'description' => 'Customize your hosting package',
    'display_order' => 1,
    'hidden' => false
]
```

### Assign to Product

```php
// Link options to product
[
    'product_id' => 1,
    'option_groups' => [1, 2, 3],
    'required_groups' => [1],           // Must select
    'show_on_order' => true,
    'show_on_service' => true
]
```

## Option Pricing

### Pricing Models

```php
// Per-option pricing
[
    'options' => [
        ['value' => 'basic', 'name' => 'Basic', 'monthly' => 0, 'annually' => 0],
        ['value' => 'standard', 'name' => 'Standard', 'monthly' => 5, 'annually' => 50],
        ['value' => 'premium', 'name' => 'Premium', 'monthly' => 10, 'annually' => 100]
    ]
]
```

### Recurring vs One-Time

```php
// Price types
[
    'price_type' => 'recurring',       // recurring, onetime
    'setup_fee' => 10.00,               // One-time setup
    'monthly' => 5.00                    // Recurring
]
```

## Selection on Order

### Client Selection

```php
// Order form options
[
    'group_id' => 1,
    'options' => [
        ['id' => 1, 'name' => '10GB', 'selected' => false],
        ['id' => 2, 'name' => '50GB', 'selected' => true],
        ['id' => 3, 'name' => '100GB', 'selected' => false]
    ]
]
```

### Required Options

```php
// Must select
[
    'group_id' => 1,
    'required' => true,
    'default_option' => '10gb'
]
```

## Modify Options

### Change on Service

```php
// Modify selected options
[
    'service_id' => 1,
    'option_changes' => [
        ['group_id' => 1, 'option_id' => 2],
        ['group_id' => 2, 'option_id' => 5]
    ],
    'charge_prorate' => true
]
```

### Upgrade/Downgrade

```php
// Change option pricing
[
    'service_id' => 1,
    'group_id' => 1,
    'from_option' => '10gb',
    'to_option' => '50gb',
    'prorate_credit' => 0.00,
    'new_monthly_price' => 5.00
]
```

## Module Integration

### Pass to Module

```php
// Send options to provisioning
[
    'module' => 'cpanel',
    'action' => 'create',
    'config_options' => [
        'disk' => '50gb',
        'bandwidth' => '100gb',
        'backups' => 'daily'
    ]
]
```

## Display Options

### Order Form Display

```smarty
<!-- Configurable option in order -->
<div class="config-option">
    <label>Storage Size</label>
    <select name="configoption[1]">
        <option value="10gb">10GB - Included</option>
        <option value="50gb">50GB - $5.00/mo</option>
        <option value="100gb">100GB - $10.00/mo</option>
    </select>
</div>
```

### Invoice Display

```
+--------------------------------------------------+
| Configurable Options                             |
+--------------------------------------------------+
| Storage Size: 50GB                    | $5.00    |
| Daily Backups                         | $2.00    |
+--------------------------------------------------+
```

## API Functions

```php
// Get configurable options
$result = localAPI('GetConfigurableOptions', [
    'productid' => 1
]);

// Update service options
$result = localAPI('UpdateServiceConfigOptions', [
    'serviceid' => 1,
    'configoptions' => [
        1 => '50gb',
        2 => 1
    ]
]);
```

## Hooks

```php
// Hook: ConfigOptionsUpdated
add_hook('ConfigOptionsUpdated', 1, function($vars) {
    // $vars['serviceid']
    // $vars['configoptions']
    // Update module, notify, etc.
});
```

## Best Practices

1. **Clear labels**: Use descriptive option names
2. **Logical grouping**: Group related options
3. **Reasonable pricing**: Don't overcharge options
4. **Required vs optional**: Mark required options
5. **Test thoroughly**: Verify module receives options

## Related Documentation

- [Product Configuration](./whmcs-product-configuration.md)
- [Product Addons](./whmcs-product-addons.md)
- [Service Modification](./whmcs-service-modification.md)
- [Product Pricing](./whmcs-product-pricing.md)