# WHMCS Service Override

## Overview

Service override in WHMCS allows administrators to set custom pricing for individual services, bypassing the standard product pricing.

## Override Configuration

### Set Override Price

**Admin: Clients > Services > Override Pricing**

```php
// Override pricing
[
    'service_id' => 1,
    'override_price' => 9.00,          // Custom price
    'billing_cycle' => 'monthly',
    'override_setup_fee' => 0.00,
    'effective_date' => '2024-05-15',
    'expires' => null,                  // null = permanent
    'reason' => 'Loyalty discount'
]
```

### Override Fields

| Field | Description |
|-------|-------------|
| price | Custom recurring price |
| setup_fee | Custom setup fee |
| billing_cycle | Override billing cycle |
| reason | Reason for override |

## Override Types

### Price Override

```php
// Different price than product
[
    'service_id' => 1,
    'original_price' => 10.00,
    'override_price' => 8.00,
    'savings' => 2.00
]
```

### Billing Cycle Override

```php
// Different billing cycle
[
    'service_id' => 1,
    'product_cycle' => 'monthly',
    'override_cycle' => 'quarterly',
    'override_price' => 25.00          // Quarterly price
]
```

### Combined Override

```php
// Both price and cycle
[
    'service_id' => 1,
    'override_price' => 100.00,
    'override_cycle' => 'annually',
    'reason' => 'Special pricing'
]
```

## Override Management

### Update Override

```php
// Modify existing override
[
    'service_id' => 1,
    'override_price' => 8.50,
    'update_reason' => 'Increased discount'
]
```

### Remove Override

```php
// Revert to product pricing
[
    'service_id' => 1,
    'action' => 'remove_override',
    'effective' => 'next_billing',
    'reason' => 'Discount ended'
]
```

### Expire Override

```php
// Override expires
[
    'service_id' => 1,
    'expires' => '2024-12-31',
    'expire_action' => 'revert_to_product'
]
```

## Override Display

### Show Override in Admin

```php
// Admin service view
[
    'service_id' => 1,
    'product_price' => 10.00,
    'override_price' => 8.00,
    'status' => 'overridden',
    'reason' => 'Loyalty discount'
]
```

### Client View

```php
// Client sees override price
[
    'service_id' => 1,
    'price' => 8.00,                   // Shows override
    'billing_cycle' => 'monthly'
]
```

## Override on Invoice

### Invoice Line Item

```
Service: Web Hosting (Override Pricing)
Price: $8.00 (Product Price: $10.00)
Billing: Monthly
```

## Override Reasons

### Common Reasons

| Reason | Description |
|--------|-------------|
| Loyalty | Long-term customer discount |
| Negotiation | Negotiated special price |
| Competitor | Match competitor pricing |
| Volume | High volume discount |
| Referral | Referral incentive |

## Override Approval

### Require Approval

```php
// Override requires approval
[
    'require_approval' => true,
    'approvers' => ['admin', 'manager'],
    'auto_approve_small' => true,
    'small_threshold' => 2.00
]
```

### Approval Workflow

```php
// Override request
[
    'service_id' => 1,
    'requested_price' => 8.00,
    'reason' => 'Customer request',
    'status' => 'pending',
    'requested_by' => 'admin_id'
]
```

## Bulk Override

### Apply to Multiple

```php
// Bulk price override
[
    'action' => 'bulk_override',
    'service_ids' => [1, 2, 3],
    'override_price' => 9.00,
    'reason' => 'Summer promotion'
]
```

## API Functions

```php
// Set override price
$result = localAPI('SetServiceOverride', [
    'serviceid' => 1,
    'price' => 9.00,
    'billingcycle' => 'monthly'
]);

// Remove override
$result = localAPI('RemoveServiceOverride', [
    'serviceid' => 1
]);

// Get override
$result = localAPI('GetServiceOverride', [
    'serviceid' => 1
]);
```

## Hooks

```php
// Hook: ServiceOverrideSet
add_hook('ServiceOverrideSet', 1, function($vars) {
    // $vars['serviceid']
    // $vars['price']
    // $vars['reason']
});
```

## Best Practices

1. **Document reasons**: Record why override was set
2. **Set expiration**: Use temporary overrides with end date
3. **Review periodically**: Audit override usage
4. **Consistent application**: Apply fairly across customers
5. **Track impact**: Monitor revenue effect of overrides

## Related Documentation

- [Product Pricing](./whmcs-product-pricing.md)
- [Service Modification](./whmcs-service-modification.md)
- [Service Upgrade Override](./whmcs-service-upgrade-override.md)
- [Pricing Tiers](./whmcs-pricing-tiers.md)