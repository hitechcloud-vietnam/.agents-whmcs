# WHMCS Service Modification

## Overview

Service modification in WHMCS allows changes to existing services including domain changes, resource adjustments, and configuration updates. Modifications can affect billing and require appropriate handling.

## Modification Types

### Domain Change

```php
// Change service domain
[
    'service_id' => 1,
    'old_domain' => 'olddomain.com',
    'new_domain' => 'newdomain.com',
    'update_module' => true,
    'charge_prorate' => false
]
```

### Resource Modification

```php
// Adjust service resources
[
    'service_id' => 1,
    'changes' => [
        ['field' => 'disk_quota', 'old' => '10GB', 'new' => '20GB'],
        ['field' => 'bandwidth', 'old' => '100GB', 'new' => '200GB']
    ],
    'charge_amount' => 5.00,
    'effective_date' => 'immediately'
]
```

## Modification Process

### Request Modification

**Admin: Clients > Services > Modify**

```php
// Modify service
[
    'service_id' => 1,
    'custom_fields' => [...],
    'config_options' => [...],
    'update_module' => true
]
```

### Client Request Modification

**Client Area > Services > Upgrade/Downgrade**

```php
// Client-initiated change
[
    'service_id' => 1,
    'change_type' => 'upgrade',
    'requested_package' => 'premium'
]
```

## Configurable Options

### Add/Remove Options

```php
// Add configurable option
[
    'service_id' => 1,
    'option_id' => 5,
    'add' => true,
    'charge_prorate' => true
]

// Remove configurable option
[
    'service_id' => 1,
    'option_id' => 5,
    'remove' => true,
    'credit_amount' => 2.50
]
```

## Billing Impact

### Price Changes

```php
// Modification pricing
[
    'change_type' => 'upgrade',
    'old_price' => 10.00,
    'new_price' => 15.00,
    'prorate_credit' => 5.00,
    'new_monthly' => 15.00
]
```

### Invoice Generation

```php
// Generate modification invoice
[
    'service_id' => 1,
    'items' => [
        ['description' => 'Resource upgrade', 'amount' => 5.00]
    ],
    'due_date' => 'immediate'
]
```

## Module Update

### Sync Changes to Server

```php
// Update module with changes
[
    'module' => 'cpanel',
    'service_id' => 1,
    'action' => 'modify',
    'changes' => [
        'disk' => '20GB',
        'bandwidth' => '200GB'
    ]
]
```

## Custom Fields

### Update Custom Fields

```php
// Modify custom field values
[
    'service_id' => 1,
    'fields' => [
        ['id' => 1, 'value' => 'new_value'],
        ['id' => 2, 'value' => 'updated']
    ],
    'updated_by' => 'admin_id',
    'updated_at' => '2024-05-15'
]
```

## Modification Approval

### Approval Workflow

```php
// Require approval for changes
[
    'require_approval' => true,
    'approval_threshold' => 50.00,      // Amount requiring approval
    'approvers' => ['admin', 'manager'],
    'auto_approve_under' => 10.00
]
```

### Pending Modifications

```php
// Track pending changes
[
    'modification_id' => 789,
    'service_id' => 1,
    'requested_changes' => [...],
    'status' => 'pending',
    'requested_by' => 'admin_id',
    'requested_at' => '2024-05-15'
]
```

## Notification

### Notify Client

```smarty
Subject: Service Modified - {$service_domain}

Dear {$client_name},

Your service has been updated.

Changes Made:
- Disk: 10GB -> 20GB

Effective: {$effective_date}

{$company_name}
```

## API Functions

```php
// Modify service
$result = localAPI('ModifyService', [
    'serviceid' => 1,
    'customfields' => [...],
    'configoptions' => [...]
]);

// Update service
$result = localAPI('UpdateService', [
    'serviceid' => 1,
    'domain' => 'newdomain.com'
]);
```

## Hooks

```php
// Hook: ServiceModified
add_hook('ServiceModified', 1, function($vars) {
    // $vars['serviceid']
    // $vars['userid']
    // $vars['changes']
    // Update module, notify, etc.
});
```

## Best Practices

1. **Preview changes**: Show billing impact before confirming
2. **Sync modules**: Update server configuration
3. **Document changes**: Log all modifications
4. **Notify clients**: Inform of service changes
5. **Test updates**: Verify modifications work correctly

## Related Documentation

- [Service Creation](./whmcs-service-creation.md)
- [Service Upgrade](./whmcs-service-upgrade.md)
- [Service Downgrade](./whmcs-service-downgrade.md)
- [Configurable Options](./whmcs-product-configurable.md)