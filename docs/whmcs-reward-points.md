# WHMCS Reward Points System Documentation

## Overview

Reward points systems let customers earn points on purchases and other activities, which can be redeemed for discounts, credits, or prizes.

## Configuration

### Enable Reward Points

Navigate to: **Configuration > General Settings > Reward Points**

```php
// Reward Points Configuration
$rewardPointsConfig = [
    'enabled' => true,

    // Program Name
    'program_name' => 'Reward Points',
    'points_name' => 'Points',
    'currency_name' => 'Credits',

    // General Settings
    'require_enrollment' => false,
    'auto_enroll_on_purchase' => true,

    // Display
    'show_points_on_products' => true,
    'show_points_on_cart' => true,
    'show_points_on_invoices' => true,
    'points_to_currency_rate' => 0.01   // 100 points = $1
];
```

## Earning Points

### Purchase Earning

```php
// Points earned on purchases
$purchaseEarning = [
    'enabled' => true,
    'points_per_dollar' => 1,           // 1 point per $1 spent
    'minimum_purchase' => 0,
    'maximum_per_order' => 1000,          // Cap per order
    'exclude_shipping' => true,
    'exclude_tax' => true,
    'exclude_discounts' => false,
    'exclude_gift_cards' => true
];
```

### Bonus Earning

```php
// Bonus points for activities
$bonusEarning = [
    'signup_bonus' => [
        'enabled' => true,
        'points' => 100,
        'one_time' => true
    ],
    'review_bonus' => [
        'enabled' => true,
        'points' => 50,
        'per_product' => true,
        'max_reviews_per_month' => 5
    ],
    'referral_bonus' => [
        'enabled' => true,
        'referrer_points' => 200,
        'referee_points' => 100,
        'require_purchase' => true
    ],
    'newsletter_bonus' => [
        'enabled' => true,
        'points' => 25,
        'one_time' => true
    ]
];
```

### Time-Based Bonuses

```php
// Special earning opportunities
$timeBasedBonuses = [
    'birthday_bonus' => [
        'enabled' => true,
        'points' => 50,
        'valid_days' => 7              // 7 days around birthday
    ],
    'anniversary_bonus' => [
        'enabled' => true,
        'points' => 100,
        'bonus_per_year' => 10         // +10 points per year as customer
    ],
    'holiday_bonuses' => [
        'enabled' => true,
        'events' => [
            ['name' => 'New Year', 'multiplier' => 2],
            ['name' => 'Black Friday', 'multiplier' => 3]
        ]
    ]
];
```

## Redeeming Points

### Redemption Options

```php
// Points redemption configuration
$redemptionConfig = [
    'enabled' => true,
    'allow_partial_redemption' => true,
    'minimum_points' => 100,
    'maximum_points_per_order' => 1000,
    'max_discount_percent' => 50,       // Max 50% of order can be points

    // Redemption rates
    'points_to_dollar_rate' => 100,     // 100 points = $1
    'dollar_to_points_rate' => 1,         // $1 = 1 point

    // Auto redemption
    'auto_redeem_enabled' => false,
    'auto_redeem_threshold' => 5000,
    'auto_redeem_method' => 'credit'
];
```

### Redemption Tiers

```php
// Redeem points at different rates
$redemptionTiers = [
    'tier_1' => [
        'min_points' => 100,
        'max_points' => 499,
        'rate' => 100,                 // 100 points = $0.75
        'dollar_value' => 0.75
    ],
    'tier_2' => [
        'min_points' => 500,
        'max_points' => 999,
        'rate' => 100,                 // 100 points = $1.00
        'dollar_value' => 1.00
    ],
    'tier_3' => [
        'min_points' => 1000,
        'max_points' => null,
        'rate' => 100,                 // 100 points = $1.25
        'dollar_value' => 1.25
    ]
];
```

## API Reference

### Get Points Balance

```http
GET /clients/{client_id}/points
```

**Response:**

```json
{
  "client_id": 12345,
  "points_balance": 1500,
  "lifetime_earned": 5000,
  "lifetime_redeemed": 3500,
  "pending_points": 200,
  "expired_points": 0,
  "points_value" => {
    "currency_value" => 15.00,
    "rate" => "100 points = $1.00"
  }
}
```

### Get Points History

```http
GET /clients/{client_id}/points/history
```

**Response:**

```json
{
  "client_id": 12345,
  "history" => [
    {
      "id" => "PT-001",
      "type" => "earned",
      "points" => 100,
      "description" => "Purchase - Order #12345",
      "order_id" => "ORD-12345",
      "expires_at" => "2025-01-15",
      "created_at" => "2024-01-15T10:00:00Z"
    },
    {
      "id" => "PT-002",
      "type" => "redeemed",
      "points" => -500,
      "description" => "Redeemed for account credit",
      "credit_amount" => 5.00,
      "created_at" => "2024-01-10T10:00:00Z"
    }
  ],
  "pagination" => {...}
}
```

### Award Points

```http
POST /clients/{client_id}/points/award
```

**Request Body:**

```json
{
  "points" => 100,
  "type" => "bonus",
  "description" => "Customer service bonus",
  "expires_at" => "2025-01-15",
  "notify_customer" => true
}
```

### Redeem Points

```http
POST /clients/{client_id}/points/redeem
```

**Request Body:**

```json
{
  "points" => 500,
  "redemption_type" => "account_credit",
  "order_id" => null,
  "notify_customer" => true
}
```

**Response:**

```json
{
  "success" => true,
  "points_redeemed" => 500,
  "credit_amount" => 5.00,
  "new_balance" => 1000,
  "transaction_id" => "PT-003"
}
```

### Calculate Points Value

```http
POST /clients/{client_id}/points/calculate
```

**Request Body:**

```json
{
  "points" => 500,
  "apply_to_order_id" => "ORD-67890"
}
```

**Response:**

```json
{
  "points" => 500,
  "redemption_rate" => "100 points = $1.00",
  "credit_amount" => 5.00,
  "applied_to_order" => "ORD-67890",
  "order_total" => 100.00,
  "new_order_total" => 95.00
}
```

## Points Expiration

### Expiration Rules

```php
// Points expiration configuration
$expirationConfig = [
    'enabled' => true,
    'expiry_months' => 12,               // Points expire after 12 months
    'expiry_notification_days' => [30, 7, 1],
    'extend_on_activity' => true,          // Extend expiry on activity

    // FIFO vs LIFO
    'expiry_order' => 'fifo'             // First in, first out
];
```

### Points Expiry Calculation

```php
// Calculate expiring points
$expiryCalculation = [
    'point_batch' => 'PT-2023-001',
    'earned_date' => '2023-01-15',
    'expires_date' => '2024-01-15',
    'points' => 500,
    'days_until_expiry' => 30,
    'extendable' => true,
    'extend_on_purchase' => true
];
```

## Customer Actions

### Redeem for Discount

```php
// Redeem for discount at checkout
$redeemForDiscount = [
    'points_required' => 100,
    'discount_amount' => 1.00,
    'max_discount_percent' => 50,       // Max 50% off
    'minimum_order' => 10.00
];
```

### Redeem for Credit

```php
// Redeem for account credit
$redeemForCredit = [
    'points_required' => 100,
    'credit_amount' => 1.00,
    'credit_expires_months' => 12,
    'auto_apply_to_invoices' => false
];
```

### Redeem for Products

```php
// Redeem for free products
$redeemForProducts = [
    'enabled' => true,
    'products' => [
        ['product_id' => 100, 'points_required' => 500],
        ['addon_id' => 50, 'points_required' => 200]
    ]
];
```

## Customer Portal

### Points Dashboard

**Client Area > My Account > Reward Points**

```
+------------------------------------------------------------------+
|  Reward Points                                                   |
+------------------------------------------------------------------+
|                                                                  |
|  Your Balance: 1,500 Points                                    |
|  Value: $15.00                                                  |
|                                                                  |
|  Points Expiring Soon: 500 points (expires Jan 31, 2024)       |
|                                                                  |
|  +--------------------------------------------------------------+|
|  | Redeem Your Points                                           ||
|  |--------------------------------------------------------------||
|  | [500 points = $5.00 credit]                               ||
|  | [1000 points = $12.50 credit]                              ||
|  | [2000 points = $30.00 credit]                             ||
|  +--------------------------------------------------------------+|
|                                                                  |
|  Earn More Points:                                              |
|  - Write a product review: +50 points                        |
|  - Refer a friend: +200 points                                 |
|                                                                  |
|  [View All Ways to Earn]                                        |
+------------------------------------------------------------------+
```

### Points at Checkout

```
+------------------------------------------------------------------+
|  Checkout - Apply Points                                         |
+------------------------------------------------------------------+
|                                                                  |
|  Available Points: 1,500 (worth $15.00)                        |
|                                                                  |
|  Points to Apply: [_________]                                   |
|  (Min: 100, Max: 750 for this order)                           |
|                                                                  |
|  Your Discount: $7.50                                          |
|                                                                  |
|  [Apply Points]                                                |
+------------------------------------------------------------------+
```

## Points Notifications

### Email Notifications

```php
// Points notification settings
$notificationConfig = [
    'points_earned' => [
        'enabled' => true,
        'template' => 'points_earned',
        'include_balance' => true
    ],
    'points_redeemed' => [
        'enabled' => true,
        'template' => 'points_redeemed'
    ],
    'points_expiring' => [
        'enabled' => true,
        'notify_days_before' => [30, 7, 1],
        'template' => 'points_expiring'
    ],
    'tier_upgrade' => [
        'enabled' => true,
        'template' => 'points_tier_upgrade'
    ]
];
```

## Reporting

### Points Report

```http
GET /billing/reports/points
```

**Response:**

```json
{
  "period": "2024-01",
  "summary": {
    "total_points_earned" => 50000,
    "total_points_redeemed" => 40000,
    "net_points" => 10000,
    "total_redemptions_value" => 400.00,
    "avg_points_per_customer" => 50
  },
  "by_source" => [
    {"source" => "purchases", "points" => 35000},
    {"source" => "reviews", "points" => 5000},
    {"source" => "referrals", "points" => 8000},
    {"source" => "bonuses", "points" => 2000}
  ],
  "by_redemption" => [
    {"type" => "account_credit", "points" => 25000, "value" => 250.00},
    {"type" => "discount", "points" => 15000, "value" => 150.00}
  ],
  "expiring_soon" => {
    "points" => 5000,
    "customers_affected" => 100
  }
}
```

## Integration with Loyalty

### Combined Program

```php
// Integrate with loyalty tiers
$loyaltyIntegration = [
    'enabled' => true,
    'tier_bonus_multiplier' => [
        'bronze' => 1.0,
        'silver' => 1.25,
        'gold' => 1.5,
        'platinum' => 2.0
    ],
    'tier_redemption_bonus' => [
        'bronze' => 0,
        'silver' => 10,
        'gold' => 25,
        'platinum' => 50
    ]
];
```

## Troubleshooting

### Common Issues

| Issue | Cause | Solution |
|-------|-------|----------|
| Points not accruing | Module not configured | Check purchase integration |
| Points not applying | Checkout issue | Verify redemption settings |
| Expiry notification not sent | Cron not running | Check automation |
| Wrong value | Rate misconfigured | Verify conversion rate |

### Debug Commands

```bash
# View points balance
whmcscli points balance --client_id=12345

# Award points
whmcscli points award --client_id=12345 --points=100 --reason="bonus"

# Redeem points
whmcscli points redeem --client_id=12345 --points=500

# Check expiring points
whmcscli points expiring --days=30

# Calculate points value
whmcscli points value --points=1000
```

## Best Practices

1. **Simple value proposition** - Make points value clear
2. **Quick wins** - Give bonus for easy actions
3. **Reasonable expiration** - Don't expire too quickly
4. **Multiple redemption options** - Flexibility increases usage
5. **Communicate clearly** - Show points balance prominently
6. **Track ROI** - Measure program effectiveness

## See Also

- [Loyalty Program](./whmcs-loyalty-program.md)
- [Credit Policy](./whmcs-credit-policy.md)
- [Promotion Rules](./whmcs-promotion-rules.md)
