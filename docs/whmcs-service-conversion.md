# WHMCS Service Conversion

## Overview

Service conversion in WHMCS handles the transition of trial services to paid accounts, including billing setup and module provisioning.

## Conversion Types

### Trial to Paid

```php
// Convert trial service
[
    'service_id' => 1,
    'from_status' => 'Trial',
    'to_status' => 'Active',
    'billing_cycle' => 'monthly',
    'first_payment' => 10.00
]
```

### Free to Paid

```php
// Convert free service
[
    'service_id' => 1,
    'type' => 'free_to_paid',
    'billing_cycle' => 'annual',
    'setup_fee' => 0.00,
    'first_invoice' => 100.00
]
```

## Conversion Process

### Automatic Conversion

```php
// Auto-convert on trial end
[
    'auto_convert' => true,
    'convert_on_trial_end' => true,
    'generate_invoice' => true,
    'require_payment' => true
]
```

### Manual Conversion

**Admin: Clients > Services > Convert to Paid**

```php
// Manual conversion
[
    'service_id' => 1,
    'billing_cycle' => 'monthly',
    'prorate_trial' => true,
    'convert_now' => true
]
```

## Trial Period

### Trial Configuration

```php
// Trial settings
[
    'trial_days' => 14,
    'trial_price' => 0.00,
    'extend_trial' => true,
    'trial_warning_days' => 3
]
```

### Trial End Notification

```smarty
Subject: Trial Ending Soon - {$service_domain}

Dear {$client_name},

Your trial ends on {$trial_end_date}.

Convert now to keep your service active.

Convert: {$conversion_link}

{$company_name}
```

## Conversion Billing

### First Invoice

```php
// Generate conversion invoice
[
    'service_id' => 1,
    'items' => [
        ['description' => 'Setup Fee', 'amount' => 0.00],
        ['description' => 'First Month', 'amount' => 10.00]
    ],
    'total' => 10.00,
    'due_date' => 'immediate'
]
```

### Prorated Conversion

```php
// Prorate from conversion date
[
    'service_id' => 1,
    'days_in_trial_used' => 10,
    'trial_days' => 14,
    'prorate_amount' => 0.00        // No charge for partial
]
```

## Module Provisioning

### Provision on Conversion

```php
// Create account on conversion
[
    'module' => 'cpanel',
    'action' => 'create',
    'service_id' => 1,
    'username' => 'example',
    'password' => 'Generated123!'
]
```

## Conversion Options

### Conversion Choices

```php
// Client conversion options
[
    'service_id' => 1,
    'options' => [
        ['billing_cycle' => 'monthly', 'price' => 10.00],
        ['billing_cycle' => 'annually', 'price' => 100.00, 'discount' => 15]
    ],
    'allow_skip' => false
]
```

## Conversion Failure

### Handle Conversion Issues

```php
// If payment fails
[
    'on_failure' => [
        'suspend_days' => 7,
        'notify_client' => true,
        'convert_to_free' => false
    ]
]
```

## Conversion Email

### Conversion Confirmation

```smarty
Subject: Service Converted - {$service_domain}

Dear {$client_name},

Your service has been converted to a paid account.

Service: {$service_domain}
Billing: {$billing_cycle}
Amount: ${$amount}

Next invoice: {$next_due_date}

Thank you!

{$company_name}
```

## API Functions

```php
// Convert service
$result = localAPI('ConvertService', [
    'serviceid' => 1,
    'billingcycle' => 'monthly'
]);

// Get conversion options
$result = localAPI('GetConversionOptions', [
    'serviceid' => 1
]);
```

## Hooks

```php
// Hook: ServiceConverted
add_hook('ServiceConverted', 1, function($vars) {
    // $vars['serviceid']
    // $vars['billing_cycle']
    // $vars['first_payment']
    // Provision module, notify, etc.
});
```

## Best Practices

1. **Clear communication**: Inform about trial end
2. **Offer options**: Multiple billing cycles
3. **Incentivize annual**: Discount longer terms
4. **Quick conversion**: Process promptly
5. **Follow up**: Remind unconverted trials

## Related Documentation

- [Service Creation](./whmcs-service-creation.md)
- [Service Upgrade](./whmcs-service-upgrade.md)
- [Service Cancellation](./whmcs-service-cancellation.md)
- [Trial Products](./whmcs-trial-products.md)