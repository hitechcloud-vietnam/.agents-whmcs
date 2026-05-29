# WHMCS Product Bundles

## Overview

Product bundles in WHMCS allow you to group multiple products together for sale as a package. Bundles can include discounts and are commonly used for hosting packages with included add-ons.

## Bundle Configuration

### Create Bundle

**Products > Bundles > Create New Bundle**

```php
// Bundle configuration
[
    'name' => 'Business Hosting Bundle',
    'description' => 'Complete hosting solution',
    'display_style' => 'integrated',
    'bundle_discount' => 15              // 15% discount
]
```

### Bundle Components

```php
// Products in bundle
[
    'bundle_id' => 1,
    'products' => [
        ['product_id' => 1, 'name' => 'Web Hosting', 'qty' => 1],
        ['product_id' => 2, 'name' => 'SSL Certificate', 'qty' => 1],
        ['product_id' => 3, 'name' => 'Email Hosting', 'qty' => 1]
    ]
]
```

## Bundle Pricing

### Bundle Price Calculation

```php
// Calculate bundle price
[
    'individual_total' => 150.00,        // Sum of all products
    'bundle_discount' => 15,            // 15% off
    'bundle_price' => 127.50,           // Discounted total
    'savings' => 22.50
]
```

### Billing Cycles

```php
// Bundle pricing per cycle
[
    'monthly' => [
        'individual' => 150.00,
        'bundle' => 127.50,
        'savings' => 22.50
    ],
    'annually' => [
        'individual' => 1500.00,
        'bundle' => 1200.00,
        'savings' => 300.00
    ]
]
```

## Bundle Display

### Order Form Display

```php
// Show bundle as single item
[
    'display' => 'integrated',           // integrated, separate
    'show_products' => true,
    'show_savings' => true,
    'show_individual_prices' => true
]
```

### Bundle Presentation

```smarty
<!-- Bundle display -->
<h3>Business Hosting Bundle</h3>
<p>Complete hosting solution</p>

Included:
- Web Hosting ($50/mo)
- SSL Certificate ($30/mo)
- Email Hosting ($70/mo)

Total: $150/mo
Bundle Price: $127.50/mo
You Save: 15%
```

## Bundle Ordering

### Add Bundle to Cart

```php
// Bundle order
[
    'bundle_id' => 1,
    'billing_cycle' => 'monthly',
    'domain' => 'example.com',
    'config_options' => [...],
    'price' => 127.50
]
```

### Single Invoice

```php
// Bundle as single item
[
    'invoice_item' => [
        'description' => 'Business Hosting Bundle',
        'amount' => 127.50
    ]
]
```

## Bundle Modification

### Upgrade Bundle

```php
// Upgrade entire bundle
[
    'service_id' => 1,                  // Bundle service
    'upgrade_to_bundle' => 2,          // New bundle
    'reprice_all' => true
]
```

### Modify Component

```php
// Change one component
[
    'service_id' => 1,
    'component_product_id' => 2,
    'upgrade' => true,
    'recalculate_bundle' => true
]
```

## Bundle Components

### Individual Services

```php
// Each component creates service
[
    'bundle_service_id' => 100,
    'components' => [
        ['service_id' => 101, 'product_id' => 1, 'name' => 'Web Hosting'],
        ['service_id' => 102, 'product_id' => 2, 'name' => 'SSL Certificate'],
        ['service_id' => 103, 'product_id' => 3, 'name' => 'Email Hosting']
    ],
    'parent_service_id' => 100
]
```

## Bundle Cancellation

### Cancel Bundle

```php
// Cancel entire bundle
[
    'bundle_service_id' => 100,
    'action' => 'cancel_all_components',
    'cancel_type' => 'immediate',
    'terminate_all' => true
]
```

### Component Survival

```php
// Option to keep components
[
    'allow_keep_components' => true,
    'convert_to_standalone' => true,
    'reprice_individually' => true
]
```

## API Functions

```php
// Get bundle pricing
$result = localAPI('GetBundlePricing', [
    'bundleid' => 1
]);

// Order bundle
$result = localAPI('OrderBundle', [
    'bundleid' => 1,
    'domain' => 'example.com',
    'billingcycle' => 'monthly'
]);
```

## Best Practices

1. **Logical grouping**: Bundle related products
2. **Clear savings**: Show discount prominently
3. **Reasonable discounts**: Don't over-discount
4. **Easy upgrades**: Allow bundle upgrades
5. **Track bundles**: Monitor bundle performance

## Related Documentation

- [Product Addons](./whmcs-product-addons.md)
- [Configurable Options](./whmcs-product-configurable.md)
- [Product Pricing](./whmcs-product-pricing.md)
- [Service Creation](./whmcs-service-creation.md)