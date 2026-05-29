# WHMCS Service Renewal

## Overview

Service renewal in WHMCS extends the service billing period, generating invoices and maintaining active status. Renewals can be automatic or manual.

## Renewal Types

### Automatic Renewal

```php
// Auto-renew on due date
[
    'auto_renew' => true,
    'renewal_days_before' => 7,
    'generate_invoice' => true,
    'charge_payment' => true
]
```

### Manual Renewal

**Admin: Clients > Services > Renew**

```php
// Manual renewal
[
    'service_id' => 1,
    'renewal_period' => 1,             // months/years
    'billing_cycle' => 'monthly',
    'generate_invoice' => true,
    'charge_payment' => true
]
```

## Renewal Process

### Invoice Generation

```php
// Generate renewal invoice
[
    'service_id' => 1,
    'billing_cycle' => 'monthly',
    'amount' => 10.00,
    'due_date' => '2024-06-01',
    'items' => [
        ['description' => 'Web Hosting - Monthly', 'amount' => 10.00]
    ]
]
```

### Renewal Steps

1. Calculate next due date
2. Generate invoice
3. Attempt automatic payment
4. If paid, extend service
5. Send confirmation

## Renewal Periods

### Available Periods

| Period | Billing Cycle |
|--------|---------------|
| 1 Month | Monthly |
| 3 Months | Quarterly |
| 6 Months | Semi-Annual |
| 12 Months | Annual |
| 24 Months | Biennial |
| 36 Months | Triennial |

## Renewal Pricing

### Use Product Pricing

```php
// Renewal at product price
[
    'product_id' => 1,
    'monthly' => 10.00,
    'quarterly' => 28.00,
    'annually' => 100.00
]
```

### Custom Renewal Price

```php
// Override renewal price
[
    'service_id' => 1,
    'custom_renewal_price' => 9.00,
    'reason' => 'Loyalty discount'
]
```

## Pre-Renewal

### Reminder Emails

```php
// Send renewal reminders
[
    'reminder_days' => [14, 7, 3, 1],
    'reminder_template' => 'renewal_reminder',
    'include_payment_link' => true
]
```

### Renewal Discount

```php
// Discount for early renewal
[
    'early_renewal_discount' => 10,    // 10%
    'discount_if_renewed_days_before' => 30,
    'apply_to_billing_cycles' => ['annually']
]
```

## Domain Renewal

### Domain Auto-Renew

```php
// Domain renewal settings
[
    'auto_renew' => true,
    'renew_before_days' => 30,
    'auto_renew_pricing' => 'standard',
    'notify_before_renewal' => true
]
```

### Domain Renewal Process

```php
// Renew domain
[
    'domain_id' => 1,
    'renew_years' => 1,
    'process_at' => 'expiry',
    'register_again' => true
]
```

## Renewal Failure

### Handle Failed Renewal

```php
// Payment failure
[
    'on_payment_failure' => [
        'retry_attempts' => 3,
        'retry_days' => [1, 3, 7],
        'suspend_after_failure' => true,
        'suspend_days' => 14
    ]
]
```

## Renewal Confirmation

### Client Notification

```smarty
Subject: Service Renewed - {$service_domain}

Dear {$client_name},

Your service has been renewed.

Service: {$service_domain}
Period: {$renewal_period}
Next Due: {$next_due_date}
Amount: {$amount}

Thank you!

{$company_name}
```

## Multiple Service Renewal

### Bulk Renewal

```php
// Renew multiple services
[
    'action' => 'bulk_renewal',
    'client_id' => 123,
    'service_ids' => [1, 2, 3],
    'billing_cycle' => 'annual',
    'generate_single_invoice' => true
]
```

## API Functions

```php
// Renew service
$result = localAPI('RenewService', [
    'serviceid' => 1,
    'renewal_period' => 12,
    'billing_cycle' => 'annually'
]);

// Get renewal pricing
$result = localAPI('GetServiceRenewalPricing', [
    'serviceid' => 1
]);
```

## Hooks

```php
// Hook: ServiceRenewalComplete
add_hook('ServiceRenewalComplete', 1, function($vars) {
    // $vars['serviceid']
    // $vars['billing_cycle']
    // $vars['amount']
    // Send confirmation, etc.
});
```

## Best Practices

1. **Send reminders**: Notify before due date
2. **Auto-payment**: Set up automatic charging
3. **Offer discounts**: Incentivize longer terms
4. **Clear pricing**: Show renewal costs clearly
5. **Track failures**: Monitor renewal failures

## Related Documentation

- [Recurring Billing](./whmcs-recurring-billing.md)
- [Service Creation](./whmcs-service-creation.md)
- [Service Termination](./whmcs-service-termination.md)
- [Domain Renewal](./whmcs-domain-renewal.md)