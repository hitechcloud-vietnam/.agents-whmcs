# WHMCS Coupon Constraints Documentation

## Overview

Coupon constraints define rules and restrictions for coupon codes, controlling when, how, and by whom they can be used.

## Configuration

### Enable Coupon System

Navigate to: **Configuration > General Settings > Coupons**

```php
// Coupon Configuration
$couponConfig = [
    'enabled' => true,

    // General Settings
    'require_code' => true,
    'auto_generate_codes' => true,
    'code_format' => 'alphanumeric',  // alphanumeric, numeric, alphabetic

    // Validation
    'case_sensitive' => false,
    'strip_whitespace' => true,
    'max_attempts' => 5,              // Wrong attempts before lock

    // Limits
    'global_max_uses' => 10000,
    'require_unique_code' => true
];
```

## Coupon Types

### Single Use Coupon

```php
// One-time use coupon
$singleUseCoupon = [
    'code' => 'NEWYEAR2024',
    'type' => 'single_use',
    'uses_remaining' => 1,
    'max_uses' => 1,
    'value' => 15,                    // 15% off
    'applies_to' => 'order_total'
];
```

### Multi-Use Coupon

```php
// Multiple uses allowed
$multiUseCoupon = [
    'code' => 'SUMMERSALE',
    'type' => 'multi_use',
    'uses_remaining' => 500,
    'max_uses' => 500,
    'max_uses_per_client' => 1,
    'value' => 20,
    'applies_to' => 'order_total'
];
```

### Recurring Coupon

```php
// Applies to recurring orders
$recurringCoupon = [
    'code' => 'LOYALTY15',
    'type' => 'recurring',
    'applies_to' => 'recurring_orders',
    'recurring_duration' => 12,        // Apply for 12 billing cycles
    'value' => 15,                    // 15% off each cycle
    'first_order_discount' => 30      // 30% off first order
];
```

## Usage Constraints

### Per-User Limits

```php
// Per-client usage constraints
$perUserLimits = [
    'max_uses_per_client' => 1,
    'max_discount_per_client' => 50.00,
    'require_unique_client' => true,
    'client_blacklist' => []
];
```

### Per-Order Limits

```php
// Per-order constraints
$perOrderLimits = [
    'max_uses_per_order' => 1,
    'min_order_amount' => 25.00,
    'max_order_amount' => 1000.00,
    'min_quantity' => 1
];
```

### Global Limits

```php
// Global usage constraints
$globalLimits = [
    'max_total_uses' => 1000,
    'max_total_discount' => 10000.00,
    'max_discount_per_use' => 100.00,
    'stop_when_limit_reached' => true
];
```

## Product Constraints

### Applicable Products

```php
// Product restrictions
$productConstraints = [
    'applies_to' => 'specific_products',
    'product_ids' => [100, 101, 102],
    'exclude_products' => [],

    // Or by category
    'applies_to' => 'categories',
    'category_ids' => ['hosting', 'vps'],

    // Exclusions
    'exclude_new_customers_only' => false,
    'exclude_on_sale_items' => false
];
```

### Product-Specific Discounts

```php
// Different discount per product
$productSpecificDiscounts = [
    'enabled' => true,
    'discounts' => [
        ['product_id' => 100, 'discount' => 20],
        ['product_id' => 101, 'discount' => 15],
        ['product_id' => 102, 'discount' => 10]
    ],
    'fallback_discount' => 5
];
```

## Customer Constraints

### Customer Eligibility

```php
// Customer restrictions
$customerConstraints = [
    'new_customers_only' => true,
    'existing_customers_only' => false,
    'min_lifetime_value' => 0,
    'max_lifetime_value' => null,

    // By group
    'allowed_groups' => ['preferred', 'enterprise'],
    'excluded_groups' => [],

    // By customer
    'allowed_customers' => [],
    'excluded_customers' => []
];
```

### Registration Date

```php
// Registration date constraints
$registrationConstraints = [
    'registered_after' => '2024-01-01',
    'registered_before' => null,
    'days_since_registration' => 30  // Min days as customer
];
```

## Time Constraints

### Valid Date Range

```php
// Date range constraints
$dateConstraints = [
    'valid_from' => '2024-06-01 00:00:00',
    'valid_until' => '2024-06-30 23:59:59',
    'auto_expire' => true,
    'allow_extension' => true
];
```

### Time-of-Day Constraints

```php
// Time-of-day constraints
$timeConstraints = [
    'valid_hours' => [
        'enabled' => true,
        'start_time' => '09:00',
        'end_time' => '17:00',
        'timezone' => 'America/New_York'
    ],
    'valid_days' => ['Monday', 'Tuesday', 'Wednesday', 'Thursday', 'Friday']
];
```

### Usage Windows

```php
// Limited usage windows
$usageWindows = [
    'enabled' => true,
    'windows' => [
        ['start' => '2024-06-01', 'end' => '2024-06-07', 'max_uses' => 100],
        ['start' => '2024-06-08', 'end' => '2024-06-14', 'max_uses' => 200],
        ['start' => '2024-06-15', 'end' => '2024-06-30', 'max_uses' => null]
    ]
];
```

## Combination Constraints

### Coupon Stacking

```php
// Stacking rules
$stackingConstraints = [
    'allow_stacking' => false,
    'stacking_coupons' => [],

    // Or allow specific combinations
    'allow_stacking' => true,
    'stackable_with' => ['FREESHIPPING'],
    'non_stackable_with' => ['CLEARANCE25']
];
```

### Promotion Combination

```php
// Combine with promotions
$promotionCombination = [
    'combine_with_promotions' => true,
    'stackable_promotions' => ['percentage_off', 'free_shipping'],
    'non_stackable_promotions' => ['bundle_deal']
];
```

## Geographic Constraints

### Location-Based

```php
// Geographic restrictions
$geoConstraints = [
    'countries' => ['US', 'CA', 'GB'],
    'exclude_countries' => [],
    'require_billing_match' => true,
    'allow_shipping_match' => false
];
```

### Postal Code Restrictions

```php
// Postal code restrictions
$postalConstraints = [
    'enabled' => true,
    'valid_postal_codes' => ['10001', '10002', '10003'],
    'exclude_postal_codes' => [],
    'pattern_match' => 'NY1*'           // All NY postal codes starting with NY1
];
```

## API Reference

### Create Coupon

```http
POST /billing/coupons
```

**Request Body:**

```json
{
  "code": "SUMMER20",
  "name": "Summer Sale 20% Off",
  "type": "percentage",
  "value": 20,
  "applies_to": "order_total",
  "constraints": {
    "max_uses": 1000,
    "max_uses_per_client": 1,
    "min_order_amount": 50,
    "products": [100, 101],
    "valid_from": "2024-06-01",
    "valid_until": "2024-06-30"
  },
  "enabled": true
}
```

### Validate Coupon

```http
POST /billing/coupons/validate
```

**Request Body:**

```json
{
  "code": "SUMMER20",
  "user_id": 12345,
  "order_amount": 200.00,
  "products": [100],
  "billing_country": "US"
}
```

**Response:**

```json
{
  "valid": true,
  "discount_amount": 40.00,
  "constraints_met": [
    "within_date_range",
    "min_order_amount_met",
    "country_allowed",
    "uses_remaining"
  ],
  "constraints_not_met": []
}
```

### Get Coupon Usage

```http
GET /billing/coupons/{coupon_id}/usage
```

**Response:**

```json
{
  "coupon_id": "COUPON-001",
  "code": "SUMMER20",
  "total_uses": 500,
  "remaining_uses": 500,
  "total_discount": 10000.00,
  "usage_by_customer": [
    {"customer_id": 12345, "uses": 1, "discount": 40.00}
  ],
  "usage_by_day": [
    {"date": "2024-06-01", "uses": 50, "discount": 1000.00}
  ]
}
```

### Update Coupon Constraints

```http
PATCH /billing/coupons/{coupon_id}/constraints
```

**Request Body:**

```json
{
  "max_uses": 500,
  "valid_until": "2024-07-15",
  "constraints": {
    "min_order_amount": 75.00
  }
}
```

## Constraint Validation

### Validation Flow

```
1. Code submitted
2. Check: Code exists?
3. Check: Within valid dates?
4. Check: Uses remaining?
5. Check: User eligible?
6. Check: Order meets requirements?
7. Check: Products eligible?
8. Check: Within usage limits?
9. Check: Can stack with other codes?
10. Apply discount (if all checks pass)
```

### Error Messages

```php
// Constraint error messages
$errorMessages = [
    'code_not_found' => 'Coupon code not found',
    'expired' => 'This coupon has expired',
    'not_started' => 'This coupon is not yet valid',
    'uses_exhausted' => 'This coupon has reached its usage limit',
    'user_limit_reached' => 'You have already used this coupon',
    'min_order_not_met' => 'Minimum order amount of {amount} required',
    'product_not_eligible' => 'This coupon cannot be used with selected products',
    'customer_not_eligible' => 'This coupon is not available for your account',
    'cannot_stack' => 'This coupon cannot be combined with other offers',
    'country_restricted' => 'This coupon is not available in your region'
];
```

## Customer Portal

### Coupon Error Display

**Checkout > Apply Coupon**

```
+------------------------------------------+
|  Enter Coupon Code                       |
+------------------------------------------+
|  [SUMMER20_______________] [Apply]     |
|                                          |
|  Error: This coupon has expired.        |
|                                          |
|  [x] This coupon cannot be combined   |
|      with other offers.                  |
+------------------------------------------+
```

## Reporting

### Coupon Constraint Report

```http
GET /billing/reports/coupons
```

**Response:**

```json
{
  "period": "2024-01",
  "coupons": [
    {
      "id": "COUPON-001",
      "code": "SUMMER20",
      "constraints": {
        "max_uses": 1000,
        "uses": 800,
        "remaining": 200,
        "min_order_amount": 50,
        "products": ["hosting"]
      },
      "usage_stats": {
        "total_uses": 800,
        "successful_validations": 800,
        "failed_validations": 200,
        "failure_reasons": {
          "min_order_not_met" => 150,
          "product_not_eligible" => 50
        }
      }
    }
  ]
}
```

## Best Practices

1. **Clear constraints** - Make all rules visible
2. **Test thoroughly** - Validate all scenarios
3. **Monitor failures** - Track why coupons fail
4. **Set limits** - Prevent abuse
5. **Document exceptions** - For support staff
6. **Review effectiveness** - Adjust based on data

## Troubleshooting

### Common Issues

| Issue | Cause | Solution |
|-------|-------|----------|
| Code not working | Expired | Check valid dates |
| Constraint failed | Requirements not met | Show which constraint failed |
| Can't stack | Exclusive coupon | Remove other codes |
| Usage limit | Already used | Use different code |

### Debug Commands

```bash
# Validate coupon
whmcscli coupon validate --code=SUMMER20 --user_id=12345 --amount=200

# View coupon constraints
whmcscli coupon constraints --coupon_id=COUPON-001

# Check coupon usage
whmcscli coupon usage --coupon_id=COUPON-001
```

## See Also

- [Promotion Rules](./whmcs-promotion-rules.md)
- [Discount Limits](./whmcs-discount-limits.md)
- [Volume Discounts](./whmcs-volume-discounts.md)
