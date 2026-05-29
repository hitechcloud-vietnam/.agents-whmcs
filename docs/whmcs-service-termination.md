# WHMCS Service Termination

## Overview

Service termination in WHMCS permanently ends a service, removing it from active billing and signaling the server module to remove the account. This can be immediate or scheduled for end of billing period.

## Termination Types

### Immediate Termination

```php
// Terminate immediately
[
    'service_id' => 1,
    'type' => 'immediate',
    'preserve_data_days' => 7,
    'notify_client' => true
]
```

### End of Term Termination

```php
// Terminate at billing date
[
    'service_id' => 1,
    'type' => 'end_of_term',
    'do_not_auto_renew' => true,
    'terminate_date' => '2024-06-15'
]
```

## Termination Process

### Client-Initiated

**Client Area > Services > Request Cancellation**

```php
// Client cancellation request
[
    'service_id' => 1,
    'reason' => 'No longer needed',
    'type' => 'immediate',            // immediate, end_of_term
    'confirm' => true
]
```

### Admin Termination

**Admin: Clients > Services > Terminate**

```php
// Admin termination
[
    'service_id' => 1,
    'reason' => 'Non-payment',
    'preserve_data' => true,
    'preserve_days' => 7,
    'terminate_modules' => true
]
```

## Termination Reasons

### Common Reasons

| Reason | Description |
|--------|-------------|
| Client Request | Customer no longer needs service |
| Non-Payment | Failed to pay invoice |
| Policy Violation | Terms of service breach |
| Fraud | Fraudulent account |
| Migration | Customer transferring elsewhere |

## Module Termination

### Remove from Server

```php
// Terminate on server
[
    'module' => 'cpanel',
    'action' => 'terminate',
    'service_id' => 1,
    'username' => 'example',
    'terminate_email' => true
]
```

### Termination Steps

1. Validate termination request
2. Calculate final invoice
3. Suspend immediately (optional)
4. Terminate on server at scheduled time
5. Remove account/data
6. Update WHMCS status
7. Send confirmation email

## Final Invoice

### Calculate Final Billing

```php
// Prorated refund or charge
[
    'service_id' => 1,
    'billing_cycle' => 'monthly',
    'cycle_price' => 10.00,
    'days_used' => 15,
    'days_in_cycle' => 30,
    'refund_amount' => 5.00           // Unused portion
]
```

### No Refund Scenarios

```php
// Non-refund situations
[
    'no_refund_if' => [
        'term_length' => 'annual',
        'already_discounted' => true,
        'promotional_pricing' => true
    ]
]
```

## Data Preservation

### Backup Before Termination

```php
// Preserve data
[
    'backup_enabled' => true,
    'backup_days' => 7,
    'backup_location' => 'local',
    'notify_before_delete' => true,
    'notify_days_before' => 3
]
```

### Data Deletion

```php
// After preservation period
[
    'delete_data' => true,
    'delete_after_days' => 7,
    'deletion_notice' => true,
    'deletion_email' => true
]
```

## Termination Workflow

### Cancellation Request Flow

```
1. Client requests cancellation
2. Admin reviews request
3. Choose immediate or end-of-term
4. Send confirmation email
5. At termination time:
   - Generate final invoice
   - Terminate module account
   - Update WHMCS status
   - Send termination email
```

## Preserve Services

### Don't Terminate Immediately

```php
// End of billing period termination
[
    'type' => 'end_of_term',
    'allow_renewal' => false,
    'mark_for_termination' => true,
    'effective_date' => 'next_due_date'
]
```

## Cancellation Email

### Client Notification

```smarty
Subject: Service Termination Confirmed - {$service_domain}

Dear {$client_name},

Your service has been terminated.

Service: {$service_domain}
Terminated: {$termination_date}

{if $refund_amount > 0}
Refund: ${$refund_amount}
{/if}

{if $preserve_data}
Data preserved until: {$data_deletion_date}
{/if}

Thank you for your business.

{$company_name}
```

## Termination Prevention

### Block Termination Conditions

```php
// Prevent termination if
[
    'block_if' => [
        'has_pending_invoices' => true,
        'has_open_tickets' => false,
        'in_trial' => true
    ]
]
```

## API Functions

```php
// Terminate service
$result = localAPI('TerminateService', [
    'serviceid' => 1,
    'type' => 'immediate',
    'preserve_data_days' => 7
]);

// Request cancellation
$result = localAPI('RequestCancellation', [
    'serviceid' => 1,
    'reason' => 'No longer needed',
    'type' => 'immediate'
]);

// Cancel termination request
$result = localAPI('CancelTerminationRequest', [
    'serviceid' => 1
]);
```

## Hooks

```php
// Hook: ServiceTerminationRequested
add_hook('ServiceTerminationRequested', 1, function($vars) {
    // $vars['serviceid']
    // $vars['reason']
    // Notify, log, etc.
});

// Hook: ServiceTerminated
add_hook('ServiceTerminated', 1, function($vars) {
    // $vars['serviceid']
    // $vars['terminate_type']
    // Cleanup, notify, etc.
});
```

## Best Practices

1. **Clear cancellation policy**: Communicate terms clearly
2. **Offer alternatives**: Suggest downgrade before cancellation
3. **Backup data**: Preserve data before deletion
4. **Document reasons**: Track cancellation reasons
5. **Follow up**: Analyze cancellation patterns

## Related Documentation

- [Service Cancellation](./whmcs-service-cancellation.md)
- [Service Suspension](./whmcs-service-suspension.md)
- [Service Reactivation](./whmcs-service-reactivation.md)
- [Write Offs](./whmcs-write-offs.md)