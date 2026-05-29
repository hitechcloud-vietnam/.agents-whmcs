# WHMCS Client Filtering

## Overview

Client filtering in WHMCS allows administrators to segment and view client lists based on various criteria. Filters help manage large client databases by showing only relevant results.

## Filter Configuration

### Enable Filters

**Admin: Configuration > General > Client Settings**

```php
[
    'enable_filters' => true,
    'default_filter' => 'all',
    'remember_last_filter' => true,
    'show_filter_counts' => true
]
```

## Basic Filters

### Status Filter

```php
// Filter by client status
[
    'field' => 'status',
    'options' => [
        'Active',
        'Inactive',
        'Cancelled',
        'Suspended'
    ],
    'default' => null
]
```

### Group Filter

```php
// Filter by client group
[
    'field' => 'group',
    'options' => [
        'Default',
        'Premium',
        'VIP',
        'Trial'
    ],
    'allow_multiple' => true
]
```

## Advanced Filters

### Date Filters

```php
// Created date filter
[
    'field' => 'created',
    'type' => 'date_range',
    'presets' => [
        'today',
        'this_week',
        'this_month',
        'this_year',
        'custom'
    ]
]

// Last activity filter
[
    'field' => 'last_activity',
    'type' => 'relative',
    'options' => [
        'last_7_days',
        'last_30_days',
        'last_90_days',
        'last_year',
        'never'
    ]
]
```

### Financial Filters

```php
// Total spent filter
[
    'field' => 'total_spent',
    'type' => 'range',
    'min' => 0,
    'max' => 100000,
    'step' => 100
]

// Credit balance filter
[
    'field' => 'credit_balance',
    'type' => 'comparison',
    'operators' => ['=', '>', '<', '>=', '<=']
]

// Invoice balance filter
[
    'field' => 'invoice_balance',
    'type' => 'boolean',
    'options' => ['with_balance', 'no_balance', 'overdue']
]
```

### Geographic Filters

```php
// Country filter
[
    'field' => 'country',
    'type' => 'select',
    'options' => ['list of all countries'],
    'allow_multiple' => true
]

// State/Region filter
[
    'field' => 'state',
    'type' => 'text',
    'depends_on' => 'country'
]
```

## Service-Based Filters

### Has Services Filter

```php
// Filter by service status
[
    'field' => 'has_services',
    'type' => 'boolean',
    'options' => [
        'any',
        'active_only',
        'suspended_only',
        'none'
    ]
]
```

### Service Type Filter

```php
// Filter by product/service type
[
    'field' => 'has_product',
    'type' => 'multi_select',
    'options' => [
        'Web Hosting',
        'Reseller Hosting',
        'VPS',
        'Dedicated Server',
        'SSL Certificate'
    ]
]
```

## Tag-Based Filtering

### Tag Filter

```php
// Filter by client tags
[
    'field' => 'tag',
    'type' => 'multi_select',
    'options' => ['vip', 'enterprise', 'small_business', 'prospect'],
    'match' => 'any'      // any, all
]
```

## Custom Field Filters

### Custom Field Filters

```php
// Filter by custom fields
[
    'field' => 'custom_client_type',
    'type' => 'select',
    'options' => ['Individual', 'Business', 'Enterprise']
]

[
    'field' => 'account_manager',
    'type' => 'select',
    'options' => ['Admin 1', 'Admin 2', 'Admin 3']
]
```

## Combined Filters

### Multiple Filter Conditions

```php
// Combine filters with AND
[
    'filters' => [
        ['field' => 'status', 'value' => 'Active'],
        ['field' => 'group', 'value' => 'Premium'],
        ['field' => 'total_spent', 'operator' => '>=', 'value' => 1000]
    ],
    'match_type' => 'all'          // all, any
]
```

### Filter Groups

```php
// OR conditions within group
[
    'filter_groups' => [
        [
            'match' => 'any',
            'conditions' => [
                ['field' => 'group', 'value' => 'Premium'],
                ['field' => 'group', 'value' => 'VIP']
            ]
        ],
        [
            'match' => 'all',
            'conditions' => [
                ['field' => 'status', 'value' => 'Active'],
                ['field' => 'total_spent', 'operator' => '>=', 'value' => 1000]
            ]
        ]
    ]
]
```

## Saved Filters

### Create Saved Filter

```php
// Save filter for reuse
[
    'name' => 'High Value Premium Clients',
    'filters' => [...],
    'sort' => ['field' => 'total_spent', 'order' => 'desc'],
    'created_by' => 'admin_id',
    'shared' => false
]
```

### Use Saved Filter

```php
// Apply saved filter
[
    'filter_id' => 123,
    'name' => 'High Value Premium Clients'
]
```

## Filter Display

### Show Active Filters

```php
// Display current filters
[
    'active_filters' => [
        ['field' => 'Status', 'value' => 'Active'],
        ['field' => 'Group', 'value' => 'Premium']
    ],
    'clear_all' => true,
    'clear_individual' => true
]
```

### Results Count

```php
// Show filtered results count
[
    'total_clients' => 1000,
    'filtered_count' => 150,
    'percentage' => 15
]
```

## Filter Presets

### System Presets

```php
// Built-in filter presets
[
    'recent' => ['created' => 'last_30_days'],
    'high_value' => ['total_spent' => '>=1000'],
    'at_risk' => ['last_activity' => '>90_days'],
    'never_logged_in' => ['last_login' => 'never']
]
```

## API Functions

```php
// Get filtered clients
$result = localAPI('GetClients', [
    'filters' => [
        'status' => 'Active',
        'group' => 'Premium',
        'created' => '>=2024-01-01'
    ]
]);

// Save filter
$result = localAPI('SaveClientFilter', [
    'name' => 'Premium Clients',
    'filters' => [...]
]);
```

## Best Practices

1. **Use specific filters**: Narrow results to relevant clients
2. **Combine filters**: Use multiple criteria for precise filtering
3. **Save common filters**: Create saved filters for frequent use
4. **Review counts**: Check result counts match expectations
5. **Clear unused filters**: Remove unnecessary filters

## Related Documentation

- [Client Search](./whmcs-client-search.md)
- [Client Sorting](./whmcs-client-sorting.md)
- [Client Tags](./whmcs-client-tags.md)
- [Client Groups](./whmcs-client-groups.md)