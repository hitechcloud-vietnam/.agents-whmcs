# WHMCS Client Deletion

## Overview

Client deletion in WHMCS permanently removes client records from the system. This action is irreversible and should be handled carefully, with proper data backups and compliance considerations.

## Deletion Configuration

### Settings

**Configuration > Security > Client Deletion**

```php
[
    'allow_deletion' => true,
    'require_confirmation' => true,
    'require_password' => true,
    'retain_invoices' => true,           // Keep for records
    'retain_tickets' => true,
    'retain_data_days' => 30             // Soft delete period
]
```

## Deletion Prerequisites

### Before Deletion Checklist

```php
// Checklist before deletion
[
    'check_services' => [
        'active_services' => 0,
        'pending_services' => 0
    ],
    'check_invoices' => [
        'unpaid_invoices' => 0,
        'overdue_invoices' => 0
    ],
    'check_balance' => [
        'credit_balance' => 0.00,
        'account_balance' => 0.00
    ],
    'check_tickets' => [
        'open_tickets' => 0
    ]
]
```

### Preventing Deletion

```php
// Conditions that block deletion
[
    'has_active_services' => true,
    'has_unpaid_invoices' => true,
    'has_open_tickets' => true,
    'has_credit_balance' => true,
    'has_pending_orders' => true
]
```

## Deletion Process

### Step 1: Review Client Data

**Admin: Clients > Select Client > Delete**

```php
// Review client data
[
    'userid' => 123,
    'name' => 'John Doe',
    'email' => 'john@example.com',
    'created' => '2020-01-15',
    'total_spent' => 5000.00,
    'services' => 5,
    'invoices' => 50,
    'tickets' => 20
]
```

### Step 2: Data Export (Recommended)

```php
// Export client data before deletion
[
    'action' => 'export_client_data',
    'userid' => 123,
    'include' => [
        'profile' => true,
        'services' => true,
        'invoices' => true,
        'tickets' => true,
        'notes' => true
    ],
    'format' => 'json'
]
```

### Step 3: Termination

```php
// Terminate all services first
[
    'action' => 'terminate_all_services',
    'userid' => 123,
    'preserve_data_days' => 30,
    'send_notification' => true
]
```

### Step 4: Financial Closure

```php
// Close out financial records
[
    'action' => 'refund_credit',
    'userid' => 123,
    'refund_method' => 'original_payment'
]
```

## Soft Delete vs Hard Delete

### Soft Delete

```php
// Mark as deleted, retain data
[
    'action' => 'soft_delete',
    'userid' => 123,
    'status' => 'Deleted',
    'deleted_at' => '2024-05-15',
    'deleted_by' => 'admin_id',
    'data_retained' => true,
    'recoverable' => true,
    'recover_until' => '2024-06-15'
]
```

### Hard Delete

```php
// Permanent removal
[
    'action' => 'hard_delete',
    'userid' => 123,
    'confirm_verification' => true,
    'permanent_removal' => true,
    'backup_required' => true
]
```

## Data Retention

### Retained Data

```php
// Data kept after deletion
[
    'invoices' => true,                  // Financial records
    'transactions' => true,             // Payment history
    'tickets' => true,                   // Support history
    'services' => false,                // Service records
    'contacts' => false,                // Contact info
    'notes' => true                     // Internal notes
]
```

### Anonymized Data

```php
// Anonymize rather than delete
[
    'anonymize_pii' => true,
    'fields_to_anonymize' => [
        'email' => 'deleted_' . md5($original),
        'phone' => null,
        'address' => null,
        'name' => 'Deleted User'
    ]
]
```

## Deletion Workflow

### Manual Deletion

**Admin: Clients > Select Client > Delete Account**

```php
// Deletion form
[
    'action' => 'delete_client',
    'userid' => 123,
    'delete_services' => true,
    'delete_domains' => true,
    'retain_invoices' => true,
    'reason' => 'Customer request',
    'confirm_text' => 'I understand this action cannot be undone'
]
```

### Bulk Deletion

```php
// Delete multiple clients
[
    'action' => 'bulk_delete',
    'criteria' => [
        'status' => 'inactive',
        'last_activity' => '<2023-01-01',
        'has_no_services' => true
    ],
    'dry_run' => true,                  // Preview first
    'confirm' => true
]
```

## Deletion Approval

### Require Approval

```php
// Approval workflow
[
    'require_approval' => true,
    'approval_roles' => ['admin', 'manager'],
    'notify_on_request' => true,
    'auto_approve_small' => false,
    'threshold_for_approval' => 0        // Always require
]
```

### Approval Process

```php
// Deletion request
[
    'request_id' => 789,
    'userid' => 123,
    'requested_by' => 'admin_id',
    'requested_at' => '2024-05-15',
    'reason' => 'Customer cancellation',
    'status' => 'pending'
]
```

## Deletion Effects

### Service Impact

```php
// Services affected
[
    'services_terminated' => 5,
    'domains_released' => 3,
    'email_accounts_closed' => 10,
    'databases_deleted' => 2
]
```

### Financial Impact

```php
// Financial records
[
    'total_revenue_recorded' => 5000.00,
    'invoices_preserved' => 50,
    'transactions_preserved' => 75
]
```

## Deletion Notifications

### Admin Notification

```php
// Notify admin of deletion
[
    'notify' => true,
    'recipients' => ['admin@example.com'],
    'include_details' => true
]
```

### Customer Notification (Optional)

```smarty
Subject: Your account has been deleted

Dear {$client_name},

Your account has been permanently deleted from our system.

If you have any questions, please contact support.

{$company_name}
```

## API Functions

```php
// Delete client
$result = localAPI('DeleteClient', [
    'clientid' => 123,
    'delete_services' => true,
    'retain_invoices' => true
]);

// Anonymize client
$result = localAPI('AnonymizeClient', [
    'clientid' => 123
]);

// Get deletion eligibility
$result = localAPI('GetClientDeletionStatus', [
    'clientid' => 123
]);
```

## Hooks

```php
// Hook: PreClientDelete
add_hook('PreClientDelete', 1, function($vars) {
    // $vars['userid']
    // Return false to prevent deletion
});

// Hook: ClientDeleted
add_hook('ClientDeleted', 1, function($vars) {
    // $vars['userid']
    // $vars['deleted_at']
    // Clean up related data, send notifications
});
```

## Best Practices

1. **Export first**: Always backup client data
2. **Check prerequisites**: Verify all conditions met
3. **Terminate services**: Handle before deletion
4. **Preserve records**: Keep invoices for compliance
5. **Document reason**: Log deletion reason

## Related Documentation

- [Client Creation](./whmcs-client-creation.md)
- [Client Merging](./whmcs-client-merging.md)
- [Service Termination](./whmcs-service-termination.md)
- [Client Export](./whmcs-client-export.md)