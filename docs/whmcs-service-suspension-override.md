# WHMCS Service Suspension Override

## Overview

Service suspension override in WHMCS allows administrators to prevent certain services from being automatically suspended when configured conditions are met.

## Override Configuration

### Disable Auto-Suspension

**Admin: Clients > Services > Suspension Settings**

```php
// Override suspension
[
    'service_id' => 1,
    'override_suspension' => true,
    'reason' => 'VIP customer - manual billing',
    'expires' => null
]
```

### Override Settings

| Setting | Description |
|---------|-------------|
| no_auto_suspend | Prevent auto suspension |
| manual_suspend_only | Only manual suspension |
| extend_grace | Extend grace period |

## Suspension Override Types

### Complete Override

```php
// Never auto-suspend
[
    'service_id' => 1,
    'override_type' => 'complete',
    'reason' => 'Always manually handled'
]
```

### Extended Grace

```php
// Extend suspension grace period
[
    'service_id' => 1,
    'override_type' => 'extend_grace',
    'extend_days' => 14,
    'new_grace_days' => 28             // Instead of 14
]
```

### Conditional Override

```php
// Override under conditions
[
    'service_id' => 1,
    'override_type' => 'conditional',
    'conditions' => [
        'invoice_overdue_days' => '>=30',
        'client_group' => 'premium'
    ]
]
```

## Override Management

### Set Override

```php
// Add suspension override
[
    'service_id' => 1,
    'type' => 'complete',
    'reason' => 'VIP customer',
    'created_by' => 'admin_id',
    'created_at' => '2024-05-15'
]
```

### Update Override

```php
// Modify override
[
    'service_id' => 1,
    'extend_days' => 7,
    'reason' => 'Extended due to payment plan'
]
```

### Remove Override

```php
// Remove override
[
    'service_id' => 1,
    'action' => 'remove_override',
    'effective' => 'immediate',
    'notify_admin' => true
]
```

## Override Expiration

### Temporary Override

```php
// Override with end date
[
    'service_id' => 1,
    'override_type' => 'complete',
    'start_date' => '2024-05-01',
    'end_date' => '2024-06-30',
    'auto_remove' => true
]
```

### Auto-Restore

```php
// After expiration
[
    'auto_restore' => true,
    'restore_suspension_rules' => true,
    'notify_before' => true,
    'notify_days' => 3
]
```

## Client Group Override

### Group-Level Override

```php
// Override for entire group
[
    'group_id' => 1,
    'override_type' => 'extend_grace',
    'extend_days' => 7,
    'reason' => 'Premium group privileges'
]
```

## Override Reporting

### Suspension Override Report

**Reports > Services > Suspension Override Report**

```php
// Report data
[
    'period' => 'May 2024',
    'total_overrides' => 25,
    'active_overrides' => 15,
    'expired_overrides' => 10
]
```

## API Functions

```php
// Set suspension override
$result = localAPI('SetSuspensionOverride', [
    'serviceid' => 1,
    'type' => 'complete'
]);

// Remove override
$result = localAPI('RemoveSuspensionOverride', [
    'serviceid' => 1
]);
```

## Hooks

```php
// Hook: SuspensionOverrideSet
add_hook('SuspensionOverrideSet', 1, function($vars) {
    // $vars['serviceid']
    // $vars['type']
});

// Hook: SuspensionOverrideRemoved
add_hook('SuspensionOverrideRemoved', 1, function($vars) {
    // $vars['serviceid']
    // Suspension rules now apply
});
```

## Best Practices

1. **Clear reasons**: Document why override was set
2. **Set expiration**: Use temporary overrides
3. **Review regularly**: Audit active overrides
4. **Fair application**: Apply consistently
5. **Monitor abuse**: Track override usage

## Related Documentation

- [Service Suspension](./whmcs-service-suspension.md)
- [Service Suspension Override](./whmcs-service-suspension-override.md)
- [Dunning Settings](./whmcs-dunning-settings.md)
- [Service Override](./whmcs-service-override.md)