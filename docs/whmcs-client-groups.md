# WHMCS Client Groups

## Overview

Client groups in WHMCS allow you to categorize and segment your customer base. Groups enable differentiated pricing, targeted communications, access controls, and workflow management based on customer characteristics.

## Accessing Client Groups

**Admin: Configuration > Support > Client Groups**

## Default Groups

### Pre-Configured Groups

| Group | Description | Color |
|-------|-------------|-------|
| Default | Standard clients | Blue |
| Active | Currently active clients | Green |
| Inactive | No recent activity | Gray |
| Cancelled | Cancelled accounts | Red |

### Custom Default Groups

```php
// Create default group
[
    'name' => 'Standard',
    'color' => '#3498db',
    'add_to_all' => true,
    'sort_order' => 1
]
```

## Creating Client Groups

### Basic Group Creation

**Configuration > Support > Client Groups > Add New**

```php
// New client group
[
    'name' => 'Premium',
    'color' => '#f39c12',
    'group_type' => 'standard',
    'sort_order' => 10,
    'discount' => 15.00,         // 15% discount
    'suspension_threshold' => null
]
```

### Group Parameters

| Parameter | Description |
|-----------|-------------|
| name | Group display name |
| color | Admin panel color coding |
| discount | Automatic discount percentage |
| late_fee_exempt | Exempt from late fees |
| separate_invoices | Separate invoices per product |

## Group Discounts

### Automatic Discount Configuration

```php
// Group-level discount
[
    'group_name' => 'Premium Clients',
    'discount_type' => 'percentage',   // percentage, fixed
    'discount_value' => 15.00,
    'apply_to' => 'products',         // products, services, addons, all
    'exclude_on_sale' => true
]
```

### Discount Application

```php
// Product with base price $100
// Premium client (15% discount)
// Final price: $85.00

[
    'product_price' => 100.00,
    'group_discount' => 15.00,
    'final_price' => 85.00
]
```

## Group-Based Rules

### Access Controls

```php
// Group access restrictions
[
    'group_name' => 'Enterprise',
    'allow_feature_1' => true,
    'allow_feature_2' => true,
    'max_services' => 100,
    'allow_trial_products' => true
]

// Standard group restrictions
[
    'allow_feature_1' => false,
    'allow_feature_2' => true,
    'max_services' => 10,
    'allow_trial_products' => false
]
```

### Auto-Assignment Rules

```php
// Automatic group assignment
[
    'rules' => [
        [
            'condition' => 'country',
            'operator' => 'equals',
            'value' => 'US',
            'assign_group' => 'us_customers'
        ],
        [
            'condition' => 'total_spent',
            'operator' => '>=',
            'value' => 1000,
            'assign_group' => 'vip_customers'
        ]
    ]
]
```

## Group-Specific Pricing

### Product Pricing by Group

**Products > Pricing > Group-Specific Pricing**

```php
// Different pricing per group
[
    'product_id' => 1,
    'group_pricing' => [
        'default' => ['monthly' => 10.00],
        'premium' => ['monthly' => 8.50],
        'vip' => ['monthly' => 7.00]
    ]
]
```

### Override Global Discount

```php
// Group-specific pricing override
[
    'group_id' => 1,
    'use_product_pricing' => true,
    'discount_override' => 0,         // No group discount
    'pricing_model' => 'tiered'
]
```

## Group Communication

### Group-Based Email Templates

```php
// Send to specific group
[
    'template' => 'vip_announcement',
    'recipient_group' => 'Premium',
    'include_subgroups' => true,
    'from_name' => 'VIP Support Team'
]
```

### Email Prefix/Suffix

```php
// Group-specific email settings
[
    'group_name' => 'Enterprise',
    'email_prefix' => '[ENT]',
    'cc_tickets' => 'enterprise@example.com'
]
```

## Group-Based Automation

### Workflow Triggers

```php
// Group-specific automation
[
    'trigger' => 'invoice_created',
    'conditions' => [
        ['field' => 'client_group', 'value' => 'premium']
    ],
    'actions' => [
        ['action' => 'skip_reminder', 'days' => 0],
        ['action' => 'send_email', 'template' => 'premium_invoice']
    ]
]
```

### Dunning Exemptions

```php
// Premium group dunning
[
    'group_name' => 'Premium',
    'extend_grace_days' => 14,        // Extra 14 days
    'max_reminders' => 6,
    'skip_suspension' => true,
    'personal_dunning' => true
]
```

## Group Statistics

### Group Metrics

**Reports > Clients > Group Statistics**

```php
// Group statistics
[
    'groups' => [
        'Premium' => [
            'total_clients' => 500,
            'total_revenue' => 250000.00,
            'avg_lifetime_value' => 500.00,
            'churn_rate' => 2.5
        ],
        'Standard' => [
            'total_clients' => 2000,
            'total_revenue' => 300000.00,
            'avg_lifetime_value' => 150.00,
            'churn_rate' => 5.0
        ]
    ]
]
```

## Managing Group Membership

### Manual Assignment

**Admin: Clients > Select Client > Groups**

```php
// Change client group
[
    'userid' => 123,
    'from_group' => 'default',
    'to_group' => 'premium',
    'reason' => 'Customer upgraded',
    'changed_by' => 'admin_id',
    'changed_at' => '2024-05-15'
]
```

### Bulk Group Assignment

```php
// Move multiple clients
[
    'action' => 'bulk_group_change',
    'client_ids' => [123, 124, 125],
    'to_group' => 'vip',
    'keep_previous_groups' => false
]
```

### Multiple Group Assignment

```php
// Multiple groups per client
[
    'userid' => 123,
    'groups' => ['premium', 'enterprise', 'referral'],
    'primary_group' => 'premium'
]
```

## Group Deletion

### Deletion Rules

```php
// Cannot delete group with clients
[
    'move_clients_first' => true,
    'target_group' => 'default'
]

// Force delete (move to default)
[
    'action' => 'delete_group',
    'group_id' => 5,
    'move_clients_to' => 'default',
    'confirm_clients' => 10
]
```

## API Functions

```php
// Get client groups
$result = localAPI('GetClientGroups');

// Add client group
$result = localAPI('AddClientGroup', [
    'name' => 'Enterprise',
    'color' => '#9b59b6',
    'discount' => 20
]);

// Update client group
$result = localAPI('UpdateClientGroup', [
    'groupid' => 1,
    'discount' => 25
]);

// Change client group
$result = localAPI('UpdateClient', [
    'clientid' => 123,
    'groupid' => 2
]);
```

## Hooks

```php
// Hook: ClientGroupChanged
add_hook('ClientGroupChanged', 1, function($vars) {
    // $vars['userid']
    // $vars['old_group_id']
    // $vars['new_group_id']
    // Trigger pricing update, etc.
});
```

## Best Practices

1. **Logical naming**: Use clear, descriptive group names
2. **Color coding**: Use consistent colors for visual organization
3. **Limit groups**: Don't create too many overlapping groups
4. **Document rules**: Keep track of group assignment rules
5. **Review regularly**: Update groups based on business needs

## Related Documentation

- [Client Creation](./whmcs-client-creation.md)
- [Client Authentication](./whmcs-client-authentication.md)
- [Pricing Tiers](./whmcs-pricing-tiers.md)
- [Dunning Settings](./whmcs-dunning-settings.md)