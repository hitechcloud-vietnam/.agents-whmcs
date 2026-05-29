# WHMCS Service Terminate Override

## Overview

Service terminate override in WHMCS allows administrators to prevent automatic termination of services under specific conditions.

## Override Configuration

### Disable Auto-Termination

**Admin: Clients > Services > Termination Settings**

```php
// Override termination
[
    'service_id' => 1,
    'override_termination' => true,
    'type' => 'no_auto_terminate',
    'reason' => 'VIP customer',
    'expires' => null
]
```

### Override Types

| Type | Description |
|------|-------------|
| no_auto_terminate | Never auto-terminate |
| extend_termination_grace | Extend termination grace |
| conditional_override | Conditional override |

## Override Settings

### Complete Override

```php
// Never terminate
[
    'service_id' => 1,
    'override_type' => 'complete',
    'reason' => 'Manual billing only'
]
```

### Extended Grace

```php
// Extend termination grace
[
    'service_id' => 1,
    'override_type' => 'extend',
    'extend_days' => 30,
    'new_termination_days' => 60
]
```

## Override Management

### Set Override

```php
// Add termination override
[
    'service_id' => 1,
    'type' => 'no_auto_terminate',
    'reason' => 'Special arrangement',
    'created_by' => 'admin_id'
]
```

### Remove Override

```php
// Remove override
[
    'service_id' => 1,
    'action' => 'remove_override',
    'restore_termination_rules' => true
]
```

## Override Expiration

### Temporary Override

```php
// Override with end date
[
    'service_id' => 1,
    'end_date' => '2024-12-31',
    'auto_remove' => true,
    'notify_before' => true
]
```

## Override Reporting

### Track Overrides

```php
// Override usage
[
    'total_overrides' => 15,
    'active' => 10,
    'expired' => 5
]
```

## API Functions

```php
// Set termination override
$result = localAPI('SetTerminationOverride', [
    'serviceid' => 1,
    'type' => 'no_auto_terminate'
]);

// Remove override
$result = localAPI('RemoveTerminationOverride', [
    'serviceid' => 1
]);
```

## Hooks

```php
// Hook: TerminationOverrideSet
add_hook('TerminationOverrideSet', 1, function($vars) {
    // $vars['serviceid']
    // $vars['type']
});
```

## Best Practices

1. **Document reasons**: Record why override was set
2. **Set expiration**: Use temporary overrides
3. **Review periodically**: Audit active overrides
4. **Consider revenue**: Monitor impact on collections

## Related Documentation

- [Service Termination](./whmcs-service-termination.md)
- [Service Suspension Override](./whmcs-service-suspension-override.md)
- [Dunning Settings](./whmcs-dunning-settings.md)
- [Service Override](./whmcs-service-override.md)