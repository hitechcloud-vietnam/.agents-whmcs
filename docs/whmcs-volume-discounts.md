# WHMCS Volume Discount System Documentation

## Overview

Volume discounts provide price breaks for larger quantities or higher order values, encouraging customers to buy more.

## Configuration

### Enable Volume Discounts

Navigate to: **Configuration > Products/Services > Volume Discounts**

```php
// Volume Discount Configuration
$volumeConfig = [
    'enabled' => true,

    // General Settings
    'discount_type' => 'percentage',    // percentage, fixed_amount
    'applies_to' => 'order_total',     // order_total, specific_products

    // Display
    'show_on_product_page' => true,
    'show_savings' => true,
    'highlight_best_value' => true
];
```

## Discount Tiers

### Standard Volume Tiers

```php
// Volume discount tiers
$volumeTiers = [
    'name' => 'Standard Volume Discount',
    'discount_type' => 'percentage',
    'tiers' => [
        ['min_qty' => 1, 'max_qty' => 9, 'discount' => 0],
        ['min_qty' => 10, 'max_qty' => 49, 'discount' => 5],
        ['min_qty' => 50, 'max_qty' => 99, 'discount' => 10],
        ['min_qty' => 100, 'max_qty' => null, 'discount' => 15]
    ]
];
```

### Order Value Tiers

```php
// Discount based on order total
$orderValueTiers = [
    'name' => 'Order Value Discount',
    'discount_type' => 'percentage',
    'based_on' => 'order_total',
    'tiers' => [
        ['min_amount' => 100, 'max_amount' => 499, 'discount' => 5],
        ['min_amount' => 500, 'max_amount' => 999, 'discount' => 10],
        ['min_amount' => 1000, 'max_amount' => 4999, 'discount' => 15],
        ['min_amount' => 5000, 'max_amount' => null, 'discount' => 20]
    ]
];
```

### Quantity Tiers

```php
// Discount based on quantity
$quantityTiers = [
    'name' => 'Quantity Discount',
    'discount_type' => 'percentage',
    'based_on' => 'quantity',
    'unit' => 'licenses',
    'tiers' => [
        ['min_qty' => 1, 'max_qty' => 5, 'discount' => 0],
        ['min_qty' => 6, 'max_qty' => 10, 'discount' => 5],
        ['min_qty' => 11, 'max_qty' => 25, 'discount' => 10],
        ['min_qty' => 26, 'max_qty' => 50, 'discount' => 15],
        ['min_qty' => 51, 'max_qty' => null, 'discount' => 20]
    ]
];
```

## Discount Types

### Percentage Discount

```php
// Percentage off
$percentageDiscount = [
    'type' => 'percentage',
    'tiers' => [
        ['min_qty' => 10, 'discount' => 5],
        ['min_qty' => 25, 'discount' => 10],
        ['min_qty' => 50, 'discount' => 15]
    ]
];
```

### Fixed Amount Discount

```php
// Fixed dollar amount off
$fixedDiscount = [
    'type' => 'fixed_amount',
    'tiers' => [
        ['min_qty' => 10, 'discount' => 10.00],
        ['min_qty' => 25, 'discount' => 30.00],
        ['min_qty' => 50, 'discount' => 75.00]
    ]
];
```

### Fixed Price Tiers

```php
// Flat price per unit at each tier
$flatPriceTiers = [
    'type' => 'flat_price',
    'tiers' => [
        ['min_qty' => 1, 'max_qty' => 10, 'price_per_unit' => 9.99],
        ['min_qty' => 11, 'max_qty' => 25, 'price_per_unit' => 8.99],
        ['min_qty' => 26, 'max_qty' => 50, 'price_per_unit' => 7.99],
        ['min_qty' => 51, 'max_qty' => null, 'price_per_unit' => 6.99]
    ]
];
```

## Product-Specific Discounts

### Configure Product Discounts

```php
// Per-product volume discounts
$productVolumeDiscounts = [
    'product_id' => 100,
    'name' => 'Software License',
    'discount_type' => 'percentage',
    'tiers' => [
        ['min_qty' => 1, 'discount' => 0],
        ['min_qty' => 5, 'discount' => 10],
        ['min_qty' => 10, 'discount' => 20],
        ['min_qty' => 25, 'discount' => 30]
    ]
];
```

### Category Discounts

```php
// Discounts applied to entire category
$categoryDiscounts = [
    'category_id' => 'hosting',
    'name' => 'Hosting Volume Discount',
    'discount_type' => 'percentage',
    'tiers' => [
        ['min_qty' => 2, 'discount' => 5],
        ['min_qty' => 5, 'discount' => 10],
        ['min_qty' => 10, 'discount' => 15]
    ]
];
```

## API Reference

### Get Volume Discount

```http
GET /products/{product_id}/volume-discounts
```

**Response:**

```json
{
  "product_id": 100,
  "discount_type": "percentage",
  "tiers": [
    {"min_qty": 1, "max_qty": 4, "discount": 0},
    {"min_qty": 5, "max_qty": 9, "discount": 5},
    {"min_qty": 10, "max_qty": 24, "discount": 10},
    {"min_qty": 25, "max_qty": null, "discount": 15}
  ],
  "display_price_tiers": [
    {"qty": "5-9", "discount": "5% off", "price_per_unit": "$9.49"},
    {"qty": "10-24", "discount": "10% off", "price_per_unit": "$8.99"},
    {"qty": "25+", "discount": "15% off", "price_per_unit": "$8.49"}
  ]
}
```

### Calculate Volume Discount

```http
POST /billing/volume-discount/calculate
```

**Request Body:**

```json
{
  "product_id": 100,
  "quantity": 15,
  "unit_price": 9.99,
  "user_id": 12345
}
```

**Response:**

```json
{
  "product_id": 100,
  "quantity": 15,
  "unit_price": 9.99,
  "tier" => {
    "min_qty" => 10,
    "max_qty" => 24,
    "discount_percentage" => 10
  },
  "discount_per_unit" => 1.00,
  "subtotal_before_discount" => 149.85,
  "discount_amount" => 14.99,
  "subtotal_after_discount" => 134.87,
  "savings_percentage" => 10
}
```

## Customer Tiers

### Customer-Based Discounts

```php
// Discount based on customer type
$customerVolumeDiscounts = [
    'standard' => [
        'min_qty' => 10,
        'discount' => 5
    ],
    'preferred' => [
        'min_qty' => 5,
        'discount' => 10
    ],
    'enterprise' => [
        'min_qty' => 2,
        'discount' => 15
    ]
];
```

### Lifetime Value Tiers

```php
// Discount based on customer lifetime value
$ltvDiscounts = [
    'ltv_bronze' => [
        'min_ltv' => 0,
        'discount' => 0,
        'volume_threshold' => 20
    ],
    'ltv_silver' => [
        'min_ltv' => 1000,
        'discount' => 5,
        'volume_threshold' => 10
    ],
    'ltv_gold' => [
        'min_ltv' => 5000,
        'discount' => 10,
        'volume_threshold' => 5
    ]
];
```

## Display Options

### Volume Pricing Table

```
+------------------------------------------------------------------+
|  Volume Pricing - Software License                                |
+------------------------------------------------------------------+
|                                                                  |
|  Quantity   | Unit Price    | Discount    | Total               |
|-------------|---------------|-------------|--------------------|
| 1-4        | $9.99         | -           | $9.99 - $39.96     |
| 5-9        | $9.49         | 5% off      | $47.45 - $85.41    |
| 10-24      | $8.99         | 10% off     | $89.90 - $215.76   |
| 25+        | $8.49         | 15% off     | $212.25+           |
|                                                                  |
|  Your Quantity: [15_______] [Calculate]                         |
|                                                                  |
|  Est. Unit Price: $8.99 (10% discount)                          |
|  Est. Total: $134.85                                            |
|  You Save: $14.99                                               |
|                                                                  |
|  [Add to Cart]                                                 |
+------------------------------------------------------------------+
```

### Quantity Calculator Widget

```javascript
// Volume discount calculator
const volumeCalculator = {
    basePrice: 9.99,
    tiers: [
        {min: 5, discount: 0.05},
        {min: 10, discount: 0.10},
        {min: 25, discount: 0.15}
    ],
    calculate: function(qty) {
        let discount = 0;
        for (let tier of this.tiers) {
            if (qty >= tier.min) discount = tier.discount;
        }
        let unitPrice = this.basePrice * (1 - discount);
        let total = unitPrice * qty;
        let savings = (this.basePrice * qty) - total;
        return {unitPrice, total, savings, discount: discount * 100};
    }
};
```

## Combination Rules

### Combine with Other Discounts

```php
// Combination rules
$combinationRules = [
    'allow_with_promotions' => true,
    'allow_with_coupons' => false,
    'allow_with_loyalty' => true,
    'apply_order' => [
        'step_1' => 'volume_discount',
        'step_2' => 'loyalty_points'
    ]
];
```

### Stacking Limitations

```php
// Stacking configuration
$stackingConfig = [
    'stack_with_bundles' => true,
    'stack_with_quantity_discounts' => false,
    'max_total_discount' => 50       // Maximum combined discount
];
```

## Customer Portal

### View Volume Discounts

**Client Area > Product > Pricing**

```
+------------------------------------------------------------------+
|  Volume Pricing Available                                         |
+------------------------------------------------------------------+
|                                                                  |
|  Buy more, save more!                                           |
|                                                                  |
|  Quantity   | Unit Price    | You Save    | Status              |
|-------------|---------------|-------------|--------------------|
| 1-4        | $9.99         | -           | -                  |
| 5-9        | $9.49         | $0.50 each  | -                  |
| 10-24      | $8.99         | $1.00 each  | Current Tier        |
| 25+        | $8.49         | $1.50 each  | Best Value          |
|                                                                  |
|  Current Selection: 15 licenses                                 |
|  Your Price: $8.99/license (10% off)                            |
|  Total: $134.85                                                |
|                                                                  |
|  Need just 10 more to reach 15% tier!                          |
|                                                                  |
|  [Add to Cart]                                                 |
+------------------------------------------------------------------+
```

## Reporting

### Volume Discount Report

```http
GET /billing/reports/volume-discounts
```

**Response:**

```json
{
  "period": "2024-01",
  "summary": {
    "total_orders_with_volume" => 200,
    "total_volume_discount" => 5000.00,
    "avg_discount_percentage" => 8,
    "units_sold_at_discount" => 5000
  },
  "by_tier" => [
    {"tier" => "5-9 units", "orders" => 100, "discount" => 1000.00},
    {"tier" => "10-24 units", "orders" => 70, "discount" => 2500.00},
    {"tier" => "25+ units", "orders" => 30, "discount" => 1500.00}
  ],
  "by_product" => [
    {"product" => "Software License", "discount" => 3000.00},
    {"product" => "Add-on Pack", "discount" => 2000.00}
  ]
}
```

## Troubleshooting

### Common Issues

| Issue | Cause | Solution |
|-------|-------|----------|
| Discount not applying | Qty below threshold | Verify quantity |
| Wrong discount | Tier misconfigured | Check tier settings |
| Can't stack | Combination rules | Review stacking config |
| Display wrong | Template issue | Check display settings |

### Debug Commands

```bash
# Calculate volume discount
whmcscli volume calculate --product_id=100 --qty=15

# View discount tiers
whmcscli volume tiers --product_id=100

# Test combination
whmcscli volume test-combination --product_id=100 --qty=25 --coupon=SUMMER10
```

## Best Practices

1. **Clear thresholds** - Make tier boundaries obvious
2. **Meaningful gaps** - Discounts should reward buying more
3. **Highlight savings** - Show how much customers save
4. **Psychological pricing** - Set sweet spots strategically
5. **Monitor usage** - Track which tiers customers hit
6. **Adjust regularly** - Optimize based on behavior

## See Also

- [Tiered Pricing](./whmcs-tiered-pricing.md)
- [Bundle Pricing](./whmcs-bundle-pricing.md)
- [Promotion Rules](./whmcs-promotion-rules.md)
