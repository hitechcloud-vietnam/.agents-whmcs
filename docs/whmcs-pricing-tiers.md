# WHMCS Pricing Tiers

## Overview

Pricing tiers in WHMCS allow you to configure different price points based on quantity, client groups, or billing cycles. This feature supports volume discounts, tiered pricing models, and promotional pricing strategies.

## Accessing Pricing Tiers

**Admin Area > Configuration > System Settings > Pricing**

Or via Product/Service configuration:
**Products > Edit Product > Pricing Tab > Pricing Tiers**

## Tiered Pricing Configuration

### Basic Structure

```php
// Example: Quantity-based pricing tiers
[
    ['from' => 1,  'to' => 10,  'price' => 10.00],
    ['from' => 11, 'to' => 50,  'price' => 8.50],
    ['from' => 51, 'to' => 100, 'price' => 7.00],
    ['from' => 101,'to' => null, 'price' => 5.00]
]
```

### Product Pricing Tiers Setup

**Products/Services > Edit Product > Pricing Tab**

1. Enable "Enable Pricing Tiers"
2. Define tier ranges and prices
3. Select tier application method

## Tier Application Methods

### Per Unit Pricing
Price varies per unit based on total quantity.

```php
// Example: 15 items ordered
// Tier: 11-50 = $8.50 per unit
// Total: 15 * $8.50 = $127.50
```

### Total Price Override
Flat total for quantity range.

```php
// Example: 15 items ordered
// Tier: 11-50 = $100 flat
// Total: $100 (regardless of quantity)
```

## Client Group Pricing

Create custom pricing for specific client groups:

### Configuration

**Configuration > Support > Client Groups**

```php
// Client Group: Premium
[
    'groupid' => 1,
    'name' => 'Premium',
    'discount' => 15  // 15% discount
]
```

### Group-Based Pricing Override

Create product-specific pricing for client groups:

```php
// Product: Standard Plan - Premium Clients
[
    'groupid' => 1,
    'monthly' => 85.00,    // $85 instead of $100
    'quarterly' => 240.00, // $240 instead of $285
    'annually' => 850.00   // $850 instead of $1000
]
```

## Billing Cycle Tiers

### Standard Cycles

| Cycle | Description |
|-------|-------------|
| Monthly | Billed every 30 days |
| Quarterly | Billed every 90 days |
| Semi-Annual | Billed every 180 days |
| Annual | Billed every 365 days |
| Biennial | Billed every 730 days |
| Triennial | Billed every 1095 days |

### Custom Cycles

```php
// Custom billing cycle
[
    'setupfee' => 25.00,
    'cycle' => 'custom_45_days',
    'price' => 45.00
]
```

### Cycle Discounts

```php
// Discount for longer terms
[
    'monthly' => 100.00,
    'quarterly' => 285.00,     // ~5% discount
    'annually' => 1000.00      // ~17% discount
]
```

## Tier-Based Automation

### Pricing Rule Engine

```php
// Automation Rules > Pricing
[
    'name' => 'Volume Discount',
    'conditions' => [
        ['field' => 'quantity', 'operator' => '>=', 'value' => 50]
    ],
    'actions' => [
        ['action' => 'discount_percent', 'value' => 10]
    ]
]
```

## Quantity-Based Pricing

### Setup in Product

```php
// Product Configuration > General
Allow Quantity: Yes
Minimum Quantity: 1
Maximum Quantity: 1000
Quantity Steps: 1
```

### Pricing Matrix

```php
// Qty 1-10: $10/unit
// Qty 11-25: $9/unit  
// Qty 26-50: $8/unit
// Qty 51+: $7/unit
```

### User Selection

Clients select quantity during order:

```
Quantity: [5] units @ $10.00 each = $50.00
```

## Upgrade/Downgrade Pricing

### Tier Changes

```php
// Moving from Tier 1 to Tier 2
// Prorated difference calculation
// Tier 1: $100/month
// Tier 2: $150/month
// Difference: $50/month
// Pro-rata for remaining days
```

### Custom Upgrade Pricing

```php
// Allow custom pricing for upgrades
// Override automatic calculation
// Option: 'Do Not Prorate'
// Option: 'Full Cycle Charge'
// Option: 'Custom Amount'
```

## Promotional Pricing

### Time-Limited Tiers

```php
// Promotion: First 3 months at discount
[
    'tier' => 'promo_first_3_months',
    'price' => 9.99,       // discounted price
    'after_months' => 3,
    'regular_price' => 19.99
]
```

### First Order Discount

```php
// Client first order discount
[
    'type' => 'first_order_discount',
    'discount_percent' => 20,
    'max_discount_amount' => 50.00
]
```

## API Integration

```php
// Calculate tiered price via API
$params = [
    'productid' => 1,
    'quantity' => 25,
    'billingcycle' => 'monthly',
    'clientgroupid' => 2
];

$result = localAPI('CalculateTieredPrice', $params);
```

### API Response

```json
{
    "result": "success",
    "priceperunit": 8.50,
    "setupfee": 0.00,
    "totalamount": 212.50,
    "discount": 37.50
}
```

## Display Considerations

### Showing Tier Pricing to Customers

**Configuration > Ordering > Display**

- Show all pricing tiers on product page
- Highlight best value tier
- Display savings percentage

### Client Area Display

```smarty
{foreach $product.pricing.tiers as $tier}
    {$tier.from} - {$tier.to} units: {$tier.price}
{/foreach}
```

## Best Practices

1. **Clear tier boundaries**: Make ranges obvious
2. **Logical pricing**: Ensure price drops make sense
3. **Consider setup fees**: Decide if tiers include setup
4. **Test calculations**: Verify prorated amounts
5. **Document changes**: Track pricing tier modifications

## Troubleshooting

### Tier Not Applying
- Verify tier range includes quantity
- Check client group permissions
- Confirm product has tier pricing enabled

### Incorrect Calculations
- Review billing cycle selection
- Check for conflicting discounts
- Verify tax configuration

## Related Documentation

- [Invoice Generation](./whmcs-invoice-generation.md)
- [Product Pricing](./whmcs-product-pricing.md)
- [Service Upgrade](./whmcs-service-upgrade.md)
- [Service Downgrade](./whmcs-service-downgrade.md)