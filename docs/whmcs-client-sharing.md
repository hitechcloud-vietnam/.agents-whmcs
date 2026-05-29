# WHMCS Client Sharing

## Overview

Client sharing in WHMCS allows certain client data or services to be shared between related accounts. This feature supports reseller models, corporate structures, and partnership arrangements.

## Sharing Types

### Data Sharing

```php
// Share client data
[
    'share_type' => 'profile',
    'share_with' => [456, 789],
    'shared_fields' => ['name', 'company']
]
```

### Service Sharing

```php
// Share service access
[
    'service_id' => 1,
    'shared_with' => [456],
    'permission_level' => 'view'
]
```

## Sharing Configuration

### Enable Sharing

**Configuration > Security > Client Sharing**

```php
[
    'allow_sharing' => true,
    'require_approval' => true,
    'max_shared_accounts' => 10
]
```

## Sharing Rules

### Define Sharing Rules

```php
// Create sharing rule
[
    'userid' => 123,
    'share_type' => 'services',
    'target_userid' => 456,
    'permissions' => ['view', 'upgrade'],
    'created_by' => 'admin_id'
]
```

## Reseller Sharing

### Reseller Access

```php
// Reseller sees customer data
[
    'reseller_id' => 789,
    'customer_ids' => [123, 456, 789],
    'access_level' => 'read_only'
]
```

## Sharing Permissions

### Permission Levels

| Level | Description |
|-------|-------------|
| view | View shared data only |
| manage | Modify shared data |
| billing | Access billing information |
| full | Complete access |

## Revoke Sharing

### Remove Sharing

```php
// Revoke sharing access
[
    'share_id' => 123,
    'userid' => 123,
    'target_userid' => 456,
    'revoked_by' => 'admin_id',
    'revoked_at' => '2024-05-15'
]
```

## API Functions

```php
// Create sharing
$result = localAPI('CreateClientSharing', [
    'userid' => 123,
    'target_userid' => 456,
    'share_type' => 'services'
]);

// Get shared accounts
$result = localAPI('GetClientSharing', [
    'clientid' => 123
]);

// Revoke sharing
$result = localAPI('RevokeClientSharing', [
    'shareid' => 123
]);
```

## Best Practices

1. **Limit sharing**: Only share necessary data
2. **Clear permissions**: Define access levels clearly
3. **Regular review**: Audit sharing relationships
4. **Document agreements**: Note sharing purposes
5. **Revoke unused**: Remove stale sharing

## Related Documentation

- [Client Relationships](./whmcs-client-relationships.md)
- [Client Permissions](./whmcs-clientPermissions.md)
- [Reseller Documentation](./whmcs-reseller-documentation.md)
- [Client Groups](./whmcs-client-groups.md)