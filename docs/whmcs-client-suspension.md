# WHMCS Client Suspension

## Overview

Client suspension in WHMCS allows administrators to temporarily suspend a client account. This prevents login access while preserving all account data and services.

## Suspension Reasons

### Common Reasons

| Reason | Description |
|--------|-------------|
| Non-Payment | Payment overdue |
| Policy Violation | Terms of service violation |
| Fraud | Suspected fraudulent activity |
| Request | Customer requested suspension |
| Investigation | Security investigation |

## Suspension Process

### Suspend Client

**Admin: Clients > Select Client > Suspend Account**

```php
// Suspend account
[
    'userid' => 123,
    'reason' => 'Non-Payment',
    'suspend_services' => true,
    'notify_client' => true,
    'suspended_by' => 'admin_id',
    'suspended_at' => '2024-05-15'
]
```

### Suspension Options

```php
// Suspension configuration
[
    'suspend_login' => true,
    'suspend_services' => true,
    'suspend_domains' => false,
    'allow_ticket_view' => true,
    'allow_knowledgebase' => true
]
```

## Suspended Account Features

### Access Restrictions

```php
// What's disabled during suspension
[
    'login_disabled' => true,
    'order_disabled' => true,
    'service_access_disabled' => true,
    'invoice_payment_disabled' => true
]
```

### What's Retained

```php
// Data preserved during suspension
[
    'profile' => true,
    'services' => true,          // Suspended, not deleted
    'invoices' => true,
    'tickets' => true,
    'credits' => true
]
```

## Service Suspension

### Suspend Services

```php
// Suspend all services
[
    'userid' => 123,
    'suspend_reason' => 'Account suspended',
    'suspend_modules' => true,
    'suspend_at' => 'cron'
]
```

### Individual Service Suspension

```php
// Suspend specific service
[
    'service_id' => 1,
    'suspend_reason' => 'Non-payment',
    'suspend_date' => '2024-05-15'
]
```

## Reactivation

### Reactivate Account

**Admin: Clients > Select Client > Reactivate Account**

```php
// Reactivate client
[
    'userid' => 123,
    'reactivate_services' => true,
    'charge_overdue' => true,
    'reactivation_fee' => 0.00,
    'notify_client' => true,
    'reactivated_by' => 'admin_id',
    'reactivated_at' => '2024-05-20'
]
```

### Prorate Charges

```php
// Charge for suspended period
[
    'charge_suspended_days' => true,
    'suspended_days' => 5,
    'daily_rate' => 0.33,
    'charge_amount' => 1.65
]
```

## Suspension Notifications

### Client Notification

```smarty
Subject: Your account has been suspended

Dear {$client_name},

Your account has been suspended.

Reason: {$suspension_reason}

To reactivate, please:
1. Pay any outstanding invoices
2. Contact support

{$company_name}
```

## Suspension Reports

### Suspension Statistics

**Reports > Clients > Suspension Report**

```php
// Suspension report
[
    'period' => 'May 2024',
    'total_suspensions' => 15,
    'reactivated' => 10,
    'still_suspended' => 5,
    'by_reason' => [
        'non_payment' => 10,
        'fraud' => 3,
        'requested' => 2
    ]
]
```

## Auto Suspension

### Configure Auto-Suspension

**Configuration > Automation > Auto Suspension**

```php
// Auto-suspend based on overdue
[
    'auto_suspend_enabled' => true,
    'suspend_after_days' => 14,
    'suspend_services' => true,
    'suspend_login' => true,
    'notify_before' => true,
    'notify_days_before' => 3
]
```

## API Functions

```php
// Suspend client
$result = localAPI('SuspendClient', [
    'clientid' => 123,
    'reason' => 'Non-Payment'
]);

// Reactivate client
$result = localAPI('ReactivateClient', [
    'clientid' => 123
]);

// Get suspension status
$result = localAPI('GetClientSuspension', [
    'clientid' => 123
]);
```

## Hooks

```php
// Hook: ClientSuspended
add_hook('ClientSuspended', 1, function($vars) {
    // $vars['userid']
    // $vars['reason']
    // Suspend services, notify, etc.
});

// Hook: ClientReactivated
add_hook('ClientReactivated', 1, function($vars) {
    // $vars['userid']
    // $vars['reactivated_by']
    // Reactivate services, notify, etc.
});
```

## Best Practices

1. **Clear communication**: Notify client of suspension
2. **Document reasons**: Record suspension reason
3. **Quick resolution**: Aim to reactivate promptly
4. **Fair policies**: Apply suspension consistently
5. **Track patterns**: Monitor repeat suspensions

## Related Documentation

- [Service Suspension](./whmcs-service-suspension.md)
- [Service Reactivation](./whmcs-service-reactivation.md)
- [Dunning Settings](./whmcs-dunning-settings.md)
- [Client Deletion](./whmcs-client-deletion.md)