# WHMCS Service Custom Commands

## Overview

Service custom commands in WHMCS allow administrators to execute module-specific commands directly on services for advanced management.

## Custom Command Execution

### Execute Command

**Admin: Clients > Services > Custom Commands**

```php
// Execute custom command
[
    'service_id' => 1,
    'module' => 'cpanel',
    'command' => 'restartapache',
    'parameters' => []
]
```

### Command Types

| Command | Description |
|---------|-------------|
| Custom | Module-specific commands |
| Module Function | Direct module function call |
| API | Server API call |

## Command Configuration

### Define Custom Commands

```php
// In module configuration
[
    'custom_commands' => [
        'restartapache' => ['description' => 'Restart Apache'],
        'reboot' => ['description' => 'Reboot Server'],
        'updatebackup' => ['description' => 'Update Backup Config']
    ]
]
```

## Command Execution

### Execute Command

```php
// Execute custom command
[
    'service_id' => 1,
    'command' => 'restartapache',
    'reason' => 'Apache not responding'
]
```

### Response Handling

```php
// Command response
[
    'success' => true,
    'output' => 'Apache restarted successfully',
    'executed_at' => '2024-05-15 10:30:00'
]
```

## Security

### Command Permissions

```php
// Require permission to execute
[
    'command' => 'reboot',
    'require_permission' => 'admin',
    'log_all_executions' => true
]
```

## Command Logging

### Log Executions

```php
// Track command usage
[
    'service_id' => 1,
    'command' => 'restartapache',
    'executed_by' => 'admin_id',
    'executed_at' => '2024-05-15 10:30:00',
    'result' => 'success'
]
```

## API Functions

```php
// Execute custom command
$result = localAPI('ExecuteCustomCommand', [
    'serviceid' => 1,
    'command' => 'restartapache'
]);
```

## Best Practices

1. **Log all commands**: Track execution history
2. **Require permissions**: Limit who can execute
3. **Test commands**: Verify before production
4. **Document commands**: Note what each does

## Related Documentation

- [Module Prompt Commands](./whmcs-module-prompt.md)
- [Service Management](./whmcs-service-management.md)
- [Module Configuration](./whmcs-module-configuration.md)
- [Service Actions](./whmcs-service-actions.md)