# WHMCS Module Prompt Commands

## Overview

Module prompt commands in WHMCS are used to interact with server automation modules. These commands communicate between WHMCS and the server control panels (cPanel, Plesk, DirectAdmin, etc.).

## Command Structure

### Basic Format

```php
// Module command structure
[
    'module' => 'cpanel',
    'action' => 'create',
    'service_id' => 1,
    'username' => 'example',
    'domain' => 'example.com',
    'password' => 'SecurePass123!'
]
```

### Command Response

```php
// Response format
[
    'success' => true,
    'output' => 'Account created successfully',
    'raw_output' => '...'
]
```

## Standard Commands

### Create Account

```php
// Provision new account
[
    'action' => 'CreateAccount',
    'username' => 'example',
    'domain' => 'example.com',
    'password' => 'SecurePass123!',
    'plan' => 'default',
    'email' => 'admin@example.com'
]
```

### Suspend Account

```php
// Suspend account
[
    'action' => 'SuspendAccount',
    'username' => 'example',
    'reason' => 'Overdue payment'
]
```

### Unsuspend Account

```php
// Reactivate account
[
    'action' => 'UnsuspendAccount',
    'username' => 'example'
]
```

### Terminate Account

```php
// Remove account
[
    'action' => 'TerminateAccount',
    'username' => 'example'
]
```

### Change Password

```php
// Update password
[
    'action' => 'ChangePassword',
    'username' => 'example',
    'password' => 'NewSecurePass123!'
]
```

## Module Command Types

### Account Commands

| Command | Description |
|---------|-------------|
| CreateAccount | Provision new account |
| SuspendAccount | Suspend account |
| UnsuspendAccount | Reactivate account |
| TerminateAccount | Remove account |
| ChangePassword | Update password |

### Information Commands

| Command | Description |
|---------|-------------|
| GetDetails | Get account details |
| GetDiskUsage | Get disk usage |
| GetBandwidthUsage | Get bandwidth usage |
| GetPasswordStrength | Check password strength |

### Management Commands

| Command | Description |
|---------|-------------|
| ChangePackage | Upgrade/downgrade package |
| ChangeUsername | Change account username |
| ChangeDomain | Change account domain |
| RestartServices | Restart services |

## Module Response Handling

### Success Response

```php
// Successful command
[
    'success' => true,
    'output' => 'Account created successfully'
]
```

### Error Response

```php
// Command failed
[
    'success' => false,
    'error' => 'Username already exists'
]
```

## Custom Commands

### Module-Specific Commands

```php
// cPanel-specific
[
    'action' => 'custom',
    'cmd' => 'setupdns',
    'params' => [...]
]

// Plesk-specific
[
    'action' => 'custom',
    'cmd' => 'create_subscription',
    'params' => [...]
]
```

## Command Logging

### Log All Commands

```php
// Record command execution
[
    'module' => 'cpanel',
    'action' => 'create',
    'service_id' => 1,
    'result' => 'success',
    'timestamp' => '2024-05-15 10:30:00'
]
```

## API Functions

```php
// Execute module command
$result = localAPI('ModuleCommand', [
    'serviceid' => 1,
    'action' => 'SuspendAccount'
]);
```

## Hooks

```php
// Hook: ModuleCommandPreExecute
add_hook('ModuleCommandPreExecute', 1, function($vars) {
    // $vars['module']
    // $vars['action']
    // $vars['serviceid']
});

// Hook: ModuleCommandExecuted
add_hook('ModuleCommandExecuted', 1, function($vars) {
    // $vars['module']
    // $vars['action']
    // $vars['result']
});
```

## Best Practices

1. **Test commands**: Verify module functionality
2. **Log results**: Track all command executions
3. **Handle errors**: Process failure responses
4. **Timeout settings**: Configure appropriate timeouts
5. **Retry logic**: Handle transient failures

## Related Documentation

- [Module Configuration](./whmcs-module-configuration.md)
- [Service Creation](./whmcs-service-creation.md)
- [Service Suspension](./whmcs-service-suspension.md)
- [Service Termination](./whmcs-service-termination.md)