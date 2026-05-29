# WHMCS Service Suspension

## Overview

Service suspension in WHMCS temporarily disables a service, preventing client access while preserving all data. Services are typically suspended for non-payment but can also be suspended for other reasons like policy violations.

## Suspension Triggers

### Automatic Suspension

```php
// Auto-suspend on overdue
[
    'auto_suspend' => true,
    'suspend_days_after' => 7,        // Days after due date
    'suspend_reason' => 'Overdue Invoice',
    'suspend_modules' => true
]
```

### Manual Suspension

**Admin: Clients > Services > Suspend**

```php
// Manual service suspension
[
    'service_id' => 1,
    'reason' => 'Policy Violation',
    'suspend_modules' => true,
    'notify_client' => true
]
```

## Suspension Process

### Module Suspension

```php
// Suspend on server
[
    'module' => 'cpanel',
    'action' => 'suspend',
    'service_id' => 1,
    'username' => 'example',
    'reason' => 'Overdue payment',
    'suspend_until' => null           // null = until manually reactivated
]
```

### Suspension Steps

1. Suspend on server (lock account)
2. Update WHMCS service status
3. Send notification email
4. Log suspension event

## Suspended Service Status

### Status Change

```php
// Service status
[
    'service_id' => 1,
    'status' => 'Suspended',
    'suspended_at' => '2024-05-15',
    'suspend_reason' => 'Overdue Invoice',
    'suspend_count' => 1
]
```

### Access Restrictions

```php
// What's disabled
[
    'module_access' => false,
    'cpanel_login' => false,
    'ftp_access' => false,
    'email_access' => false
]
```

## Suspension Duration

### Temporary Suspension

```php
// Suspend for specific period
[
    'service_id' => 1,
    'suspend_until' => '2024-05-20',
    'auto_reactivate' => true
]
```

### Until Reactivated

```php
// Manual reactivation required
[
    'service_id' => 1,
    'suspend_until' => null,
    'require_manual_reactivate' => true
]
```

## Service Reactivation

### Reactivate Service

**Admin: Clients > Services > Reactivate**

```php
// Reactivate suspended service
[
    'service_id' => 1,
    'charge_overdue' => true,
    'reactivate_modules' => true,
    'notify_client' => true
]
```

### Module Reactivation

```php
// Unsuspend on server
[
    'module' => 'cpanel',
    'action' => 'unsuspend',
    'service_id' => 1,
    'username' => 'example'
]
```

## Suspension Notice

### Client Notification

```smarty
Subject: Service Suspended - {$service_domain}

Dear {$client_name},

Your service has been suspended.

Service: {$service_domain}
Reason: {$suspend_reason}

To reactivate:
1. Pay outstanding invoice
2. Contact support

{$company_name}
```

## Multiple Suspension

### Track Suspensions

```php
// Suspension history
[
    'service_id' => 1,
    'suspensions' => [
        ['date' => '2024-05-15', 'reason' => 'Non-payment', 'duration' => 5],
        ['date' => '2024-03-10', 'reason' => 'Non-payment', 'duration' => 3]
    ],
    'total_suspensions' => 2
]
```

## API Functions

```php
// Suspend service
$result = localAPI('SuspendService', [
    'serviceid' => 1,
    'reason' => 'Overdue payment'
]);

// Unsuspend service
$result = localAPI('UnsuspendService', [
    'serviceid' => 1
]);
```

## Hooks

```php
// Hook: ServiceSuspended
add_hook('ServiceSuspended', 1, function($vars) {
    // $vars['serviceid']
    // $vars['reason']
    // Suspend module, notify, etc.
});

// Hook: ServiceUnsuspended
add_hook('ServiceUnsuspended', 1, function($vars) {
    // $vars['serviceid']
    // Reactivate module, notify, etc.
});
```

## Best Practices

1. **Notify clients**: Always inform before suspension
2. **Set clear policies**: Define suspension reasons
3. **Quick reactivation**: Process payments promptly
4. **Track patterns**: Monitor repeat suspensions
5. **Preserve data**: Don't delete during suspension

## Related Documentation

- [Service Reactivation](./whmcs-service-reactivation.md)
- [Service Termination](./whmcs-service-termination.md)
- [Dunning Settings](./whmcs-dunning-settings.md)
- [Service Suspension Override](./whmcs-service-suspension-override.md)