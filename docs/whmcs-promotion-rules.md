# WHMCS Promotion Configuration Documentation

## Overview

Promotions in WHMCS allow you to create time-limited offers, special deals, and marketing campaigns with flexible conditions and rules.

## Configuration

### Enable Promotions

Navigate to: **Configuration > General Settings > Promotions**

```php
// Promotion Configuration
$promotionConfig = [
    'enabled' => true,

    // General Settings
    'allow_multiple' => false,           // Allow multiple promos per order
    'allow_with_coupons' => true,
    'allow_with_other_promotions' => false,

    // Limits
    'max_promotions_per_order' => 1,
    'global_max_uses' => null,
    'require_coupon_code' => true,

    // Stacking Rules
    'stackable_promotions' => [
        'percentage_off',
        'fixed_amount_off'
    ],
    'non_stackable' => [
        'free_domain',
        'free_setup'
    ]
];
```

## Promotion Types

### Percentage Discount

```php
// Percentage discount promotion
$percentagePromo = [
    'type' => 'percentage',
    'value' => 20,                     // 20% off
    'applies_to' => 'order_total',
    'max_discount' => 100.00,          // Cap at $100

    // Or apply to specific items
    'applies_to' => 'specific_products',
    'product_ids' => [100, 101, 102],

    'conditions' => [
        'min_order_amount' => 50.00,
        'first_order_only' => false,
        'for_new_customers_only' => false
    ]
];
```

### Fixed Amount Discount

```php
// Fixed amount discount
$fixedPromo = [
    'type' => 'fixed_amount',
    'value' => 25.00,                  // $25 off
    'applies_to' => 'order_total',

    'conditions' => [
        'min_order_amount' => 100.00
    ]
];
```

### Free Product/Service

```php
// Free product promotion
$freeProductPromo = [
    'type' => 'free_product',
    'free_items' => [
        ['product_id' => 100, 'qty' => 1],
        ['addon_id' => 50, 'qty' => 1]
    ],
    'requires_purchase' => true,
    'required_products' => ['hosting']
];
```

### Free Domain

```php
// Free domain with hosting
$freeDomainPromo = [
    'type' => 'free_domain',
    'domain_extensions' => ['.com', '.net', '.org'],
    'requires_product' => 'hosting',
    'max_domain_value' => 15.00
];
```

### Free Setup

```php
// Free setup promotion
$freeSetupPromo = [
    'type' => 'free_setup',
    'applies_to' => 'products',
    'product_ids' => [100, 101, 102],
    'max_setup_value' => 99.00
];
```

## Promotion Conditions

### Order Conditions

```php
// Order-based conditions
$orderConditions = [
    'min_order_amount' => 100.00,
    'max_order_amount' => 1000.00,
    'min_quantities' => [
        ['product_id' => 100, 'min_qty' => 2]
    ],
    'require_products' => [
        ['product_id' => 100, 'qty' => 1]
    ]
];
```

### Customer Conditions

```php
// Customer-based conditions
$customerConditions = [
    'new_customers_only' => true,
    'existing_customers_only' => false,
    'min_total_lifetime_value' => 0,
    'customer_groups' => ['preferred', 'enterprise'],
    'customer_ids' => [],
    'exclude_customer_ids' => []
];
```

### Product Conditions

```php
// Product-based conditions
$productConditions = [
    'require_all_products' => false,
    'require_any_products' => true,
    'products' => [100, 101, 102],
    'exclude_products' => [],
    'require_categories' => ['hosting'],
    'exclude_categories' => ['custom_development']
];
```

### Time Conditions

```php
// Time-based conditions
$timeConditions = [
    'start_date' => '2024-01-01',
    'end_date' => '2024-01-31',
    'days_of_week' => ['Monday', 'Tuesday', 'Wednesday', 'Thursday', 'Friday'],
    'time_of_day_start' => '00:00',
    'time_of_day_end' => '23:59'
];
```

## API Reference

### Create Promotion

```http
POST /billing/promotions
```

**Request Body:**

```json
{
  "code": "SUMMER20",
  "name" => "Summer Sale 20% Off",
  "type" => "percentage",
  "value" => 20,
  "applies_to" => "order_total",
  "max_uses" => 1000,
  "max_uses_per_client" => 1,
  "min_order_amount" => 50.00,
  "start_date" => "2024-06-01",
  "end_date" => "2024-08-31",
  "conditions" => {
    "new_customers_only" => false,
    "products" => []
  },
  "enabled" => true
}
```

### Update Promotion

```http
PATCH /billing/promotions/{promo_id}
```

### Get Promotion

```http
GET /billing/promotions/{promo_id}
```

**Response:**

```json
{
  "id": "PROMO-001",
  "code": "SUMMER20",
  "name": "Summer Sale 20% Off",
  "type": "percentage",
  "value": 20,
  "applies_to": "order_total",
  "uses" => 150,
  "max_uses" => 1000,
  "remaining_uses" => 850,
  "min_order_amount" => 50.00,
  "start_date": "2024-06-01",
  "end_date": "2024-08-31",
  "status" => "active",
  "conditions" => {...}
}
```

### List Promotions

```http
GET /billing/promotions
```

**Query Parameters:**
- `status`: active, expired, scheduled, disabled
- `type`: percentage, fixed_amount, free_product, free_domain

### Validate Promotion

```http
POST /billing/promotions/validate
```

**Request Body:**

```json
{
  "code": "SUMMER20",
  "user_id": 12345,
  "order_amount": 200.00,
  "products" => [100, 101]
}
```

**Response:**

```json
{
  "valid": true,
  "discount_amount": 40.00,
  "final_amount": 160.00,
  "conditions_met" => true,
  "conditions_not_met" => []
}
```

### Apply Promotion

```http
POST /billing/promotions/apply
```

**Request Body:**

```json
{
  "code": "SUMMER20",
  "order_id" => "ORD-12345"
}
```

## Restriction Rules

### Geographic Restrictions

```php
// Location-based restrictions
$geoRestrictions = [
    'countries' => ['US', 'CA'],
    'exclude_countries' => [],
    'regions' => ['US-CA', 'US-NY'],
    'require_billing_address' => true
];
```

### Product Restrictions

```php
// Product restrictions
$productRestrictions = [
    'applies_to' => 'specific_products',
    'products' => [100, 101, 102],
    'categories' => ['hosting'],
    'exclude_products' => [200],
    'require_products' => false
];
```

### Customer Restrictions

```php
// Customer restrictions
$customerRestrictions = [
    'new_customers_only' => true,
    'existing_customers_only' => false,
    'customer_ids' => [],
    'customer_groups' => [],
    'exclude_customer_ids' => [],
    'min_lifetime_value' => 0
];
```

## Combination Rules

### Stacking Configuration

```php
// Allow promotion stacking
$stackingConfig = [
    'allow_stacking' => true,
    'max_stackable' => 2,

    // Which can stack
    'stackable_with' => [
        'percentage' => ['free_product'],
        'fixed_amount' => ['free_setup'],
        'free_domain' => ['percentage', 'fixed_amount']
    ],

    // How to combine
    'combination_method' => 'sum'      // sum, first_only, highest_only
];
```

### Exclusive Promotions

```php
// Exclusive promotion
$exclusivePromo = [
    'exclusive' => true,
    'cannot_combine_with' => ['all'],
    'show_as_banner' => true,
    'banner_priority' => 1
];
```

## Usage Limits

### Per-Promotion Limits

```php
// Usage limits per promotion
$promoLimits = [
    'max_uses' => 1000,
    'max_uses_per_client' => 1,
    'max_uses_per_order' => 1,
    'max_discount_per_use' => 100.00,

    // Time-based limits
    'uses_per_day' => 100,
    'uses_per_week' => 500
];
```

### Global Limits

```php
// Global promotion limits
$globalLimits = [
    'max_discount_percentage' => 50,
    'max_discount_amount' => 1000.00,
    'allow_free_products' => true,
    'allow_free_domains' => true
];
```

## Notifications

### Promotion Notifications

```php
// Notification settings
$promoNotifications = [
    'notify_on_apply' => false,
    'notify_on_expire' => true,
    'notify_near_expiry_days' => 3,

    // Customer notifications
    'email_template' => 'promotion_applied',
    'include_discount_amount' => true,

    // Admin notifications
    'notify_admin' => true,
    'admin_threshold_uses' => 100  // Notify after 100 uses
];
```

## Customer Portal

### View Available Promotions

**Client Area > Promotions**

```
+------------------------------------------------------------------+
|  Available Promotions                                            |
+------------------------------------------------------------------+
|                                                                  |
|  +--------------------------------------------------------------+|
|  | [ACTIVE] Summer Sale 20% Off                                 ||
|  | Use code: SUMMER20                                           ||
|  | Valid: June 1 - August 31, 2024                            ||
|  | Minimum order: $50.00                                        ||
|  | [Apply Code]                                                ||
|  +--------------------------------------------------------------+|
|                                                                  |
|  +--------------------------------------------------------------+|
|  | [ACTIVE] Free Domain with Hosting                           ||
|  | Code: FREEDOMAIN                                            ||
|  | Get a free .com/.net/.org domain with any hosting plan.     ||
|  | [Apply Code]                                                ||
|  +--------------------------------------------------------------+|
+------------------------------------------------------------------+
```

### Apply Promotion Code

**Checkout > Add Promo Code**

```
+------------------------------------------+
|  Have a promo code?                       |
+------------------------------------------+
|  [SUMMER20___________________] [Apply]   |
|                                          |
|  Summer Sale 20% Off                     |
|  Applied successfully!                   |
|  You save: $40.00                       |
|  New Total: $160.00                      |
+------------------------------------------+
```

## Reporting

### Promotion Report

```http
GET /billing/reports/promotions
```

**Response:**

```json
{
  "period": "2024-01",
  "promotions": [
    {
      "id": "PROMO-001",
      "code": "SUMMER20",
      "name": "Summer Sale 20% Off",
      "type": "percentage",
      "uses": 500,
      "total_discount" => 10000.00,
      "avg_discount_per_use" => 20.00,
      "revenue_impact" => 50000.00
    }
  ],
  "summary": {
    "total_promotions_used" => 1000,
    "total_discount_given" => 25000.00,
    "total_revenue" => 125000.00,
    "effective_discount_rate" => 20
  }
}
```

## Troubleshooting

### Common Issues

| Issue | Cause | Solution |
|-------|-------|----------|
| Code not working | Expired or not started | Check dates |
| Conditions not met | Order doesn't qualify | Verify conditions |
| Already used | Per-client limit reached | Use different code |
| Can't stack | Exclusive promotion | Remove other promos |

### Debug Commands

```bash
# Test promotion code
whmcscli promo validate --code=SUMMER20 --amount=200 --user_id=12345

# View promotion stats
whmcscli promo stats --promo_id=PROMO-001

# Disable promotion
whmcscli promo disable --promo_id=PROMO-001
```

## Best Practices

1. **Clear conditions** - Make requirements obvious
2. **Set limits** - Prevent abuse
3. **Test thoroughly** - Verify calculations
4. **Monitor usage** - Track effectiveness
5. **Communicate clearly** - Explain savings
6. **Review regularly** - Adjust based on results

## See Also

- [Coupon Constraints](./whmcs-coupon-constraints.md)
- [Discount Limits](./whmcs-discount-limits.md)
- [Bundle Pricing](./whmcs-bundle-pricing.md)
