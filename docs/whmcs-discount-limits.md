# WHMCS Discount Limits Documentation

## Overview

Discount limits control the maximum discounts that can be applied to products, orders, and invoices, protecting profit margins while allowing flexible pricing.

## Configuration

### Enable Discount Limits

Navigate to: **Configuration > General Settings > Discount Settings**

```php
// Discount Configuration
$discountConfig = [
    'enabled' => true,

    // Global Limits
    'max_discount_percentage' => 50,     // Maximum any discount
    'max_discount_amount' => null,        // Or maximum in currency

    // Approval Requirements
    'require_approval_above' => 20,     // Require approval above 20%
    'require_approval_role' => 'billing_manager',

    // Logging
    'log_all_discounts' => true,
    'report_discount_usage' => true
];
```

## Discount Limit Types

### Global Discount Limit

```php
// Maximum discount across all products
$globalLimit = [
    'type' => 'percentage',
    'max_percentage' => 50,
    'max_amount' => null,           // No currency limit
    'applies_to' => 'all_products',
    'exceptions' => []
];
```

### Product Discount Limits

```php
// Discount limits per product
$productDiscountLimits = [
    'hosting' => [
        'max_discount_percentage' => 50,
        'max_discount_amount' => 100.00,
        'allow_free' => false
    ],
    'domain' => [
        'max_discount_percentage' => 30,
        'max_discount_amount' => 20.00,
        'allow_free' => false,
        'registry_rules_apply' => true
    ],
    'ssl' => [
        'max_discount_percentage' => 20,
        'max_discount_amount' => 100.00,
        'allow_free' => false
    ],
    'dedicated' => [
        'max_discount_percentage' => 30,
        'max_discount_amount' => 500.00,
        'allow_free' => false
    ],
    'addons' => [
        'max_discount_percentage' => 50,
        'max_discount_amount' => null,
        'allow_free' => true
    ]
];
```

### Category Discount Limits

```php
// Discount limits per product category
$categoryDiscountLimits = [
    'shared_hosting' => [
        'max_discount_percentage' => 50,
        'allow_free' => false
    ],
    'reseller_hosting' => [
        'max_discount_percentage' => 40,
        'allow_free' => false
    ],
    'vps' => [
        'max_discount_percentage' => 35,
        'allow_free' => false
    ],
    'dedicated_servers' => [
        'max_discount_percentage' => 25,
        'allow_free' => false
    ]
];
```

## Discount Limit Enforcement

### Enforcement Levels

```php
// How discounts are enforced
$enforcementLevel = [
    'soft' => [
        'warn_above_limit' => true,
        'allow_override' => true,
        'require_approval' => true
    ],
    'hard' => [
        'block_above_limit' => true,
        'allow_override' => false,
        'approval_required' => true
    ],
    'flexible' => [
        'enforce_at_checkout' => true,
        'allow_admin_override' => true,
        'require_approval_above' => 25,
        'max_override_role' => 'admin'
    ]
];
```

### Limit Check Flow

```
1. Discount applied
2. Check against product limit
3. Check against category limit
4. Check against global limit
5. If exceeded:
   - Soft limit: Warn but allow
   - Hard limit: Block/override to max
   - Flexible: Require approval
```

## API Reference

### Calculate Discount

```http
POST /billing/discount/calculate
```

**Request Body:**

```json
{
  "product_id": 100,
  "original_price": 99.99,
  "discount_percentage": 30,
  "discount_amount": null,
  "user_id": 12345
}
```

**Response:**

```json
{
  "original_price": 99.99,
  "discount_percentage": 30,
  "discount_amount": 29.99,
  "discounted_price": 69.99,
  "limits": {
    "product_max_percentage": 50,
    "product_max_amount": 100.00,
    "current_discount_within_limits": true
  }
}
```

### Validate Discount

```http
POST /billing/discount/validate
```

**Request Body:**

```json
{
  "product_id": 100,
  "discount_percentage": 60,
  "discount_amount": null
}
```

**Response:**

```json
{
  "valid": false,
  "errors": [
    "Discount 60% exceeds product limit of 50%"
  ],
  "max_allowed": {
    "percentage": 50,
    "amount": 49.99
  }
}
```

### Apply Discount with Override

```http
POST /billing/discount/apply
```

**Request Body:**

```json
{
  "product_id": 100,
  "discount_percentage": 60,
  "override_reason" => "Customer loyalty - approved by manager",
  "override_by" => "admin@example.com",
  "notify_customer" => true
}
```

## Discount Tier Limits

### Customer Tier Limits

```php
// Discount limits by customer tier
$customerTierLimits = [
    'new' => [
        'max_discount_percentage' => 10,
        'max_discount_amount' => 50.00,
        'allowed_promotions' => ['first_order']
    ],
    'standard' => [
        'max_discount_percentage' => 20,
        'max_discount_amount' => 100.00,
        'allowed_promotions' => ['seasonal', 'clearance']
    ],
    'preferred' => [
        'max_discount_percentage' => 30,
        'max_discount_amount' => 200.00,
        'allowed_promotions' => ['all']
    ],
    'enterprise' => [
        'max_discount_percentage' => 50,
        'max_discount_amount' => null,    // No limit
        'allowed_promotions' => ['all'],
        'custom_negotiation' => true
    ]
];
```

### Order Volume Limits

```php
// Discount limits based on order size
$volumeBasedLimits = [
    'first_order' => [
        'max_discount_percentage' => 15,
        'max_discount_amount' => 50.00
    ],
    'small_order' => [
        'min_amount' => 50,
        'max_discount_percentage' => 10,
        'max_discount_amount' => 25.00
    ],
    'medium_order' => [
        'min_amount' => 500,
        'max_discount_percentage' => 20,
        'max_discount_amount' => 200.00
    ],
    'large_order' => [
        'min_amount' => 5000,
        'max_discount_percentage' => 30,
        'max_discount_amount' => null
    ]
];
```

## Approval Workflow

### Approval Requirements

```php
// Approval requirements
$approvalConfig = [
    'enabled' => true,
    'threshold_percentage' => 20,
    'threshold_amount' => 100.00,

    // Either threshold triggers approval
    'threshold_type' => 'either',     // either, both

    // Approval workflow
    'approval_roles' => ['billing_manager', 'admin'],
    'approval_timeframe_hours' => 24,
    'auto_approve_below' => 10,
    'auto_approve_amount_below' => 50.00,

    // Notifications
    'notify_on_approval' => true,
    'notify_on_rejection' => true,
    'notify_customer_on_outcome' => true
];
```

### Approval Process

```php
// Approval workflow
$approvalWorkflow = [
    'steps' => [
        'submit' => [
            'action' => 'submit_for_approval',
            'notify_approvers' => true
        ],
        'review' => [
            'action' => 'manager_review',
            'required_role' => 'billing_manager'
        ],
        'decision' => [
            'approve' => 'apply_discount',
            'reject' => 'notify_customer'
        ]
    ]
];
```

## Customer Discount Limits

### View Allowed Discounts

**Client Area > Pricing**

```
+------------------------------------------------------------------+
|  Available Discounts                                             |
+------------------------------------------------------------------+
|                                                                  |
|  Your Tier: Preferred                                           |
|  Maximum Regular Discount: 30%                                  |
|  Available Promotions: Seasonal, Clearance                     |
|                                                                  |
|  Volume Discounts:                                               |
|  Orders $500+: Up to 20% off                                  |
|  Orders $5000+: Up to 30% off (Contact sales)                 |
+------------------------------------------------------------------+
```

## Reporting

### Discount Usage Report

```http
GET /billing/reports/discounts
```

**Response:**

```json
{
  "period": "2024-01",
  "summary": {
    "total_discounts_given" => 15000.00,
    "total_orders_with_discount" => 200,
    "average_discount_percentage" => 15,
    "average_discount_amount" => 75.00
  },
  "by_product" => [
    {"product" => "hosting", "discount_amount" => 8000.00},
    {"product" => "ssl", "discount_amount" => 3000.00}
  ],
  "over_limit_applied" => [
    {"count" => 5, "amount" => 500.00, "approval_status" => "approved"}
  ],
  "by_approval_status" => [
    {"status" => "auto_approved", "count" => 180},
    {"status" => "manager_approved", "count" => 15},
    {"status" => "rejected", "count" => 5}
  ]
}
```

## Exception Handling

### Override Limits

```php
// Admin override capabilities
$overrideConfig = [
    'allow_override' => true,
    'override_roles' => ['admin', 'billing_manager'],
    'max_override_percentage' => 100,
    'require_reason' => true,
    'require_approval_for_high' => true,
    'high_discount_threshold' => 75,
    'audit_log_override' => true
];
```

### Override Reasons

```php
// Acceptable override reasons
$overrideReasons = [
    'customer_loyalty' => [
        'enabled' => true,
        'max_discount_boost' => 10
    ],
    'competitive_match' => [
        'enabled' => true,
        'requires_competitor_name' => true,
        'max_discount_boost' => 15
    ],
    'account_issue' => [
        'enabled' => true,
        'requires_ticket_reference' => true
    ],
    'new_customer_acquisition' => [
        'enabled' => true,
        'max_discount_boost' => 25
    ]
];
```

## Troubleshooting

### Common Issues

| Issue | Cause | Solution |
|-------|-------|----------|
| Discount blocked | Exceeds limit | Lower discount or get approval |
| Wrong limit applied | Product miscategorized | Check category assignment |
| Approval pending | Requires manager | Contact approver |
| Override not working | Permission denied | Check admin role |

### Debug Commands

```bash
# Check discount limits for product
whmcscli discount limits --product_id=100

# Calculate discount
whmcscli discount calculate --product_id=100 --percentage=30

# View pending approvals
whmcscli discount pending-approvals

# Override discount
whmcscli discount override --product_id=100 --percentage=60 --reason="loyalty"
```

## Best Practices

1. **Set reasonable limits** - Balance flexibility with margin protection
2. **Use tiered limits** - Give better discounts to better customers
3. **Require approval** - For significant discounts
4. **Track discount usage** - Monitor margin impact
5. **Review regularly** - Adjust limits based on data
6. **Document exceptions** - Maintain clear audit trail

## See Also

- [Promotion Rules](./whmcs-promotion-rules.md)
- [Coupon Constraints](./whmcs-coupon-constraints.md)
- [Volume Discounts](./whmcs-volume-discounts.md)
