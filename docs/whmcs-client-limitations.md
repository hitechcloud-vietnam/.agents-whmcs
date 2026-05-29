# WHMCS Client Limitations

## Overview

Client limitations in WHMCS allow administrators to restrict client access to certain features, products, or functionality. These controls help manage tiered services, control resource usage, and enforce business rules.

## Limitation Types

### Product Limitations

```php
// Limit service capabilities
[
    'product_id' => 1,
    'limitations' => [
        'max_domains' => 5,
        'max_subdomains' => 10,
        'max_email_accounts' => 20,
        'max_databases' => 5,
        'disk_quota_gb' => 10
    ]
]
```

### Feature Limitations

```php
// Control access to features
[
    'feature_limits' => [
        'support_tickets' => 10,           // Per month
        'api_calls' => 1000,               // Per day
        'concurrent_sessions' => 1,
        'file_uploads_mb' => 50            // Per month
    ]
]
```

## Resource Limits

### Usage Quotas

```php
// Define resource quotas
[
    'disk_space' => [
        'limit' => 10000,                  // MB
        'used' => 5000,
        'warning_threshold' => 80,         // percentage
        'action_on_exceed' => 'notify'     // notify, suspend, upgrade_prompt
    ],
    'bandwidth' => [
        'limit' => 500000,                 // MB per month
        'used' => 250000,
        'reset_day' => 1
    ],
    'emails' => [
        'limit' => 1000,                   // Per day
        'used' => 500
    ]
]
```

### Soft vs Hard Limits

```php
// Soft limit - warning only
[
    'disk_soft_limit' => 8000,             // 80%
    'action' => 'send_warning_email'
]

// Hard limit - restrict action
[
    'disk_hard_limit' => 10000,
    'action' => 'block_uploads'
]
```

## Module Command Limits

### Control Module Usage

```php
// Limit what client can do via module
[
    'module_limits' => [
        'cpanel' => [
            'allow_cpanel_access' => false,
            'allow_webmail_access' => true
        ],
        'plesk' => [
            'allow_plesk_access' => false
        ]
    ]
]
```

## Service Limitations

### Package-Based Limits

```php
// Basic package
[
    'package' => 'Basic',
    'max_additional_ftps' => 0,
    'max_email_lists' => 0,
    'max_databases' => 1,
    'max_subdomains' => 0,
    'max_park_domains' => 0
]

// Premium package
[
    'package' => 'Premium',
    'max_additional_ftps' => 10,
    'max_email_lists' => 5,
    'max_databases' => 10,
    'max_subdomains' => 25,
    'max_park_domains' => 10
]
```

## Client Group Limitations

### Group-Based Restrictions

```php
// Limit entire client groups
[
    'group' => 'Trial Users',
    'limitations' => [
        'can_upgrade' => false,
        'can_transfer' => false,
        'max_tickets_per_week' => 2,
        'require_approval_for_purchase' => true
    ]
]
```

## API Access Limits

### Rate Limiting

```php
// API usage limits
[
    'client_api_limits' => [
        'hourly_limit' => 100,
        'daily_limit' => 1000,
        'monthly_limit' => 10000,
        'allow_burst' => true
    ]
]
```

## Order Limitations

### Prevent Specific Orders

```php
// Restrict order placement
[
    'client_id' => 123,
    'order_restrictions' => [
        'cannot_order_product_ids' => [5, 10],
        'cannot_order_category_ids' => [3],
        'require_approval' => true
    ]
]
```

## Limitation Enforcement

### Automatic Enforcement

```php
// Check limits on action
function checkServiceLimit($serviceId, $action, $resource) {
    $limits = getServiceLimits($serviceId);
    $usage = getCurrentUsage($serviceId, $resource);
    
    if ($usage >= $limits[$resource]['max']) {
        return ['allowed' => false, 'reason' => 'Limit exceeded'];
    }
    return ['allowed' => true];
}
```

### When Limits Exceeded

```php
// Actions on limit exceeded
[
    'on_exceed' => [
        'notify_client' => true,
        'notify_admin' => true,
        'block_action' => true,
        'offer_upgrade' => true
    ]
]
```

## Viewing Limitations

### Admin View

**Admin: Clients > Select Client > Limits**

```php
// Client limitation summary
[
    'userid' => 123,
    'services' => [
        ['serviceid' => 1, 'product' => 'Basic', 'limits' => [...]],
        ['serviceid' => 2, 'product' => 'Email', 'limits' => [...]]
    ],
    'usage_summary' => [
        'disk' => '50%',
        'bandwidth' => '30%',
        'emails' => '25%'
    ]
]
```

### Client View

**Client Area > Account > Resource Usage**

```php
// Client sees their usage
[
    'disk' => [
        'used' => 5000,
        'total' => 10000,
        'percentage' => 50
    ],
    'bandwidth' => [
        'used' => 150,
        'total' => 500,
        'percentage' => 30,
        'resets_on' => 'June 1'
    ]
]
```

## Overriding Limits

### Admin Override

```php
// Admin increases limits
[
    'action' => 'increase_limit',
    'serviceid' => 1,
    'resource' => 'disk_quota',
    'new_limit' => 20000,
    'reason' => 'Customer request',
    'temporary' => false,
    'expires' => null
]
```

### Temporary Limits

```php
// Temporary increase
[
    'serviceid' => 1,
    'resource' => 'disk_quota',
    'temporary_limit' => 20000,
    'start_date' => '2024-05-15',
    'end_date' => '2024-06-15',
    'auto_revert' => true
]
```

## Notifications

### Limit Warning Notifications

```php
// Notify when approaching limit
[
    'warning_threshold' => 80,            // percentage
    'warning_template' => 'resource_warning',
    'send_to_client' => true,
    'send_to_admin' => false,
    'frequency' => 'once'                 // once, daily, weekly
]
```

### Template Variables

```smarty
{$resource_name}
{$used_amount}
{$limit_amount}
{$percentage_used}
{$upgrade_link}
```

## API Functions

```php
// Get client limits
$result = localAPI('GetClientLimits', [
    'clientid' => 123
]);

// Update client limit
$result = localAPI('UpdateClientLimit', [
    'clientid' => 123,
    'limit_type' => 'api',
    'daily_limit' => 2000
]);

// Check limit
$result = localAPI('CheckClientLimit', [
    'clientid' => 123,
    'resource' => 'disk'
]);
```

## Best Practices

1. **Clear communication**: Let clients know their limits
2. **Warning systems**: Alert before limits reached
3. **Easy upgrades**: Provide upgrade paths
4. **Document policies**: Explain limitation rules
5. **Regular review**: Adjust limits based on usage

## Related Documentation

- [Client Groups](./whmcs-client-groups.md)
- [Service Management](./whmcs-service-management.md)
- [Product Configuration](./whmcs-product-configuration.md)
- [Usage Billing](./whmcs-usage-billing.md)