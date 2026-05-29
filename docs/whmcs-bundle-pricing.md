# WHMCS Bundle Pricing Documentation

## Overview

Bundle pricing allows you to offer multiple products together at a discounted rate, increasing average order value and customer perceived value.

## Configuration

### Enable Bundle Pricing

Navigate to: **Configuration > Products/Services > Bundle Pricing**

```php
// Bundle Configuration
$bundleConfig = [
    'enabled' => true,
    'allow_multiple_bundles' => false,
    'allow_bundles_with_other_offers' => false,

    // Display
    'show_savings' => true,
    'show_individual_prices' => true,
    'highlight_popular' => true,

    // Checkout
    'allow_item_removal' => false,
    'require_all_items' => true
];
```

## Bundle Types

### Standard Bundle

```php
// Standard product bundle
$standardBundle = [
    'name' => 'Starter Hosting Bundle',
    'description' => 'Everything you need to get started',
    'products' => [
        ['product_id' => 100, 'qty' => 1, 'optional' => false],
        ['addon_id' => 50, 'qty' => 1, 'optional' => false],
        ['addon_id' => 51, 'qty' => 1, 'optional' => false]
    ],
    'price' => 79.99,
    'individual_total' => 119.97,
    'savings' => 39.98,
    'savings_percentage' => 33
];
```

### Multi-Tier Bundle

```php
// Tiered bundle options
$multiTierBundle = [
    'name' => 'Hosting Packages',
    'tiers' => [
        'basic' => [
            'products' => [
                ['product_id' => 100, 'qty' => 1]
            ],
            'price' => 9.99,
            'savings' => 0
        ],
        'standard' => [
            'products' => [
                ['product_id' => 100, 'qty' => 1],
                ['addon_id' => 50, 'qty' => 1]
            ],
            'price' => 14.99,
            'savings' => 20
        ],
        'premium' => [
            'products' => [
                ['product_id' => 100, 'qty' => 1],
                ['addon_id' => 50, 'qty' => 1],
                ['addon_id' => 51, 'qty' => 1]
            ],
            'price' => 19.99,
            'savings' => 40
        ]
    ]
];
```

### Build Your Own Bundle

```php
// Customizable bundle
$customBundle = [
    'name' => 'Build Your Hosting Package',
    'base_product' => ['product_id' => 100, 'price' => 9.99],
    'addons' => [
        ['addon_id' => 50, 'name' => 'SSL', 'price' => 5.99],
        ['addon_id' => 51, 'name' => 'Backup', 'price' => 3.99],
        ['addon_id' => 52, 'name' => 'Email', 'price' => 2.99]
    ],
    'discount_type' => 'percentage',
    'discount_amount' => 10,           // 10% off when 3+ addons
    'discount_tiers' => [
        ['addon_count' => 1, 'discount' => 0],
        ['addon_count' => 2, 'discount' => 5],
        ['addon_count' => 3, 'discount' => 10]
    ]
];
```

## Bundle Pricing Models

### Fixed Bundle Price

```php
// Single bundle price
$fixedBundlePrice = [
    'price' => 79.99,
    'products' => [
        ['product_id' => 100, 'price' => 29.99],
        ['addon_id' => 50, 'price' => 29.99],
        ['addon_id' => 51, 'price' => 29.99]
    ],
    'savings' => 9.99,
    'savings_percentage' => 11
];
```

### Percentage Discount Bundle

```php
// Percentage off total
$percentageBundle = [
    'discount_type' => 'percentage',
    'discount_percentage' => 20,
    'min_products' => 2,
    'max_products' => 5,
    'applies_to' => 'total',
    'exclude_products' => []
];
```

### Volume Discount Bundle

```php
// Volume-based pricing
$volumeBundle = [
    'type' => 'volume_pricing',
    'tiers' => [
        ['qty' => 1, 'price' => 99.99],
        ['qty' => 2, 'price' => 189.99],      // 5% off
        ['qty' => 3, 'price' => 269.99],      // 10% off
        ['qty' => 5, 'price' => 429.99]        // 15% off
    ]
];
```

## Bundle Products

### Bundle Components

```php
// Bundle component structure
$bundleComponents = [
    'required' => [
        ['product_id' => 100, 'qty' => 1, 'price' => 0]  // Included
    ],
    'optional' => [
        ['addon_id' => 50, 'qty' => 1, 'price' => 9.99],
        ['addon_id' => 51, 'qty' => 1, 'price' => 4.99]
    ],
    'upgrades' => [
        ['from_product_id' => 100, 'to_product_id' => 101, 'upgrade_price' => 5.00]
    ]
];
```

### Product Upgrades in Bundle

```php
// Upgrade option within bundle
$bundleUpgrades = [
    'base_product_id' => 100,
    'upgrades' => [
        ['product_id' => 101, 'name' => 'Premium Hosting', 'upgrade_price' => 10.00],
        ['product_id' => 102, 'name' => 'Business Hosting', 'upgrade_price' => 20.00]
    ],
    'allow_downgrade' => true
];
```

## API Reference

### Create Bundle

```http
POST /billing/bundles
```

**Request Body:**

```json
{
  "name": "Starter Hosting Bundle",
  "description": "Everything you need to get started",
  "products": [
    {"product_id": 100, "qty": 1},
    {"addon_id": 50, "qty": 1},
    {"addon_id": 51, "qty": 1}
  ],
  "price": 79.99,
  "billing_cycle": "monthly",
  "enabled": true
}
```

### Get Bundle

```http
GET /billing/bundles/{bundle_id}
```

**Response:**

```json
{
  "id": "BUNDLE-001",
  "name": "Starter Hosting Bundle",
  "products": [
    {"product_id": 100, "name": "Basic Hosting", "price": 29.99},
    {"addon_id": 50, "name": "SSL Certificate", "price": 29.99},
    {"addon_id": 51, "name": "Daily Backup", "price": 29.99}
  ],
  "price": 79.99,
  "individual_total": 89.97,
  "savings": 9.98,
  "savings_percentage": 11
}
```

### Validate Bundle

```http
POST /billing/bundles/validate
```

**Request Body:**

```json
{
  "bundle_id": "BUNDLE-001",
  "user_id": 12345,
  "remove_items" => [],
  "add_items" => []
}
```

### Calculate Bundle Price

```http
POST /billing/bundles/calculate
```

**Request Body:**

```json
{
  "bundle_id": "BUNDLE-001",
  "customizations" => [
    {"addon_id": 50, "remove" => true},
    {"addon_id": 52, "add" => true}
  ]
}
```

**Response:**

```json
{
  "bundle_id": "BUNDLE-001",
  "original_price": 79.99,
  "modified_price": 74.99,
  "items" => [
    {"product_id": 100, "included": true},
    {"addon_id": 50, "removed": true},
    {"addon_id": 51, "included": true},
    {"addon_id": 52, "added": true, "price": 4.99}
  ],
  "adjustments" => [
    {"item": "SSL Certificate", "action": "removed", "savings": 0},
    {"item": "Email Package", "action": "added", "cost": 4.99}
  ]
}
```

## Bundle Display

### Customer Portal Display

**Client Area > Products > Bundles**

```
+------------------------------------------------------------------+
|  Hosting Bundles                                                 |
+------------------------------------------------------------------+
|                                                                  |
|  STARTER HOSTING BUNDLE                                [Popular] |
|  +--------------------------------------------------------------+|
|  | What's Included:                                            ||
|  | - Basic Hosting ($29.99 value)                             ||
|  | - SSL Certificate ($29.99 value)                          ||
|  | - Daily Backup ($29.99 value)                             ||
|  |                                                              ||
|  | Individual Total: $89.97                                  ||
|  |                                                              ||
|  | BUNDLE PRICE: $79.99/month                              ||
|  | You Save: $9.98 (11% off)                               ||
|  |                                                              ||
|  | [Add to Cart]                                             ||
|  +--------------------------------------------------------------+|
+------------------------------------------------------------------+
```

### Bundle Comparison Table

```
+------------------------------------------------------------------+
|  Compare Packages                                                 |
+------------------------------------------------------------------+
|                   | Basic   | Standard | Premium | Bundle    |
|-------------------|---------|----------|---------|------------|
| Hosting           | Yes     | Yes      | Yes     | Yes        |
| SSL Certificate   | No      | Yes      | Yes     | Yes        |
| Daily Backup      | No      | No       | Yes     | Yes        |
| Priority Support  | No      | No       | Yes     | Yes        |
|                   |         |          |         |            |
| Regular Price     | $29.99  | $39.99   | $59.99  | $49.99     |
| Bundle Price      | -       | -        | -       | $44.99     |
| Savings           | -        | -        | -       | 25% off    |
+------------------------------------------------------------------+
```

## Bundle Constraints

### Customer Constraints

```php
// Bundle eligibility
$bundleConstraints = [
    'new_customers_only' => false,
    'customer_groups' => [],
    'require_existing_products' => [
        ['product_id' => 100, 'qty' => 1]  // Must have basic hosting
    ],
    'exclude_customers' => []
];
```

### Order Constraints

```php
// Order restrictions
$orderConstraints = [
    'min_order_amount' => 0,
    'allow_with_other_bundles' => false,
    'allow_with_discounts' => true,
    'allow_with_coupons' => false,
    'require_separate_checkout' => false
];
```

## Bundle Management

### Add Bundle to Order

```php
// Add bundle to cart
$addBundle = [
    'bundle_id' => 'BUNDLE-001',
    'qty' => 1,
    'customizations' => [
        ['addon_id' => 50, 'selected' => true],
        ['addon_id' => 51, 'selected' => false]
    ]
];
```

### Modify Bundle

```php
// Modify existing bundle order
$modifyBundle = [
    'order_id' => 'ORD-12345',
    'add_items' => ['addon_id' => 52],
    'remove_items' => ['addon_id' => 50],
    'recalculate_price' => true
];
```

## Reporting

### Bundle Report

```http
GET /billing/reports/bundles
```

**Response:**

```json
{
  "period": "2024-01",
  "summary": {
    "total_bundle_orders" => 100,
    "total_bundle_revenue" => 4999.00,
    "total_savings_given" => 999.00,
    "avg_bundle_size" => 3,
    "avg_items_per_bundle" => 2.5
  },
  "by_bundle" => [
    {
      "bundle_id" => "BUNDLE-001",
      "name" => "Starter Hosting Bundle",
      "orders" => 60,
      "revenue" => 2999.00,
      "savings" => 600.00
    },
    {
      "bundle_id" => "BUNDLE-002",
      "name" => "Business Bundle",
      "orders" => 40,
      "revenue" => 2000.00,
      "savings" => 399.00
    }
  ]
}
```

## Troubleshooting

### Common Issues

| Issue | Cause | Solution |
|-------|-------|----------|
| Bundle not applying | Constraints not met | Check eligibility |
| Wrong price | Bundle misconfigured | Verify bundle setup |
| Can't modify | Bundle locked | Unlock bundle first |
| Conflict with promo | Incompatible offers | Check combination rules |

### Debug Commands

```bash
# Calculate bundle price
whmcscli bundle calculate --bundle_id=BUNDLE-001

# Validate bundle eligibility
whmcscli bundle validate --bundle_id=BUNDLE-001 --user_id=12345

# View bundle stats
whmcscli bundle stats --bundle_id=BUNDLE-001
```

## Best Practices

1. **Clear value proposition** - Show savings clearly
2. **Logical bundles** - Group related products
3. **Competitive pricing** - Make bundles attractive
4. **Flexible options** - Allow customization
5. **Track performance** - Monitor bundle sales
6. **Iterate based on data** - Adjust based on results

## See Also

- [Tiered Pricing](./whmcs-tiered-pricing.md)
- [Volume Discounts](./whmcs-volume-discounts.md)
- [Promotion Rules](./whmcs-promotion-rules.md)
