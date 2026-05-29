# WHMCS Domain Transfer Lock

## Overview

Domain transfer lock in WHMCS is a security feature that prevents unauthorized domain transfers by locking the domain at the registry level.

## Lock Configuration

### Enable/Disable Lock

**Admin: Clients > Domains > Domain Settings**

```php
// Domain transfer lock
[
    'domain_id' => 1,
    'transfer_lock' => true,
    'locked_by' => 'admin_id',
    'locked_at' => '2024-05-15'
]
```

## Lock Types

### Registry Lock

```php
// Registry-level lock
[
    'type' => 'registry',
    'registrar_lock' => true,
    'description' => 'Locked at registry level - highest security'
]
```

### Registrar Lock

```php
// Registrar lock
[
    'type' => 'registrar',
    'registrar_lock' => true,
    'change_requires_auth' => true
]
```

## Lock Management

### Enable Lock

```php
// Lock domain
[
    'domain_id' => 1,
    'action' => 'lock',
    'reason' => 'Security - prevent unauthorized transfer',
    'locked_by' => 'admin_id'
]
```

### Disable Lock

```php
// Unlock domain for transfer
[
    'domain_id' => 1,
    'action' => 'unlock',
    'reason' => 'Customer requested transfer',
    'unlocked_by' => 'admin_id',
    'temporary_unlock' => true,
    'auto_relock_days' => 7
]
```

## Lock Status

### Current Lock Status

```php
// Check lock status
[
    'domain_id' => 1,
    'domain' => 'example.com',
    'transfer_lock' => true,
    'locked' => true,
    'locked_at' => '2024-05-15',
    'locked_by' => 'admin_id'
]
```

## Transfer Unlock

### Temporary Unlock

```php
// Unlock for transfer
[
    'domain_id' => 1,
    'temporary_unlock' => true,
    'unlock_duration_days' => 7,
    'auto_lock_after' => true,
    'notify_on_auto_lock' => true
]
```

### Unlock for Customer

```php
// Customer-initiated unlock
[
    'domain_id' => 1,
    'customer_request' => true,
    'verify_ownership' => true,
    'unlock_period' => 7,
    'notify_when_unlocked' => true
]
```

## Lock Security

### Security Settings

```php
// Lock security options
[
    'require_verification' => true,
    'verification_method' => 'email',
    'verification_timeout' => 24,            // hours
    'log_unlock_requests' => true,
    'notify_admin_on_unlock' => true
]
```

## Module Integration

### Registrar Lock Commands

```php
// Lock via module
[
    'module' => 'enom',
    'action' => 'LockDomain',
    'domain' => 'example.com'
]

// Unlock via module
[
    'module' => 'enom',
    'action' => 'UnlockDomain',
    'domain' => 'example.com'
]
```

## API Functions

```php
// Lock domain
$result = localAPI('LockDomain', [
    'domainid' => 1
]);

// Unlock domain
$result = localAPI('UnlockDomain', [
    'domainid' => 1
]);

// Get lock status
$result = localAPI('GetDomainLockStatus', [
    'domainid' => 1
]);
```

## Hooks

```php
// Hook: DomainLockChanged
add_hook('DomainLockChanged', 1, function($vars) {
    // $vars['domainid']
    // $vars['locked']
    // $vars['changed_by']
});
```

## Best Practices

1. **Lock by default**: Enable lock on new domains
2. **Verify requests**: Confirm unlock requests
3. **Temporary unlock**: Use time-limited unlocks
4. **Auto-relock**: Re-lock after transfer window
5. **Log changes**: Track all lock/unlock events

## Related Documentation

- [Domain Transfer](./whmcs-domain-transfer.md)
- [Domain Registration](./whmcs-domain-registration.md)
- [EPP Code](./whmcs-domain-epp-code.md)
- [Domain Sync](./whmcs-domain-sync.md)