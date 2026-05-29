# WHMCS Service Reactivation

## Overview

Service reactivation in WHMCS restores a suspended service, re-enabling client access and synchronizing with the server module. Reactivation typically follows payment of overdue invoices.

## Reactivation Triggers

### Payment Received

```php
// Auto-reactivate on payment
[
    'auto_reactivate' => true,
    'trigger' => 'invoice_paid',
    'reactivate_modules' => true,
    'notify_client' => true
]
```

### Manual Reactivation

**Admin: Clients > Services > Reactivate**

```php
// Admin reactivates service
[
    'service_id' => 1,
    'reason' => 'Payment received',
    'reactivate_modules' => true,
    'notify_client' => true
]
```

## Reactivation Process

### Module Un suspension

```php
// Unuspend on server
[
    'module' => 'cpanel',
    'action' => 'unsuspend',
    'service_id' => 1,
    'username' => 'example'
]
```

### Reactivation Steps

1. Verify payment received
2. Unsuspend on server
3. Update WHMCS status to Active
4. Send notification email

## Reactivation Fees

### Fee Configuration

```php
// Reactivation fee
[
    'reactivation_fee' => 5.00,
    'charge_on_reactivate' => true,
    'waive_if_first' => true,
    'waive_if_within_days' => 3
]
```

### Fee Application

```php
// Apply reactivation fee
[
    'service_id' => 1,
    'fee_amount' => 5.00,
    'add_to_invoice' => true,
    'reason' => 'Reactivation fee'
]
```

## Overdue Invoice Handling

### Charge Overdue Amount

```php
// Charge outstanding balance
[
    'service_id' => 1,
    'charge_overdue' => true,
    'overdue_amount' => 25.00,
    'include_late_fees' => true,
    'payment_method' => 'default'
]
```

## Service Status Update

### Change to Active

```php
// Update service status
[
    'service_id' => 1,
    'old_status' => 'Suspended',
    'new_status' => 'Active',
    'reactivated_at' => '2024-05-20',
    'reactivated_by' => 'admin_id',
    'suspension_duration_days' => 5
]
```

## Reactivation Email

### Notify Client

```smarty
Subject: Service Reactivated - {$service_domain}

Dear {$client_name},

Your service has been reactivated.

Service: {$service_domain}
Access Restored: {$service_url}

{if $overdue_charged}
Payment received: ${$amount}
Remaining balance: ${$balance}
{/if}

Thank you!

{$company_name}
```

## Partial Reactivation

### Selective Service

```php
// Reactivate specific services
[
    'userid' => 123,
    'service_ids' => [1, 2],
    'skip_service_id' => 3
]
```

## Reactivation Confirmation

### Show to Client

```php
// Confirmation message
[
    'service' => 'example.com',
    'status' => 'Active',
    'reactivation_fee' => '$5.00',
    'overdue_charged' => '$25.00',
    'total_charged' => '$30.00'
]
```

## API Functions

```php
// Reactivate service
$result = localAPI('ReactivateService', [
    'serviceid' => 1,
    'charge_overdue' => true
]);

// Batch reactivate
$result = localAPI('BatchReactivateServices', [
    'serviceids' => [1, 2, 3]
]);
```

## Hooks

```php
// Hook: ServiceReactivated
add_hook('ServiceReactivated', 1, function($vars) {
    // $vars['serviceid']
    // $vars['reactivated_by']
    // $vars['suspension_duration']
    // Unsuspend module, notify, etc.
});
```

## Best Practices

1. **Verify payment**: Confirm payment before reactivating
2. **Charge fees**: Apply reactivation fees appropriately
3. **Quick response**: Reactivate promptly after payment
4. **Clear communication**: Send reactivation confirmation
5. **Track patterns**: Monitor suspension/reactivation cycles

## Related Documentation

- [Service Suspension](./whmcs-service-suspension.md)
- [Service Creation](./whmcs-service-creation.md)
- [Service Termination](./whmcs-service-termination.md)
- [Service Cancellation](./whmcs-service-cancellation.md)