# WHMCS Client Tags

## Overview

Client tags in WHMCS provide a flexible way to categorize and label clients beyond standard groups. Tags allow for multiple categorization and more granular client management.

## Tag Configuration

### Enable Tags

**Configuration > General > Tag Settings**

```php
[
    'enable_tags' => true,
    'allow_multiple' => true,
    'allow_admin_tags' => true,
    'allow_client_tags' => false,
    'max_tags_per_client' => 10
]
```

## Creating Tags

### Tag Management

**Admin: Configuration > Tags**

### Add New Tag

```php
// Create new tag
[
    'name' => 'Enterprise',
    'color' => '#e74c3c',
    'description' => 'Enterprise level clients',
    'auto_apply' => false
]
```

### Tag Parameters

| Parameter | Description |
|-----------|-------------|
| name | Tag name (required) |
| color | Tag color for display |
| description | Tag description |
| auto_rules | Auto-assignment rules |

## Assigning Tags

### Manual Assignment

**Admin: Clients > Select Client > Tags**

```php
// Assign tags to client
[
    'userid' => 123,
    'tags' => ['enterprise', 'referral', 'high_priority'],
    'assigned_by' => 'admin_id',
    'assigned_at' => '2024-05-15'
]
```

### Bulk Tag Assignment

```php
// Assign tags to multiple clients
[
    'action' => 'bulk_add_tags',
    'client_ids' => [123, 456, 789],
    'tags' => ['vip'],
    'assigned_by' => 'admin_id'
]
```

### Remove Tags

```php
// Remove tag from client
[
    'userid' => 123,
    'tag' => 'trial',
    'removed_by' => 'admin_id',
    'removed_at' => '2024-05-15'
]
```

## Tag-Based Automation

### Auto-Assign Rules

```php
// Automatically assign tags based on rules
[
    'rules' => [
        [
            'condition' => 'total_spent >= 10000',
            'assign_tag' => 'high_value'
        ],
        [
            'condition' => 'created_days <= 30 AND total_spent = 0',
            'assign_tag' => 'new_prospect'
        ]
    ]
]
```

### Tag-Based Actions

```php
// Trigger actions based on tags
[
    'trigger' => 'tag_added',
    'tag' => 'enterprise',
    'actions' => [
        ['action' => 'send_email', 'template' => 'enterprise_welcome'],
        ['action' => 'assign_account_manager', 'admin_id' => 1],
        ['action' => 'apply_discount', 'percentage' => 10]
    ]
]
```

## Tag Filtering

### Filter by Tag

```php
// Filter clients by tag
[
    'tag' => 'enterprise',
    'match' => 'any'    // any, all
]
```

### Multiple Tag Filter

```php
// Filter by multiple tags
[
    'tags' => ['enterprise', 'referral'],
    'match' => 'all'    // Client must have ALL listed tags
]
```

### Exclude by Tag

```php
// Exclude clients with tag
[
    'exclude_tag' => 'cancelled'
]
```

## Tag Reports

### Tag Statistics

**Reports > Clients > Tag Report**

```php
// Tag usage report
[
    'tags' => [
        'enterprise' => ['count' => 50, 'total_spent' => 500000],
        'vip' => ['count' => 100, 'total_spent' => 250000],
        'referral' => ['count' => 75, 'total_spent' => 75000]
    ]
]
```

### Tag Activity Report

```php
// Tag assignment history
[
    'period' => 'May 2024',
    'tags_added' => 150,
    'tags_removed' => 50,
    'net_change' => 100
]
```

## Tag Display

### Client Summary

```php
// Tags shown in client summary
[
    'userid' => 123,
    'tags' => [
        ['name' => 'Enterprise', 'color' => '#e74c3c'],
        ['name' => 'Referral', 'color' => '#3498db']
    ]
]
```

### Template Display

```smarty
// Display tags in templates
{foreach $client.tags as $tag}
    <span class="tag" style="background-color: {$tag.color}">
        {$tag.name}
    </span>
{/foreach}
```

## Tag Management

### Edit Tag

```php
// Update tag
[
    'tag_id' => 1,
    'name' => 'Enterprise Plus',
    'color' => '#9b59b6',
    'description' => 'Updated description'
]
```

### Delete Tag

```php
// Delete tag
[
    'tag_id' => 1,
    'remove_from_all_clients' => true,
    'confirm' => true
]
```

### Merge Tags

```php
// Merge duplicate tags
[
    'source_tag' => 'enterpise',    // Typo
    'target_tag' => 'enterprise',
    'merge_action' => 'combine'
]
```

## Client Tag Management

### Client Tag View

**Client Area > Account > Tags** (if enabled)

```php
// Client can view their tags
[
    'tags' => ['preferred_customer'],
    'can_remove' => false,
    'description' => 'Contact preference: Email'
]
```

## API Functions

```php
// Add tag to client
$result = localAPI('AddClientTag', [
    'clientid' => 123,
    'tag' => 'vip'
]);

// Remove tag
$result = localAPI('RemoveClientTag', [
    'clientid' => 123,
    'tag' => 'vip'
]);

// Get client tags
$result = localAPI('GetClientTags', [
    'clientid' => 123
]);

// Get all tags
$result = localAPI('GetTags');

// Get clients by tag
$result = localAPI('GetClientsByTag', [
    'tag' => 'enterprise'
]);
```

## Hooks

```php
// Hook: ClientTagAdded
add_hook('ClientTagAdded', 1, function($vars) {
    // $vars['userid']
    // $vars['tag']
});

// Hook: ClientTagRemoved
add_hook('ClientTagRemoved', 1, function($vars) {
    // $vars['userid']
    // $vars['tag']
});
```

## Best Practices

1. **Consistent naming**: Use clear, consistent tag names
2. **Color coding**: Use colors to indicate tag categories
3. **Limited tags**: Don't create too many overlapping tags
4. **Regular review**: Audit tags periodically
5. **Use automation**: Set up auto-assignment rules

## Related Documentation

- [Client Groups](./whmcs-client-groups.md)
- [Client Filtering](./whmcs-client-filtering.md)
- [Client Tags](./whmcs-client-tags.md)
- [Client Search](./whmcs-client-search.md)