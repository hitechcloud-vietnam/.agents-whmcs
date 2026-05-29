# WHMCS Service Actions

## Overview

Service actions in WHMCS are the available operations that can be performed on services, including upgrades, cancellations, and module operations.

## Available Actions

### Standard Actions

| Action | Description |
|--------|-------------|
| upgrade | Upgrade to higher product |
| downgrade | Downgrade to lower product |
| cancel | Request cancellation |
| terminate | Immediate termination |
| suspend | Suspend service |
| unsuspend | Reactivate service |

### Module Actions

| Action | Module Command |
|--------|---------------|
| provision | Create account |
| suspend | Suspend account |
| unsuspend | Unsuspend account |
| terminate | Terminate account |
| change-password | Update password |
| reboot | Reboot server |

## Action Configuration

### Enable/Disable Actions

```php
// Configure available actions
[
    'service_id' => 1,
    'allow_upgrade' => true,
    'allow_downgrade' => true,
    'allow_cancel' => true,
    'allow_suspend' => false
]
```

## Action Execution

### Perform Action

**Admin: Clients > Services > Select Action**

```php
// Execute service action
[
    'service_id' => 1,
    'action' => 'suspend',
    'reason' => 'Non-payment',
    'notify' => true
]
```

## Action Hooks

### Hook Triggers

```php
// Hooks for service actions
add_hook('ServiceUpgrade', 1, function($vars) {
    // Triggered on upgrade
});

add_hook('ServiceCancel', 1, function($vars) {
    // Triggered on cancel
});
```

## Action Permissions

### Role-Based Access

```php
// Who can perform actions
[
    'action' => 'terminate',
    'allowed_roles' => ['admin', 'manager'],
    'require_confirmation' => true
]
```

## API Functions

```php
// Execute service action
$result = localAPI('ExecuteServiceAction', [
    'serviceid' => 1,
    'action' => 'suspend'
]);
```

## Best Practices

1. **Clear permissions**: Control who can perform actions
2. **Log actions**: Track all service changes
3. **Notify clients**: Keep customers informed
4. **Require confirmation**: Confirm destructive actions

## Related Documentation

- [Service Suspension](./whmcs-service-suspension.md)
- [Service Termination](./whmcs-service-termination.md)
- [Service Upgrade](./whmcs-service-upgrade.md)
- [Service Actions](./whmcs-service-actions.md)