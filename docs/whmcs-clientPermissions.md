# WHMCS Client Permissions

## Overview

Client permissions in WHMCS control what actions and information clients can access. Permissions can be set at the account level, service level, and contact level.

## Permission Types

### Account-Level Permissions

```php
// Core client permissions
[
    'view_services' => true,
    'view_invoices' => true,
    'view_domains' => true,
    'view_tickets' => true,
    'create_tickets' => true,
    'pay_invoices' => true,
    'update_profile' => true,
    'manage_contacts' => true
]
```

### Service Permissions

```php
// Per-service permissions
[
    'service_id' => 1,
    'allow_upgrade' => true,
    'allow_downgrade' => true,
    'allow_cancel' => false,
    'allow_config' => true
]
```

## Permission Levels

### Full Access

```php
// Full client access
[
    'services' => 'full',
    'invoices' => 'full',
    'domains' => 'full',
    'tickets' => 'full',
    'billing' => 'full'
]
```

### Limited Access

```php
// View-only access
[
    'services' => 'view',
    'invoices' => 'view',
    'domains' => 'view',
    'tickets' => 'view',
    'billing' => 'none'
]
```

## Product-Based Permissions

### Module Permissions

```php
// Control module access
[
    'service_id' => 1,
    'module_permissions' => [
        'cpanel' => ['access' => false],
        'plesk' => ['access' => true, 'features' => ['dns', 'email']]
    ]
]
```

## Contact Permissions

### Subaccount Permissions

```php
// Contact permission levels
[
    'contact_id' => 789,
    'permissions' => [
        'services' => ['view' => true, 'manage' => false],
        'domains' => ['view' => true, 'manage' => false],
        'billing' => ['view' => false, 'manage' => false]
    ]
]
```

## Permission Management

### Update Permissions

```php
// Modify client permissions
[
    'userid' => 123,
    'permissions' => [
        'can_upgrade' => true,
        'can_cancel' => true,
        'can_impersonate' => false
    ]
]
```

### Group-Based Defaults

```php
// Permissions inherited from group
[
    'group_id' => 1,
    'default_permissions' => [
        'services' => 'full',
        'invoices' => 'view',
        'tickets' => 'full'
    ]
]
```

## API Functions

```php
// Get client permissions
$result = localAPI('GetClientPermissions', [
    'clientid' => 123
]);

// Update permissions
$result = localAPI('UpdateClientPermissions', [
    'clientid' => 123,
    'permissions' => ['can_upgrade' => true]
]);
```

## Best Practices

1. **Principle of least privilege**: Grant minimum required access
2. **Document changes**: Track permission modifications
3. **Regular audit**: Review permissions periodically
4. **Group-based defaults**: Use groups for consistent permissions
5. **Service-specific**: Control per-service permissions

## Related Documentation

- [Client Groups](./whmcs-client-groups.md)
- [Client Contacts](./whmcs-client-contacts.md)
- [Service Permissions](./whmcs-service-permissions.md)
- [Client Portal](./whmcs-client-portal.md)