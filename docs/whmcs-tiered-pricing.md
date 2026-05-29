# WHMCS Tiered Pricing Documentation

## Overview

Tiered pricing allows different price points based on quantity tiers, product options, or customer segments, enabling sophisticated pricing strategies.

## Configuration

### Enable Tiered Pricing

Navigate to: **Configuration > Products/Services > Tiered Pricing**

```php
// Tiered Pricing Configuration
$tieredConfig = [
    'enabled' => true,
    'default_tier_type' => 'quantity',

    // Display
    'show_all_tiers' => true,
    'highlight_recommended' => true,
    'show_savings' => true,

    // Calculation
    'price_per_tier' => true,           // Each tier at its own price
    'graduated_pricing' => false,       // Each unit at tier price
    'mixed_pricing' => false
];
```

## Tier Types

### Quantity-Based Tiers

```php
// Quantity pricing tiers
$quantityTiers = [
    'name' => 'API Call Pricing',
    'unit' => '1000 API calls',
    'tiers' => [
        ['min' => 1, 'max' => 10, 'price' => 10.00],
        ['min' => 11, 'max' => 50, 'price' => 8.00],
        ['min' => 51, 'max' => 100, 'price' => 6.00],
        ['min' => 101, 'max' => null, 'price' => 4.00]
    ]
];
```

### Price-Per Tier Calculation

```php
// Price per tier (each tier has flat rate)
$pricePerTier = [
    'type' => 'price_per_tier',
    'example' => [
        'qty' => 25,
        'tier_price' => 8.00,         // Flat rate for tier
        'total' => 25 * 8.00 = 200.00
    ]
];
```

### Graduated Pricing

```php
// Graduated pricing (each unit priced at its tier)
$graduatedPricing = [
    'type' => 'graduated',
    'example' => [
        'qty' => 25,
        'calculation' => [
            // First 10 at $10
            ['qty' => 10, 'price' => 10.00, 'subtotal' => 100.00],
            // Next 15 at $8
            ['qty' => 15, 'price' => 8.00, 'subtotal' => 120.00]
        ],
        'total' => 220.00
    ]
];
```

## Product Tiers

### Hosting Tiered Pricing

```php
// Hosting product tiers
$hostingTiers = [
    'product_id' => 100,
    'name' => 'Cloud Hosting',
    'billing_cycle' => 'monthly',
    'tiers' => [
        'tier_1' => [
            'name' => 'Starter',
            'storage' => '10GB',
            'bandwidth' => '100GB',
            'price' => 9.99
        ],
        'tier_2' => [
            'name' => 'Professional',
            'storage' => '50GB',
            'bandwidth' => '500GB',
            'price' => 24.99
        ],
        'tier_3' => [
            'name' => 'Business',
            'storage' => '100GB',
            'bandwidth' => '1TB',
            'price' => 49.99
        ],
        'tier_4' => [
            'name' => 'Enterprise',
            'storage' => '500GB',
            'bandwidth' => 'Unlimited',
            'price' => 99.99
        ]
    ]
];
```

### Addon Tiers

```php
// Addon tiers
$addonTiers = [
    'addon_id' => 50,
    'name' => 'Additional Storage',
    'tiers' => [
        ['qty' => '10GB', 'price' => 5.00],
        ['qty' => '50GB', 'price' => 20.00],
        ['qty' => '100GB', 'price' => 35.00]
    ]
];
```

## Customer Tiers

### Customer Segment Pricing

```php
// Pricing by customer tier
$customerTierPricing = [
    'enabled' => true,
    'tiers' => [
        'standard' => [
            'discount_percentage' => 0,
            'access_to_tiers' => [1, 2]
        ],
        'preferred' => [
            'discount_percentage' => 10,
            'access_to_tiers' => [1, 2, 3]
        ],
        'enterprise' => [
            'discount_percentage' => 20,
            'access_to_tiers' => [1, 2, 3, 4],
            'custom_pricing' => true
        ]
    ]
];
```

### Loyalty Tiers

```php
// Loyalty-based pricing
$loyaltyTiers = [
    'bronze' => [
        'min_lifetime_value' => 0,
        'discount' => 0,
        'tier_access' => [1, 2]
    ],
    'silver' => [
        'min_lifetime_value' => 500,
        'discount' => 5,
        'tier_access' => [1, 2, 3]
    ],
    'gold' => [
        'min_lifetime_value' => 2000,
        'discount' => 10,
        'tier_access' => [1, 2, 3, 4]
    ],
    'platinum' => [
        'min_lifetime_value' => 10000,
        'discount' => 20,
        'tier_access' => 'all',
        'custom_pricing' => true
    ]
];
```

## API Reference

### Get Tiered Pricing

```http
GET /products/{product_id}/pricing/tiers
```

**Response:**

```json
{
  "product_id": 100,
  "name": "Cloud Hosting",
  "tiers": [
    {
      "tier_id" => "tier_1",
      "name" => "Starter",
      "features" => {
        "storage" => "10GB",
        "bandwidth" => "100GB"
      },
      "price" => 9.99,
      "monthly_equivalent" => 9.99
    },
    {
      "tier_id" => "tier_2",
      "name" => "Professional",
      "features" => {...},
      "price" => 24.99,
      "monthly_equivalent" => 24.99,
      "recommended" => true
    }
  ]
}
```

### Calculate Tiered Price

```http
POST /billing/calculate
```

**Request Body:**

```json
{
  "product_id": 100,
  "tier_id" => "tier_2",
  "user_id": 12345,
  "billing_cycle" => "monthly"
}
```

**Response:**

```json
{
  "product_id": 100,
  "tier_id" => "tier_2",
  "base_price" => 24.99,
  "customer_discount" => 10,
  "final_price" => 22.49,
  "savings" => 2.50,
  "price_breakdown" => {
    "tier_price" => 24.99,
    "loyalty_discount" => 2.50
  }
}
```

### Update Tier Pricing

```http
PATCH /products/{product_id}/pricing/tiers/{tier_id}
```

**Request Body:**

```json
{
  "price" => 29.99,
  "features" => {
    "storage" => "60GB"
  }
}
```

## Quantity Pricing

### Configure Quantity Tiers

```php
// Quantity-based pricing
$quantityPricing = [
    'product_id' => 200,
    'unit' => 'users',
    'tiers' => [
        ['min' => 1, 'max' => 10, 'price_per_unit' => 10.00],
        ['min' => 11, 'max' => 50, 'price_per_unit' => 8.00],
        ['min' => 51, 'max' => 100, 'price_per_unit' => 6.00],
        ['min' => 101, 'max' => null, 'price_per_unit' => 4.00]
    ]
];
```

### Calculate Quantity Price

```php
// Calculate price for quantity
function calculateQuantityPrice($qty, $tiers) {
    $total = 0;
    $remaining = $qty;

    foreach ($tiers as $tier) {
        $max = $tier['max'] ?? $qty;
        $min = $tier['min'];
        $tierSize = $max - $min + 1;

        if ($remaining <= 0) break;

        $unitsInTier = min($remaining, $tierSize);
        $total += $unitsInTier * $tier['price_per_unit'];
        $remaining -= $unitsInTier;
    }

    return $total;
}

// Example: 25 users
// First 10 @ $10 = $100
// Next 15 @ $8 = $120
// Total = $220
```

## Display Options

### Tier Comparison Table

```
+------------------------------------------------------------------+
|  Cloud Hosting Plans                                             |
+------------------------------------------------------------------+
|                   | Starter   | Professional | Business | Enterprise |
|-------------------|-----------|--------------|----------|------------|
| Storage           | 10GB      | 50GB         | 100GB    | 500GB      |
| Bandwidth         | 100GB     | 500GB        | 1TB      | Unlimited  |
| Email Accounts    | 5         | 25           | 100      | Unlimited  |
| SSL Certificate   | Basic     | Standard     | Premium  | Premium    |
| Support           | Email      | Email+Chat   | Priority | Dedicated  |
|                   |           |              |          |            |
| Monthly Price     | $9.99     | $24.99 [+]   | $49.99   | $99.99     |
| Annual Price     | $99.99    | $249.99 [+]  | $499.99  | $999.99    |
| Annual Savings    | -         | $49.99        | $99.99   | $199.99    |
+------------------------------------------------------------------+
```

### Pricing Slider

```javascript
// Interactive pricing slider
const pricingSlider = {
    min: 1,
    max: 100,
    step: 1,
    tiers: [
        {min: 1, max: 10, price: 10},
        {min: 11, max: 50, price: 8},
        {min: 51, max: 100, price: 6}
    ],
    calculate: function(qty) {
        let total = 0;
        let remaining = qty;
        for (let tier of this.tiers) {
            if (remaining <= 0) break;
            let units = Math.min(remaining, tier.max - tier.min + 1);
            total += units * tier.price;
            remaining -= units;
        }
        return total;
    }
};
```

## Customization

### Allow Tier Upgrades

```php
// Allow upgrading between tiers
$tierUpgradeConfig = [
    'allow_upgrade' => true,
    'prorate_on_upgrade' => true,
    'allow_downgrade' => true,
    'downgrade_timing' => 'end_of_cycle',
    'upgrade_timing' => 'immediate'
];
```

### Tier Features

```php
// Configure tier features
$tierFeatures = [
    'storage' => [
        'tier_1' => '10GB',
        'tier_2' => '50GB',
        'tier_3' => '100GB',
        'tier_4' => '500GB'
    ],
    'bandwidth' => [
        'tier_1' => '100GB',
        'tier_2' => '500GB',
        'tier_3' => '1TB',
        'tier_4' => 'Unlimited'
    ],
    'support' => [
        'tier_1' => 'Email',
        'tier_2' => 'Email + Chat',
        'tier_3' => 'Priority',
        'tier_4' => 'Dedicated'
    ]
];
```

## Customer Portal

### View Tier Options

**Client Area > My Services > Upgrade Plan**

```
+------------------------------------------------------------------+
|  Upgrade Your Hosting Plan                                        |
+------------------------------------------------------------------+
|                                                                  |
|  Current Plan: Professional ($24.99/month)                       |
|                                                                  |
|  Available Upgrades:                                            |
|  +--------------------------------------------------------------+|
|  | [ ] Business ($49.99/mo)                                     ||
|  |     +$25.00/mo upgrade                                      ||
|  |     Includes: Double storage, priority support              ||
|  |                                                              ||
|  | [ ] Enterprise ($99.99/mo)                                   ||
|  |     +$75.00/mo upgrade                                      ||
|  |     Includes: Unlimited storage, dedicated support           ||
|  +--------------------------------------------------------------+|
|                                                                  |
|  [Confirm Upgrade]                                               |
+------------------------------------------------------------------+
```

## Reporting

### Tier Pricing Report

```http
GET /billing/reports/tiered-pricing
```

**Response:**

```json
{
  "period": "2024-01",
  "summary": {
    "total_tiered_products" => 50,
    "upgrades" => 30,
    "downgrades" => 5,
    "total_upgrade_revenue" => 750.00
  },
  "by_tier" => [
    {"tier" => "tier_1", "customers" => 100},
    {"tier" => "tier_2", "customers" => 80},
    {"tier" => "tier_3", "customers" => 40},
    {"tier" => "tier_4", "customers" => 20}
  ],
  "tier_transitions" => [
    {"from" => "tier_1", "to" => "tier_2", "count" => 20},
    {"from" => "tier_2", "to" => "tier_3", "count" => 10}
  ]
}
```

## Troubleshooting

### Common Issues

| Issue | Cause | Solution |
|-------|-------|----------|
| Wrong price | Tier misconfigured | Verify tier settings |
| Can't upgrade | Prorate error | Check calculation |
| Missing features | Features not set | Configure tier features |
| Customer sees wrong tiers | Tier access restricted | Check access rules |

### Debug Commands

```bash
# Calculate tiered price
whmcscli tiered calculate --product_id=100 --tier=tier_2 --user_id=12345

# View tier configuration
whmcscli tiered config --product_id=100

# Test upgrade
whmcscli tiered upgrade --user_id=12345 --from=tier_1 --to=tier_2
```

## Best Practices

1. **Clear differentiation** - Make tier differences obvious
2. **Logical pricing** - Price gaps should be justified
3. **Sweet spot tier** - Highlight most popular option
4. **Easy upgrades** - Minimize friction for upgrades
5. **Monitor transitions** - Track tier movement
6. **Iterate based on data** - Adjust based on customer behavior

## See Also

- [Volume Discounts](./whmcs-volume-discounts.md)
- [Bundle Pricing](./whmcs-bundle-pricing.md)
- [Customer Tiers](../customers/customer-tiers.md)
