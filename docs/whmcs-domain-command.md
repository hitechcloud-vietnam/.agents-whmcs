# WHMCS Domain Command

## Overview

Domain command handlers in WHMCS execute registrar API commands for domain management operations.

## Command Types

### Standard Commands

| Command | Description |
|---------|-------------|
| Register | Register new domain |
| Transfer | Initiate transfer |
| Renew | Renew domain |
| UpdateNS | Update nameservers |
| Lock | Enable transfer lock |
| Unlock | Disable transfer lock |
| GetEPPCode | Retrieve auth code |

## Command Execution

### Execute Command

```php
// Execute domain command
[
    'module' => 'enom',
    'action' => 'RegisterDomain',
    'domain' => 'example.com',
    'registrant' => [...],
    'years' => 1
]
```

### Command Response

```php
// Response handling
[
    'success' => true,
    'domain_id' => 1,
    'message' => 'Domain registered successfully'
]
```

## Custom Commands

### Registrar-Specific Commands

```php
// Custom module commands
[
    'module' => 'enom',
    'action' => 'custom',
    'command' => 'DomainContacts',
    'params' => ['domain' => 'example.com']
]
```

## Command Logging

### Log All Commands

```php
// Track command execution
[
    'domain_id' => 1,
    'command' => 'Renew',
    'executed_by' => 'admin_id',
    'timestamp' => '2024-05-15 10:30:00',
    'result' => 'success'
]
```

## API Functions

```php
// Execute domain command
$result = localAPI('DomainCommand', [
    'domainid' => 1,
    'command' => 'Renew',
    'params' => ['years' => 1]
]);
```

## Hooks

```php
// Hook: DomainCommandPreExecute
add_hook('DomainCommandPreExecute', 1, function($vars) {
    // $vars['command']
    // Validate command
});

// Hook: DomainCommandPostExecute
add_hook('DomainCommandPostExecute', 1, function($vars) {
    // $vars['command']
    // $vars['result']
    // Log result
});
```

## Best Practices

1. **Log commands**: Track all domain operations
2. **Handle errors**: Process failures gracefully
3. **Validate inputs**: Ensure correct parameters
4. **Test commands**: Verify before production

## Related Documentation

- [Module Prompt Commands](./whmcs-module-prompt.md)
- [Domain Registration](./whmcs-domain-registration.md)
- [Domain Transfer](./whmcs-domain-transfer.md)