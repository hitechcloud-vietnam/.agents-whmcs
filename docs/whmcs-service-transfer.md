# WHMCS Service Transfer

## Overview

Service transfer in WHMCS moves a service from one client account to another. This is useful for account mergers, reselling arrangements, or gifting services.

## Transfer Types

### Full Transfer

```php
// Transfer to new owner
[
    'service_id' => 1,
    'from_userid' => 123,
    'to_userid' => 456,
    'transfer_type' => 'full',
    'include_data' => true
]
```

### Server Transfer

```php
// Transfer to different server
[
    'service_id' => 1,
    'from_server' => 1,
    'to_server' => 2,
    'migrate_data' => true
]
```

## Transfer Process

### Initiate Transfer

**Admin: Clients > Services > Transfer**

```php
// Request transfer
[
    'service_id' => 1,
    'new_owner_email' => 'new@example.com',
    'reason' => 'Account merger',
    'approve_required' => true
]
```

### Transfer Approval

```php
// Approve transfer
[
    'transfer_id' => 789,
    'approve' => true,
    'notify_both' => true,
    'effective_date' => 'immediate'
]
```

## Data Transfer

### Service Data

```php
// Transfer service data
[
    'service_id' => 1,
    'include_settings' => true,
    'include_custom_fields' => true,
    'include_config_options' => true
]
```

### Module Data

```php
// Transfer module account
[
    'module' => 'cpanel',
    'action' => 'change_owner',
    'service_id' => 1,
    'new_username' => 'newexample'
]
```

## Transfer Validation

### Validation Checks

```php
// Before transfer
[
    'check' => [
        'no_pending_invoices' => true,
        'no_overdue_balance' => true,
        'new_owner_exists' => true,
        'service_active' => true
    ]
]
```

### Block Transfer

```php
// Prevent transfer if
[
    'block_if' => [
        'has_unpaid_invoices' => true,
        'service_suspended' => true,
        'pending_upgrades' => true
    ]
]
```

## Transfer Notification

### Both Parties Notified

```smarty
Subject: Service Transfer Completed

Dear {$client_name},

Service {$service_domain} has been transferred.

{if $is_sender}
Transferred to: {$new_owner}
{/if}

{if $is_receiver}
Transferred from: {$previous_owner}
{/if}

{$company_name}
```

## Transfer History

### Track Transfers

```php
// Transfer log
[
    'service_id' => 1,
    'transfers' => [
        ['date' => '2024-05-15', 'from' => 123, 'to' => 456, 'by' => 'admin']
    ]
]
```

## Reseller Transfer

### Reseller Arrangement

```php
// Transfer between resellers
[
    'service_id' => 1,
    'from_reseller' => 789,
    'to_reseller' => 101,
    'update_module' => true,
    'update_registrant' => true
]
```

## API Functions

```php
// Transfer service
$result = localAPI('TransferService', [
    'serviceid' => 1,
    'newclientid' => 456
]);

// Get transfer eligibility
$result = localAPI('GetServiceTransferStatus', [
    'serviceid' => 1
]);
```

## Hooks

```php
// Hook: ServiceTransferred
add_hook('ServiceTransferred', 1, function($vars) {
    // $vars['serviceid']
    // $vars['from_userid']
    // $vars['to_userid']
    // Update module, notify, etc.
});
```

## Best Practices

1. **Verify ownership**: Confirm transfer authorization
2. **Check balances**: Ensure no outstanding invoices
3. **Document transfer**: Record reason and details
4. **Notify both**: Inform both parties
5. **Update module**: Sync server account ownership

## Related Documentation

- [Client Merging](./whmcs-client-merging.md)
- [Service Creation](./whmcs-service-creation.md)
- [Service Termination](./whmcs-service-termination.md)
- [Service Modification](./whmcs-service-modification.md)