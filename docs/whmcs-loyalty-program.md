# WHMCS Loyalty Program Documentation

## Overview

Loyalty programs reward repeat customers with exclusive benefits, discounts, and perks based on their tenure and spending history.

## Configuration

### Enable Loyalty Program

Navigate to: **Configuration > General Settings > Loyalty Program**

```php
// Loyalty Program Configuration
$loyaltyConfig = [
    'enabled' => true,

    // Program Settings
    'program_name' => 'Rewards Program',
    'default_tier' => 'bronze',

    // Enrollment
    'auto_enroll' => true,
    'enroll_on_first_purchase' => true,
    'manual_enrollment' => true,

    // Benefits
    'enable_tier_benefits' => true,
    'enable_points' => true,
    'enable_exclusive_offers' => true
];
```

## Loyalty Tiers

### Tier Structure

```php
// Loyalty tier levels
$loyaltyTiers = [
    'bronze' => [
        'name' => 'Bronze',
        'min_points' => 0,
        'min_lifetime_value' => 0,
        'benefits' => [
            'discount' => 0,
            'free_shipping_threshold' => 500,
            'birthday_discount' => 0,
            'priority_support' => false,
            'early_access' => false
        ]
    ],
    'silver' => [
        'name' => 'Silver',
        'min_points' => 500,
        'min_lifetime_value' => 500,
        'benefits' => [
            'discount' => 5,
            'free_shipping_threshold' => 250,
            'birthday_discount' => 10,
            'priority_support' => false,
            'early_access' => false
        ]
    ],
    'gold' => [
        'name' => 'Gold',
        'min_points' => 2000,
        'min_lifetime_value' => 2000,
        'benefits' => [
            'discount' => 10,
            'free_shipping_threshold' => 100,
            'birthday_discount' => 15,
            'priority_support' => true,
            'early_access' => true
        ]
    ],
    'platinum' => [
        'name' => 'Platinum',
        'min_points' => 5000,
        'min_lifetime_value' => 10000,
        'benefits' => [
            'discount' => 15,
            'free_shipping_threshold' => 0,
            'birthday_discount' => 20,
            'priority_support' => true,
            'early_access' => true,
            'dedicated_account_manager' => true
        ]
    ]
];
```

### Tier Upgrade Rules

```php
// Automatic tier upgrades
$tierUpgrades = [
    'automatic_upgrade' => true,
    'upgrade_check_frequency' => 'daily',
    'require_manual_review' => false,
    'grace_period_days' => 30,              // Days to reach next tier before downgrade
    'maintain_tier_for_days' => 365          // Must maintain tier for this long
];
```

## Benefits by Tier

### Discount Benefits

```php
// Tier-specific discounts
$tierDiscounts = [
    'bronze' => ['discount' => 0],
    'silver' => ['discount' => 5],
    'gold' => ['discount' => 10],
    'platinum' => ['discount' => 15]
];
```

### Free Shipping

```php
// Free shipping thresholds
$freeShipping = [
    'bronze' => ['threshold' => 500.00],
    'silver' => ['threshold' => 250.00],
    'gold' => ['threshold' => 100.00],
    'platinum' => ['threshold' => 0]           // Always free
];
```

### Birthday Rewards

```php
// Birthday discounts
$birthdayRewards = [
    'enabled' => true,
    'discount_type' => 'percentage',
    'tiers' => [
        'bronze' => ['discount' => 5],
        'silver' => ['discount' => 10],
        'gold' => ['discount' => 15],
        'platinum' => ['discount' => 20]
    ],
    'valid_days' => 7,                       // 7 days around birthday
    'auto_email' => true,
    'coupon_code' => 'auto_generated'
];
```

### Priority Support

```php
// Support benefits
$supportBenefits = [
    'silver' => [
        'queue_priority' => 1,              // Jump ahead 1 position
        'response_time_sla' => '4_hours'
    ],
    'gold' => [
        'queue_priority' => 5,
        'response_time_sla' => '2_hours',
        'dedicated_queue' => false
    ],
    'platinum' => [
        'queue_priority' => 10,
        'response_time_sla' => '1_hour',
        'dedicated_queue' => true
    ]
];
```

## Points System

### Earning Points

```php
// Points earning rules
$earnPoints = [
    'per_dollar_spent' => 1,                // 1 point per $1
    'per_review' => 50,                     // Points for product review
    'per_referral' => 200,                  // Points for referral signup
    'signup_bonus' => 100,                  // Points for joining
    'birthday_points' => 50,                 // Points for birthday
    'anniversary_points' => 100               // Points per year anniversary
];
```

### Points Redemption

```php
// Points redemption
$redeemPoints = [
    'points_per_dollar' => 100,              // 100 points = $1
    'minimum_redeem' => 500,                // Min 500 points to redeem
    'max_discount_percent' => 20,            // Max 20% off with points
    'max_discount_amount' => 100,            // Max $100 discount
    'expiry_months' => 12                   // Points expire after 12 months
];
```

### Points Value

```php
// Points value configuration
$pointsValue = [
    'earn_rate' => 1,                       // Points per $1
    'redeem_rate' => 100,                   // Points = $1
    'dollar_value_per_point' => 0.01,        // Each point = $0.01
    'round_redeem' => true
];
```

## API Reference

### Get Customer Loyalty Status

```http
GET /clients/{client_id}/loyalty
```

**Response:**

```json
{
  "client_id": 12345,
  "program_status": "active",
  "tier" => {
    "id" => "gold",
    "name" => "Gold",
    "points" => 2500,
    "lifetime_value" => 2500.00,
    "member_since" => "2022-01-15",
    "next_tier" => "platinum",
    "points_to_next_tier" => 2500
  },
  "benefits" => {
    "discount_percentage" => 10,
    "free_shipping_threshold" => 100.00,
    "birthday_discount" => 15,
    "priority_support" => true
  },
  "points" => {
    "balance" => 2500,
    "expires_soon" => 0,
    "expired" => 0,
    "lifetime_earned" => 5000,
    "lifetime_redeemed" => 2500
  }
}
```

### Get Tier Benefits

```http
GET /loyalty/tiers/{tier_id}
```

**Response:**

```json
{
  "tier_id": "gold",
  "name": "Gold",
  "requirements" => {
    "min_points" => 2000,
    "min_lifetime_value" => 2000
  },
  "benefits" => {
    "discount_percentage" => 10,
    "free_shipping_threshold" => 100.00,
    "birthday_discount" => 15,
    "early_access_days" => 7,
    "priority_support" => true
  },
  "progress" => {
    "current_points" => 2500,
    "next_tier" => "platinum",
    "points_needed" => 2500
  }
}
```

### Award Points

```http
POST /clients/{client_id}/loyalty/points
```

**Request Body:**

```json
{
  "points" => 100,
  "type" => "purchase",
  "reference" => "ORD-12345",
  "description" => "Points for order",
  "expires_at" => "2025-01-15"
}
```

### Redeem Points

```http
POST /clients/{client_id}/loyalty/redeem
```

**Request Body:**

```json
{
  "points" => 500,
  "reward_type" => "account_credit",
  "order_id" => "ORD-67890"
}
```

**Response:**

```json
{
  "success" => true,
  "points_redeemed" => 500,
  "credit_amount" => 5.00,
  "remaining_points" => 2000
}
```

## Tier Progression

### Progress Calculation

```php
// Calculate tier progress
$tierProgress = [
    'current_tier' => 'silver',
    'current_points' => 800,
    'tiers' => [
        'bronze' => ['threshold' => 0, 'status' => 'exceeded'],
        'silver' => ['threshold' => 500, 'status' => 'current'],
        'gold' => ['threshold' => 2000, 'status' => 'in_progress', 'progress' => 15],
        'platinum' => ['threshold' => 5000, 'status' => 'locked']
    ]
];
```

### Upgrade Notification

```php
// Tier upgrade notifications
$upgradeNotifications = [
    'notify_on_upgrade' => true,
    'notify_near_upgrade' => true,
    'days_before_upgrade' => [30, 7],
    'email_template_upgrade' => 'loyalty_tier_upgrade',
    'email_template_reminder' => 'loyalty_tier_reminder'
];
```

## Customer Portal

### Loyalty Dashboard

**Client Area > My Account > Rewards**

```
+------------------------------------------------------------------+
|  Rewards Program - Gold Member                                    |
+------------------------------------------------------------------+
|                                                                  |
|  Points Balance: 2,500                                          |
|  Points Value: $25.00 credit                                    |
|                                                                  |
|  +--------------------------------------------------------------+|
|  | Next Reward Status                                          ||
|  | Platinum (500 points away)                                 ||
|  | [================>                    ] 83% to Platinum     ||
|  +--------------------------------------------------------------+|
|                                                                  |
|  Your Benefits:                                                 |
|  - 10% discount on all orders                                 |
|  - Free shipping on orders over $100                          |
|  - 15% birthday discount                                       |
|  - Priority support                                            |
|  - Early access to new products                               |
|                                                                  |
|  Recent Activity:                                              |
|  +--------------------------------------------------------------+|
|  | Date       | Activity                   | Points             ||
|  |------------|---------------------------|--------------------||
|  | Jan 15    | Order purchase             | +250               ||
|  | Jan 10    | Product review             | +50                ||
|  | Jan 5     | Points redeemed            | -100               ||
|  +--------------------------------------------------------------+|
|                                                                  |
|  [Redeem Points] [View All Benefits]                           |
+------------------------------------------------------------------+
```

## Tier Benefits Table

```
+------------------------------------------------------------------+
|  Rewards Program - Compare Tiers                                   |
+------------------------------------------------------------------+
|                              | Bronze  | Silver | Gold  | Platinum|
|------------------------------|---------|--------|-------|--------|
| Discount on Orders           | 0%      | 5%     | 10%   | 15%    |
| Free Shipping Threshold      | $500     | $250    | $100  | $0     |
| Birthday Discount            | 0%       | 10%     | 15%   | 20%    |
| Priority Support            | -        | -       | Yes   | Yes    |
| Early Access               | -        | -       | Yes   | Yes    |
| Dedicated Account Manager   | -        | -       | -     | Yes    |
| Anniversary Bonus          | -        | $10     | $25   | $50    |
+------------------------------------------------------------------+
```

## Reporting

### Loyalty Program Report

```http
GET /billing/reports/loyalty
```

**Response:**

```json
{
  "period": "2024-01",
  "summary": {
    "total_members" => 1000,
    "tier_distribution" => [
      {"tier" => "bronze", "members" => 400, "percentage" => 40},
      {"tier" => "silver", "members" => 350, "percentage" => 35},
      {"tier" => "gold", "members" => 200, "percentage" => 20},
      {"tier" => "platinum", "members" => 50, "percentage" => 5}
    ],
    "tier_upgrades" => 50,
    "tier_downgrades" => 10
  },
  "financial_impact" => {
    "total_discount_given" => 5000.00,
    "avg_discount_per_member" => 5.00,
    "program_cost" => 5000.00,
    "program_revenue_attributed" => 50000.00,
    "roi_percentage" => 900
  }
}
```

## Troubleshooting

### Common Issues

| Issue | Cause | Solution |
|-------|-------|----------|
| Points not accruing | Purchase not tracked | Check integration |
| Tier not upgrading | Criteria not met | Verify tier requirements |
| Benefits not applying | Cache issue | Clear cache |
| Points expired | Points expired | Review expiry settings |

### Debug Commands

```bash
# View customer tier
whmcscli loyalty status --client_id=12345

# Award points manually
whmcscli loyalty award --client_id=12345 --points=100 --reason="bonus"

# Check tier progress
whmcscli loyalty progress --client_id=12345
```

## Best Practices

1. **Clear value proposition** - Show tangible benefits
2. **Achievable tiers** - Make first upgrades easy
3. **Communicate status** - Keep customers informed
4. **Reward engagement** - Points for more than purchases
5. **Expire points wisely** - Encourage redemption
6. **Monitor ROI** - Track program effectiveness

## See Also

- [Reward Points](./whmcs-reward-points.md)
- [Credit Policy](./whmcs-credit-policy.md)
- [Customer Tiers](../customers/customer-tiers.md)
