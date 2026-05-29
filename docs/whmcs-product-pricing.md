# WHMCS Product Pricing

## Overview

Product pricing in WHMCS defines how much products cost across different billing cycles and currencies. Proper pricing configuration is essential for profitability.

## Pricing Structure

### Basic Pricing

```php
// Product pricing
[
    'product_id' => 1,
    'pricing' => [
        'monthly' => 10.00,
        'quarterly' => 27.00,
        'semiannually' => 50.00,
        'annually' => 95.00,
        'biennially' => 180.00,
        'triennially' => 250.00
    ]
]
```

### Billing Cycle Options

| Cycle | Description | Setup Fee |
|-------|-------------|-----------|
| Monthly | 30 days | 0.00 |
| Quarterly | 90 days | 0.00 |
| Semi-Annual | 180 days | 0.00 |
| Annual | 365 days | 0.00 |
| Biennial | 730 days | 0.00 |
| Triennial | 1095 days | 0.00 |

## Setup Fees

### One-Time Setup

```php
// Setup fee configuration
[
    'product_id' => 1,
    'setup_fee' => [
        'monthly' => 0.00,
        'quarterly' => 0.00,
        'annually' => 0.00
    ]
]
```

### Waived Setup

```php
// Conditional setup fee
[
    'setup_fee' => 25.00,
    'waive_if_annual' => true,
    'waive_for_groups' => ['premium', 'vip']
]
```

## Currency Pricing

### Multi-Currency Support

```php
// Pricing per currency
[
    'USD' => ['monthly' => 10.00, 'annually' => 95.00],
    'EUR' => ['monthly' => 9.00, 'annually' => 85.00],
    'GBP' => ['monthly' => 8.00, 'annually' => 75.00]
]
```

## Pricing Models

### Standard Pricing

```php
// Fixed price per cycle
[
    'model' => 'standard',
    'monthly' => 10.00,
    'annually' => 95.00
]
```

### Tiered Pricing

```php
// Volume-based pricing
[
    'model' => 'tiered',
    'tiers' => [
        ['qty' => '1-10', 'price' => 10.00],
        ['qty' => '11-50', 'price' => 8.50],
        ['qty' => '51+', 'price' => 7.00]
    ]
]
```

### Hybrid Pricing

```php
// Base + usage
[
    'base_price' => 10.00,
    'per_unit' => 0.05,
    'minimum' => 10.00
]
```

## Client Group Pricing

### Group-Specific Prices

```php
// Different price per group
[
    'product_id' => 1,
    'group_pricing' => [
        'default' => ['monthly' => 10.00],
        'premium' => ['monthly' => 8.50],
        'vip' => ['monthly' => 7.00]
    ]
]
```

### Automatic Discount

```php
// Group discount percentage
[
    'group_id' => 1,
    'discount' => 15,                    // 15% off
    'discount_type' => 'percentage'
]
```

## Pricing Display

### Show/Hide Pricing

```php
// Display settings
[
    'show_pricing' => true,
    'show_setup_fee' => true,
    'show_discount' => true,
    'show_annual_savings' => true
]
```

### Pricing Table

```smarty
<!-- Display pricing -->
<table>
    <tr>
        <th>Billing Cycle</th>
        <th>Price</th>
        <th>Savings</th>
    </tr>
    <tr>
        <td>Monthly</td>
        <td>$10.00</td>
        <td>-</td>
    </tr>
    <tr>
        <td>Annually</td>
        <td>$95.00</td>
        <td>Save $25 (21%)</td>
    </tr>
</table>
```

## Override Pricing

### Service-Level Override

```php
// Custom price for specific service
[
    'service_id' => 1,
    'override_price' => 9.00,
    'billing_cycle' => 'monthly',
    'override_reason' => 'Loyalty discount',
    'expires' => null
]
```

### Promotion Pricing

```php
// Temporary price
[
    'product_id' => 1,
    'promo_price' => 7.00,
    'start_date' => '2024-05-01',
    'end_date' => '2024-05-31'
]
```

## API Functions

```php
// Get product pricing
$result = localAPI('GetProductPricing', [
    'productid' => 1
]);

// Update pricing
$result = localAPI('UpdateProductPricing', [
    'productid' => 1,
    'pricing' => [
        'monthly' => 12.00
    ]
]);
```

## Best Practices

1. **Competitive pricing**: Research market rates
2. **Offer discounts**: Incentivize longer terms
3. **Group pricing**: Reward loyal customers
4. **Clear display**: Show savings prominently
5. **Regular review**: Adjust pricing as needed

## Related Documentation

- [Pricing Tiers](./whmcs-pricing-tiers.md)
- [Product Configuration](./whmcs-product-configuration.md)
- [Multi-Currency](./whmcs-multi-currency.md)
- [Client Groups](./whmcs-client-groups.md)