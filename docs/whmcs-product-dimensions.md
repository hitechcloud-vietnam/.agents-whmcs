# WHMCS Product Dimensions

## Overview

Product dimensions in WHMCS define billing-related characteristics of products, including billing cycles, trial periods, and automatic renewal settings.

## Billing Dimensions

### Cycle Configuration

```php
// Define billing cycles
[
    'product_id' => 1,
    'cycles' => [
        'monthly' => ['days' => 30, 'label' => 'Monthly'],
        'quarterly' => ['days' => 90, 'label' => 'Quarterly'],
        'annually' => ['days' => 365, 'label' => 'Annually']
    ],
    'default_cycle' => 'monthly'
]
```

### Dimension Types

| Dimension | Description |
|-----------|-------------|
| Billing Cycle | Recurrence period |
| Trial Period | Initial trial days |
| Renewal Term | Auto-renewal period |
| Prorate Day | Proration calculation base |

## Trial Configuration

### Trial Period Settings

```php
// Trial setup
[
    'product_id' => 1,
    'trial_enabled' => true,
    'trial_days' => 14,
    'trial_price' => 0.00,
    'allow_convert' => true,
    'auto_suspend_trial_end' => true
]
```

### Trial End Actions

```php
// What happens at trial end
[
    'action' => 'convert_or_suspend',
    'convert_on_payment' => true,
    'suspend_after_days' => 7,
    'terminate_after_days' => 14
]
```

## Renewal Dimensions

### Renewal Settings

```php
// Auto-renewal configuration
[
    'auto_renew' => true,
    'renewal_days_before' => 7,
    'renewal_billing_cycle' => 'same',
    'renewal_price' => 'current',
    'auto_charge' => true
]
```

### Renewal Pricing

```php
// Different renewal pricing
[
    'initial_price' => 10.00,
    'renewal_price' => 10.00,
    'renewal_discount' => 0,
    'allow_early_renewal' => true,
    'early_renewal_discount' => 0
]
```

## Proration Dimensions

### Prorate Configuration

```php
// How to calculate proration
[
    'prorate_enabled' => true,
    'prorate_method' => 'daily',
    'prorate_annual' => true,
    'prorate_round' => 2
]
```

### Prorate Handling

```php
// Prorate behavior
[
    'prorate_upgrades' => true,
    'prorate_downgrades' => true,
    'prorate_credit_method' => 'balance',
    'prorate_minimum_days' => 0
]
```

## Billing Dimensions Configuration

### Setup in Product

**Products > Edit Product > Billing Tab**

```php
// Configure dimensions
[
    'setup_fee' => 0.00,
    'billing_cycle' => 'monthly',
    'allow_qty' => false,
    'max_qty' => 1,
    'prorate' => true,
    'autosetup' => true
]
```

## API Functions

```php
// Get product dimensions
$result = localAPI('GetProductDimensions', [
    'productid' => 1
]);
```

## Best Practices

1. **Clear billing terms**: Define cycles clearly
2. **Trial management**: Process trials promptly
3. **Proration accuracy**: Calculate correctly
4. **Renewal automation**: Set up auto-renewals
5. **Document policies**: Communicate terms

## Related Documentation

- [Product Pricing](./whmcs-product-pricing.md)
- [Service Renewal](./whmcs-service-renewal.md)
- [Pro-Rated Billing](./whmcs-pro-rated-billing.md)
- [Service Conversion](./whmcs-service-conversion.md)