# WHMCS Service Downgrade

## Overview

Service downgrade in WHMCS allows clients to move to lower-tier products or reduce resources. Downgrades typically credit the difference for remaining billing period.

## Downgrade Types

### Product Downgrade

```php
// Change to lower product
[
    'service_id' => 1,
    'from_product_id' => 2,          // Premium
    'to_product_id' => 1,           // Basic
    'billing_cycle' => 'monthly',
    'credit_amount' => 5.00
]
```

### Resource Reduction

```php
// Decrease resources
[
    'service_id' => 1,
    'changes' => [
        ['type' => 'config_option', 'id' => 5, 'remove' => true]
    ],
    'credit_amount' => 2.50
]
```

## Downgrade Process

### Client Downgrade Request

**Client Area > Services > Downgrade**

```php
// Client requests downgrade
[
    'service_id' => 1,
    'downgrade_to' => 'basic',
    'effective' => 'immediately',     // immediately, next_billing
    'credit_amount' => 5.00
]
```

### Admin Downgrade

**Admin: Clients > Services > Downgrade**

```php
// Admin performs downgrade
[
    'service_id' => 1,
    'new_product_id' => 1,
    'credit_type' => 'account_credit',  // account_credit, next_invoice
    'send_notification' => true
]
```

## Credit Calculation

### Prorated Credit

```php
// Credit for remaining period
[
    'old_price' => 20.00,            // Premium - $20/mo
    'new_price' => 10.00,           // Basic - $10/mo
    'days_remaining' => 15,
    'days_in_cycle' => 30,
    'credit_amount' => 5.00          // $10 difference * 15/30
]
```

### Credit Handling

```php
// How to apply credit
[
    'credit_to_balance' => true,      // Add to client credit
    'apply_to_next_invoice' => false,
    'refund_amount' => false
]
```

## Next Cycle Downgrade

### Delayed Downgrade

```php
// Effective at renewal
[
    'effective_date' => 'next_billing',
    'next_due_date' => '2024-06-15',
    'new_price' => 10.00,
    'no_immediate_credit' => true,
    'note' => 'Downgrade will apply on next billing date'
]
```

## Downgrade Confirmation

### Show to Client

```php
// Confirmation message
[
    'service' => 'example.com',
    'from' => 'Premium ($20/mo)',
    'to' => 'Basic ($10/mo)',
    'credit' => '$5.00 to your account',
    'next_invoice' => '$10.00'
]
```

## Module Update

### Sync to Server

```php
// Update server after downgrade
[
    'module' => 'cpanel',
    'action' => 'downgrade',
    'service_id' => 1,
    'new_plan' => 'basic',
    'new_resources' => [
        'disk' => '10GB',
        'bandwidth' => '100GB'
    ]
]
```

### Warning About Data

```smarty
Subject: Service Downgrade Notice - {$service_domain}

Dear {$client_name},

Your service will be downgraded.

Important: Please ensure your usage is within the new plan limits
before the downgrade takes effect.

From: Premium (50GB disk, 500GB bandwidth)
To: Basic (10GB disk, 100GB bandwidth)

Effective: {$effective_date}

{$company_name}
```

## Completed Downgrade

### Update Records

```php
// After downgrade completes
[
    'service_id' => 1,
    'old_product_id' => 2,
    'new_product_id' => 1,
    'new_price' => 10.00,
    'billing_cycle' => 'monthly',
    'downgraded_at' => '2024-05-15',
    'credit_applied' => 5.00
]
```

## Downgrade Restrictions

### Prevent Downgrade

```php
// Block if conditions met
[
    'cannot_downgrade_if' => [
        'usage_exceeds_new_limits' => true,
        'has_custom_config' => true,
        'in_trial_period' => true
    ]
]
```

### Data Check

```php
// Check usage before downgrade
[
    'current_disk_usage' => '15GB',
    'new_plan_limit' => '10GB',
    'over_limit' => true,
    'block_message' => 'Reduce disk usage below 10GB to downgrade'
]
```

## API Functions

```php
// Downgrade service
$result = localAPI('DowngradeService', [
    'serviceid' => 1,
    'newproductid' => 1,
    'type' => 'product',
    'credit' => 'balance'
]);

// Calculate downgrade
$result = localAPI('CalculateDowngrade', [
    'serviceid' => 1,
    'newproductid' => 1
]);
```

## Hooks

```php
// Hook: ServiceDowngrade
add_hook('ServiceDowngrade', 1, function($vars) {
    // $vars['serviceid']
    // $vars['old_product_id']
    // $vars['new_product_id']
    // $vars['credit_amount']
    // Update module, notify, etc.
});
```

## Best Practices

1. **Check limits**: Verify usage fits new plan
2. **Warn clients**: Alert about data restrictions
3. **Clear credits**: Show credit amount clearly
4. **Document changes**: Track all downgrades
5. **Update modules**: Sync server configuration

## Related Documentation

- [Service Upgrade](./whmcs-service-upgrade.md)
- [Service Modification](./whmcs-service-modification.md)
- [Prorate Calculator](./whmcs-prorate-calculator.md)
- [Credit System](./whmcs-credit-system.md)