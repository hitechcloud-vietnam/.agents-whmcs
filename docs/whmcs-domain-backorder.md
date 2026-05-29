# WHMCS Domain Backorder

## Overview

Domain backorder in WHMCS allows customers to reserve expiring domains before they become available, with automatic registration attempts when the domain drops.

## Backorder Configuration

### Enable Backorder

**Configuration > Domains > Backorders**

```php
// Backorder settings
[
    'enable_backorders' => false,
    'price' => 19.95,
    'success_fee' => 0,
    'auto_register' => true
]
```

## Backorder Process

### Place Backorder

```php
// Customer backorder
[
    'domain' => 'expiring.com',
    'customer_id' => 123,
    'backorder_price' => 19.95,
    'notify_on_drop' => true
]
```

## Backorder Priority

### Priority System

```php
// Backorder priority
[
    'domain' => 'expiring.com',
    'backorders' => [
        ['customer_id' => 123, 'priority' => 1, 'date' => '2024-05-01'],
        ['customer_id' => 456, 'priority' => 2, 'date' => '2024-05-05']
    ],
    'first_priority' => 'customer_123'
]
```

## Domain Drops

### Auto-Catch

```php
// Automatic registration
[
    'domain' => 'expiring.com',
    'backorder_customer' => 123,
    'auto_register' => true,
    'max_price' => 50.00,
    'success_fee' => 25.00
]
```

### Manual Confirmation

```php
// Confirm before catch
[
    'domain' => 'expiring.com',
    'auto_register' => false,
    'notify_customer' => true,
    'confirm_hours' => 24
]
```

## Backorder Fees

### Pricing

```php
// Backorder pricing
[
    'backorder_fee' => 19.95,
    'success_fee' => 0,
    'annual_renewal' => 9.95,
    'refund_if_failed' => true
]
```

## API Functions

```php
// Place backorder
$result = localAPI('PlaceDomainBackorder', [
    'domain' => 'expiring.com'
]);

// Get backorder status
$result = localAPI('GetDomainBackorder', [
    'domain' => 'expiring.com'
]);
```

## Hooks

```php
// Hook: DomainBackorderFulfilled
add_hook('DomainBackorderFulfilled', 1, function($vars) {
    // $vars['domain']
    // $vars['customerid']
    // Notify customer, etc.
});
```

## Best Practices

1. **Clear pricing**: Show all fees upfront
2. **Notify customers**: Update on domain status
3. **Fair priority**: First-come first-served
4. **Auto-catch**: Register quickly when available

## Related Documentation

- [Domain Catching](./whmcs-domain-catching.md)
- [Domain Auction](./whmcs-domain-auction.md)
- [Domain Registration](./whmcs-domain-registration.md)