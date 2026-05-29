# WHMCS Service Cancellation

## Overview

Service cancellation in WHMCS initiates the termination process for a service. Cancellations can be immediate or scheduled for the end of the billing period.

## Cancellation Types

### Immediate Cancellation

```php
// Cancel now
[
    'service_id' => 1,
    'type' => 'immediate',
    'terminate_service' => true,
    'preserve_data_days' => 7
]
```

### End of Term Cancellation

```php
// Cancel at billing date
[
    'service_id' => 1,
    'type' => 'end_of_term',
    'do_not_renew' => true,
    'allow_undo_until' => true
]
```

## Cancellation Process

### Client Request

**Client Area > Services > Request Cancellation**

```php
// Client cancellation
[
    'service_id' => 1,
    'reason' => 'No longer needed',
    'type' => 'end_of_term',
    'feedback' => 'Too expensive',
    'confirm' => true
]
```

### Admin Cancellation

**Admin: Clients > Services > Cancel**

```php
// Admin cancellation
[
    'service_id' => 1,
    'reason' => 'Non-payment',
    'type' => 'immediate',
    'terminate_modules' => true
]
```

## Cancellation Reasons

### Common Reasons

| Reason | Frequency |
|-------|-----------|
| Too expensive | 35% |
| Not using service | 25% |
| Switching provider | 20% |
| Poor support | 10% |
| Other | 10% |

### Feedback Collection

```php
// Collect feedback
[
    'reason' => 'Too expensive',
    'feedback' => 'Found cheaper elsewhere',
    'would_return' => true,
    'suggestions' => 'Lower prices'
]
```

## Cancellation Workflow

### Request Flow

```
1. Client requests cancellation
2. Admin reviews request
3. Approve/deny cancellation
4. Set cancellation type
5. Send confirmation
6. At termination time:
   - Terminate service
   - Process final billing
   - Send termination email
```

## End of Term Cancellation

### Mark for Non-Renewal

```php
// Don't renew at end
[
    'service_id' => 1,
    'cancel_at_end' => true,
    'next_due_date' => '2024-06-15',
    'allow_undo' => true,
    'undo_deadline' => '2024-06-10'
]
```

### Cancellation Warning

```php
// Notify client of upcoming end
[
    'warning_days' => 14,
    'warning_template' => 'cancellation_warning',
    'include_reactivation_link' => true
]
```

## Cancellation Prevention

### Cannot Cancel If

```php
// Block conditions
[
    'cannot_cancel_if' => [
        'has_pending_invoices' => true,
        'in_minimum_term' => true,
        'has_open_disputes' => true
    ]
]
```

### Minimum Term

```php
// Enforce minimum term
[
    'minimum_term_months' => 3,
    'cannot_cancel_until' => '2024-08-15',
    'early_termination_fee' => 25.00
]
```

## Cancellation Fees

### Early Termination Fee

```php
// Fee for early cancel
[
    'fee_enabled' => true,
    'fee_amount' => 25.00,
    'fee_type' => 'fixed',
    'waive_if_reason' => ['technical_issues', 'service_quality']
]
```

## Cancellation Confirmation

### Send Confirmation

```smarty
Subject: Cancellation Request Received

Dear {$client_name},

We have received your cancellation request.

Service: {$service_domain}
Type: {$cancellation_type}
{if $type == 'end_of_term'}
Effective: {$next_due_date}
{else}
Effective: Immediate
{/if}

You can undo this request before it takes effect.

{$company_name}
```

## Undo Cancellation

### Cancel the Cancellation

```php
// Reverse cancellation
[
    'service_id' => 1,
    'action' => 'undo_cancellation',
    'reason' => 'Customer changed mind',
    'restore_status' => true
]
```

## Cancellation Email Templates

### Immediate Cancellation

```smarty
Subject: Service Terminated - {$service_domain}

Dear {$client_name},

Your service has been terminated.

Service: {$service_domain}
Terminated: {$termination_date}

{if $refund_amount > 0}
Refund: ${$refund_amount}
{/if}

Thank you for your business.

{$company_name}
```

### End of Term Cancellation

```smarty
Subject: Service Will Not Renew - {$service_domain}

Dear {$client_name},

Your service will not renew.

Service: {$service_domain}
Last Active Date: {$last_date}

Thank you for your business.

{$company_name}
```

## API Functions

```php
// Cancel service
$result = localAPI('CancelService', [
    'serviceid' => 1,
    'reason' => 'No longer needed',
    'type' => 'end_of_term'
]);

// Undo cancellation
$result = localAPI('UndoCancellation', [
    'serviceid' => 1
]);

// Get cancellation status
$result = localAPI('GetCancellationStatus', [
    'serviceid' => 1
]);
```

## Hooks

```php
// Hook: ServiceCancellationRequested
add_hook('ServiceCancellationRequested', 1, function($vars) {
    // $vars['serviceid']
    // $vars['reason']
    // Offer retention, notify, etc.
});

// Hook: ServiceCancelled
add_hook('ServiceCancelled', 1, function($vars) {
    // $vars['serviceid']
    // $vars['type']
    // $vars['terminate_date']
    // Handle termination, etc.
});
```

## Best Practices

1. **Offer alternatives**: Suggest downgrade first
2. **Collect feedback**: Understand cancellation reasons
3. **Allow undo**: Give time to change mind
4. **Clear communication**: Confirm all steps
5. **Track patterns**: Analyze cancellation trends

## Related Documentation

- [Service Termination](./whmcs-service-termination.md)
- [Service Suspension](./whmcs-service-suspension.md)
- [Service Upgrade Request](./whmcs-service-upgrade-request.md)
- [Dunning Settings](./whmcs-dunning-settings.md)