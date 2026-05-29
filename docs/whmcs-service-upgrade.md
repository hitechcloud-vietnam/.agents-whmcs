# WHMCS Service Upgrade

## Overview

Service upgrade in WHMCS allows clients to move to higher-tier products or add resources. Upgrades can be done with proration or at renewal time.

## Upgrade Types

### Product Upgrade

```php
// Change to higher product
[
    'service_id' => 1,
    'from_product_id' => 1,          // Basic
    'to_product_id' => 2,           // Premium
    'billing_cycle' => 'monthly',
    'prorate' => true
]
```

### Resource Upgrade

```php
// Increase resources
[
    'service_id' => 1,
    'changes' => [
        ['type' => 'config_option', 'id' => 5, 'add' => true],
        ['type' => 'config_option', 'id' => 6, 'qty' => 2]
    ]
]
```

## Upgrade Process

### Client Upgrade Request

**Client Area > Services > Upgrade Options**

```php
// Client selects upgrade
[
    'service_id' => 1,
    'upgrade_to' => 'premium',
    'effective' => 'immediately',     // immediately, next_billing
    'prorate_charge' => 5.33
]
```

### Admin Upgrade

**Admin: Clients > Services > Upgrade**

```php
// Admin performs upgrade
[
    'service_id' => 1,
    'new_product_id' => 2,
    'override_price' => null,
    'send_notification' => true
]
```

## Proration Calculation

### Immediate Upgrade

```php
// Prorated upgrade charge
[
    'old_price' => 10.00,            // Basic - $10/mo
    'new_price' => 20.00,           // Premium - $20/mo
    'days_remaining' => 15,
    'days_in_cycle' => 30,
    'prorate_amount' => 5.00       // $10 difference * 15/30
]
```

### Next Cycle Upgrade

```php
// Upgrade at renewal
[
    'upgrade_effective' => 'next_billing',
    'next_invoice_amount' => 20.00,
    'no_prorate_charge' => true
]
```

## Upgrade Pricing

### Price Calculation

```php
// Calculate upgrade cost
function calculateUpgradeProrate($oldPrice, $newPrice, $daysRemaining, $daysInCycle) {
    $priceDifference = $newPrice - $oldPrice;
    $dailyRate = $priceDifference / $daysInCycle;
    return $dailyRate * $daysRemaining;
}

// Example
// $20 - $10 = $10 difference
// $10 / 30 days = $0.33/day
// 15 days remaining = $5.00
```

### Override Pricing

```php
// Admin can override price
[
    'override_prorate' => true,
    'custom_amount' => 10.00,
    'reason' => 'Customer loyalty'
]
```

## Module Update

### Sync to Server

```php
// Update server after upgrade
[
    'module' => 'cpanel',
    'action' => 'upgrade',
    'service_id' => 1,
    'new_plan' => 'premium',
    'new_resources' => [
        'disk' => '50GB',
        'bandwidth' => '500GB'
    ]
]
```

## Upgrade Confirmation

### Show to Client

```php
// Confirmation message
[
    'service' => 'example.com',
    'from' => 'Basic ($10/mo)',
    'to' => 'Premium ($20/mo)',
    'one_time_charge' => '$5.00',
    'next_invoice' => '$20.00'
]
```

## Completed Upgrade

### Update Records

```php
// After upgrade completes
[
    'service_id' => 1,
    'old_product_id' => 1,
    'new_product_id' => 2,
    'new_price' => 20.00,
    'billing_cycle' => 'monthly',
    'upgraded_at' => '2024-05-15',
    'prorate_charged' => 5.00
]
```

## Notifications

### Client Notification

```smarty
Subject: Service Upgraded - {$service_domain}

Dear {$client_name},

Your service has been upgraded.

From: Basic Hosting
To: Premium Hosting

One-time charge: {$prorate_amount}
Your next invoice will reflect: {$new_price}

{$company_name}
```

## API Functions

```php
// Upgrade service
$result = localAPI('UpgradeService', [
    'serviceid' => 1,
    'newproductid' => 2,
    'type' => 'product',
    'prorate' => true
]);

// Calculate upgrade
$result = localAPI('CalculateUpgrade', [
    'serviceid' => 1,
    'newproductid' => 2
]);
```

## Hooks

```php
// Hook: ServiceUpgrade
add_hook('ServiceUpgrade', 1, function($vars) {
    // $vars['serviceid']
    // $vars['old_product_id']
    // $vars['new_product_id']
    // $vars['amount']
    // Update module, notify, etc.
});
```

## Best Practices

1. **Clear pricing**: Show proration clearly
2. **Immediate sync**: Update server promptly
3. **Document upgrades**: Track all changes
4. **Notify clients**: Send upgrade confirmation
5. **Test module**: Verify upgrade works on server

## Related Documentation

- [Service Downgrade](./whmcs-service-downgrade.md)
- [Service Modification](./whmcs-service-modification.md)
- [Prorate Calculator](./whmcs-prorate-calculator.md)
- [Pricing Tiers](./whmcs-pricing-tiers.md)