# WHMCS Domain Renewal

## Overview

Domain renewal in WHMCS extends domain registration periods before expiration. Auto-renewal prevents accidental domain loss.

## Renewal Process

### Auto-Renewal

```php
// Automatic renewal
[
    'domain_id' => 1,
    'auto_renew' => true,
    'renewal_threshold_days' => 30,     // Renew 30 days before expiry
    'renewal_years' => 1,
    'charge_payment' => true
]
```

### Manual Renewal

**Admin: Clients > Domains > Renew**

```php
// Manual domain renewal
[
    'domain_id' => 1,
    'years' => 1,
    'generate_invoice' => true,
    'charge_payment' => true
]
```

## Renewal Pricing

### Renewal Costs

```php
// Renewal pricing
[
    'tld' => 'com',
    'renewal_price' => 9.95,
    'renewal_years_options' => [1, 2, 5, 10],
    'early_renewal_discount' => 0,
    'bulk_renewal_discount' => 0
]
```

## Renewal Periods

### Available Periods

| TLD | Max Renewal Years |
|-----|-------------------|
| .com | 10 |
| .net | 10 |
| .org | 10 |
| .info | 10 |
| .biz | 10 |
| .io | 5 |
| .co | 5 |

## Renewal Reminders

### Notification Schedule

```php
// Renewal reminders
[
    'reminders' => [
        ['days' => 30, 'template' => 'renewal_30_days'],
        ['days' => 14, 'template' => 'renewal_14_days'],
        ['days' => 7, 'template' => 'renewal_7_days'],
        ['days' => 1, 'template' => 'renewal_1_day']
    ],
    'include_payment_link' => true
]
```

## Pre-Renewal

### Early Renewal

```php
// Allow early renewal
[
    'allow_early_renewal' => true,
    'early_renewal_days' => 90,        // Can renew 90 days early
    'preserve_existing_expiry' => true
]
```

### Pre-Renewal Discount

```php
// Discount for early
[
    'early_renewal_discount' => 10,     // 10% off
    'apply_if_renewed_days_before' => 60
]
```

## Expiration Handling

### Grace Period

```php
// Post-expiry grace period
[
    'grace_period_days' => 30,
    'redemption_fee' => 0,             // Additional fee during grace
    'restore_available' => true
]
```

### Redemption Period

```php
// Restore expired domain
[
    'redemption_available' => true,
    'redemption_period_days' => 30,
    'redemption_fee' => 80.00,          // High restore cost
    'restore_process' => 'manual_approval'
]
```

## Renewal Notification

### Client Reminder

```smarty
Subject: Domain Renewal Reminder - {$domain}

Dear {$client_name},

Your domain {$domain} will expire on {$expiry_date}.

Renew now to keep your domain active:
{$renewal_link}

Renewal Price: ${$renewal_price}/year

{$company_name}
```

## Bulk Renewal

### Multiple Domains

```php
// Bulk renewal
[
    'action' => 'bulk_renewal',
    'domain_ids' => [1, 2, 3],
    'years' => 1,
    'generate_single_invoice' => true,
    'auto_charge' => true
]
```

## API Functions

```php
// Renew domain
$result = localAPI('RenewDomain', [
    'domainid' => 1,
    'years' => 1
]);

// Get renewal pricing
$result = localAPI('GetDomainRenewalPricing', [
    'domain' => 'example.com'
]);
```

## Hooks

```php
// Hook: DomainRenewed
add_hook('DomainRenewed', 1, function($vars) {
    // $vars['domainid']
    // $vars['domain']
    // $vars['new_expiry']
});
```

## Best Practices

1. **Auto-renew**: Enable by default
2. **Send reminders**: Notify before expiration
3. **Early renewal**: Offer early renewal option
4. **Track expiring**: Monitor upcoming expirations
5. **Handle grace**: Process redemption requests

## Related Documentation

- [Domain Registration](./whmcs-domain-registration.md)
- [Domain Transfer](./whmcs-domain-transfer.md)
- [Domain Sync](./whmcs-domain-sync.md)
- [Domain Pricing](./whmcs-domain-pricing.md)