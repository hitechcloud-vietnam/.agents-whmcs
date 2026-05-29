# WHMCS Billing Cycle Options Documentation

## Overview

Billing cycles define the frequency and timing of invoice generation for recurring products and services. WHMCS supports flexible billing cycle configurations.

## Available Billing Cycles

### Supported Cycles

| Cycle | Code | Duration | Best For |
|-------|------|----------|----------|
| One Time | onetime | N/A | Non-recurring |
| Monthly | monthly | 30 days | Standard subscriptions |
| Quarterly | quarterly | 90 days | Quarterly billing |
| Semi-Annually | semiannually | 180 days | 6-month terms |
| Annually | annually | 365 days | Annual subscriptions |
| Biennially | biennially | 730 days | 2-year terms |
| Triennially | triennially | 1095 days | 3-year terms |

## Configuration

### Global Billing Cycle Settings

Navigate to: **Configuration > General Settings > Billing**

```php
// Billing Cycle Configuration
$billingCycleConfig = [
    // Available Cycles
    'available_cycles' => [
        'monthly',
        'quarterly',
        'semiannually',
        'annually'
    ],

    // Default Cycle
    'default_cycle' => 'monthly',

    // Cycle Naming
    'cycle_names' => [
        'onetime' => 'One Time',
        'monthly' => 'Monthly',
        'quarterly' => 'Quarterly',
        'semiannually' => 'Semi-Annually',
        'annually' => 'Annually',
        'biennially' => 'Biennially',
        'triennially' => 'Triennially'
    ],

    // Pricing
    'discount_by_cycle' => [
        'monthly' => 0,              // Base price
        'quarterly' => 5,           // 5% discount
        'semiannually' => 10,        // 10% discount
        'annually' => 15,            // 15% discount
        'biennially' => 20,          // 20% discount
        'triennially' => 25          // 25% discount
    ],

    // Generation Settings
    'days_before_due' => 7,
    'generation_time' => '00:00',
    'invoice_prefix' => 'INV-'
];
```

### Product Billing Cycle Settings

```php
// Product-specific cycle configuration
$productCycles = [
    'hosting' => [
        'available_cycles' => [
            'monthly' => ['price' => 9.99, 'setup' => 0],
            'quarterly' => ['price' => 27.99, 'setup' => 0],
            'annually' => ['price' => 99.99, 'setup' => 0]
        ],
        'default_cycle' => 'monthly',
        'allow_recurring' => true,
        'recurring_only' => false
    ],
    'domain' => [
        'available_cycles' => [
            'annually' => ['price' => 12.99, 'setup' => 0],
            'biennially' => ['price' => 24.99, 'setup' => 0],
            'triennially' => ['price' => 34.99, 'setup' => 0]
        ],
        'default_cycle' => 'annually',
        'allow_recurring' => true,
        'minimum_cycle' => 'annually'
    ],
    'ssl' => [
        'available_cycles' => [
            'annually' => ['price' => 49.99, 'setup' => 0],
            'biennially' => ['price' => 94.99, 'setup' => 0]
        ],
        'default_cycle' => 'annually'
    ]
];
```

## Pricing Models

### Standard Pricing

```php
// Standard pricing per cycle
$standardPricing = [
    'monthly' => 9.99,
    'quarterly' => 27.99,      // 9.99 * 3 = 29.97 (3% discount)
    'semiannually' => 54.99,  // 9.99 * 6 = 59.94 (8% discount)
    'annually' => 99.99       // 9.99 * 12 = 119.88 (17% discount)
];
```

### Discount Pricing

```php
// Percentage discount by cycle
$discountPricing = [
    'type' => 'percentage_discount',
    'base_price' => 9.99,
    'discounts' => [
        'monthly' => 0,
        'quarterly' => 5,
        'semiannually' => 10,
        'annually' => 15,
        'biennially' => 20,
        'triennially' => 25
    ],
    'example' => [
        'monthly' => '$9.99 (no discount)',
        'annually' => '$8.49/month ($101.88 billed annually, 15% off)'
    ]
];
```

### Tiered Pricing

```php
// Price varies by cycle selection
$tieredPricing = [
    'monthly' => [
        'price' => 9.99,
        'setup_fee' => 0,
        'total_first' => 9.99
    ],
    'annually' => [
        'price' => 99.99,
        'setup_fee' => 0,
        'setup_fee_waived' => true,
        'effective_monthly' => 8.33,
        'savings' => 19.89,
        'savings_percentage' => 17
    ]
];
```

## Cycle Configuration

### Monthly Cycle

```php
$monthlyCycle = [
    'code' => 'monthly',
    'days' => 30,
    'invoice_days_before' => 7,
    'prorate_on_upgrade' => true,
    'allow_downgrade' => true,
    'auto_renew' => true
];
```

### Annual Cycle

```php
$annualCycle = [
    'code' => 'annually',
    'days' => 365,
    'invoice_days_before' => 14,
    'prorate_on_upgrade' => true,
    'prorate_on_downgrade' => true,
    'early_renewal_allowed' => true,
    'renewal_discount' => 0
];
```

### Custom Cycle

```php
// Custom billing cycle
$customCycle = [
    'code' => 'custom_18months',
    'name' => '18 Months',
    'days' => 548,
    'price' => 140.00,
    'invoice_days_before' => 14,
    'enabled' => true
];
```

## API Reference

### Get Billing Cycles

```http
GET /billing/cycles
```

**Response:**

```json
{
  "cycles": [
    {
      "code": "monthly",
      "name": "Monthly",
      "days": 30,
      "available": true,
      "price" => 9.99,
      "setup_fee" => 0
    },
    {
      "code": "annually",
      "name": "Annually",
      "days": 365,
      "available": true,
      "price" => 99.99,
      "setup_fee" => 0,
      "savings" => 19.89,
      "savings_percentage" => 17
    }
  ]
}
```

### Get Cycle Pricing

```http
GET /billing/cycles/{code}/pricing
```

**Query Parameters:**
- `product_id`: Filter by product
- `user_id`: Filter by user

**Response:**

```json
{
  "cycle_code": "annually",
  "pricing": {
    "base_price" => 99.99,
    "setup_fee" => 0,
    "discount_percentage" => 15,
    "effective_monthly_rate" => 8.33,
    "total_billed" => 99.99,
    "savings_vs_monthly" => 19.89
  }
}
```

### Update Cycle Pricing

```http
PATCH /billing/cycles/{code}/pricing
```

**Request Body:**

```json
{
  "price": 109.99,
  "setup_fee": 0,
  "enabled": true,
  "discount_percentage": 10
}
```

## Customer Selection

### Display Pricing Table

**Client Area > Product Selection**

```
+------------------------------------------------------------------+
|  Premium Hosting Plan                                              |
+------------------------------------------------------------------+
|                                                                  |
|  +--------------------------------------------------------------+|
|  | Billing Cycle | Price       | Savings      | Select           ||
|  |---------------|-------------|--------------|------------------||
|  | Monthly       | $9.99/mo   | -            | [ ]              ||
|  | Quarterly     | $27.99/qtr | 5% off       | [ ]              ||
|  | Annually      | $99.99/yr  | 17% off      | [x]              ||
|  +--------------------------------------------------------------+|
|                                                                  |
|  Total: $99.99 billed annually ($8.33/month equivalent)         |
|                                                                  |
+------------------------------------------------------------------+
```

### Change Billing Cycle

**Client Area > My Services > Manage > Change Billing Cycle**

```
+------------------------------------------+
| Change Billing Cycle                      |
+------------------------------------------+
| Current Cycle: Monthly ($9.99)           |
| New Cycle: [Annually ($99.99)_______]  |
|                                          |
| Prorated Adjustment:                     |
| Credit for remaining 15 days: -$4.99    |
| Prorated charge for new plan: +$4.12    |
| ----------------------------------------|
| Amount Due Today: -$0.87 (credit)       |
| Next Invoice: January 1, 2025 ($99.99)  |
|                                          |
| [Confirm Change]                         |
+------------------------------------------+
```

## Automation

### Invoice Generation Schedule

```php
// Automation settings per cycle
$automationConfig = [
    'monthly' => [
        'invoice_days_before' => 7,
        'reminder_days' => [3, 1],
        'suspend_days' => 14,
        'terminate_days' => 30
    ],
    'quarterly' => [
        'invoice_days_before' => 14,
        'reminder_days' => [7, 3],
        'suspend_days' => 14,
        'terminate_days' => 30
    ],
    'annually' => [
        'invoice_days_before' => 14,
        'reminder_days' => [14, 7, 3],
        'suspend_days' => 21,
        'terminate_days' => 45
    ]
];
```

### Custom Schedule

```php
// Custom billing schedule
$customSchedule = [
    'enabled' => true,
    'generation_time' => '00:00',
    'batching' => [
        'enabled' => true,
        'batch_size' => 100,
        'pause_between' => 60  // seconds
    ],
    'retry' => [
        'enabled' => true,
        'max_attempts' => 3,
        'retry_delay' => 3600  // 1 hour
    ]
];
```

## Renewal Handling

### Renewal Notification Schedule

```php
// Renewal notification schedule
$renewalNotifications = [
    'email_templates' => [
        'invoice_created' => true,
        'reminder_1' => ['days' => 14, 'template' => 'renewal_reminder_14'],
        'reminder_2' => ['days' => 7, 'template' => 'renewal_reminder_7'],
        'reminder_3' => ['days' => 3, 'template' => 'renewal_reminder_3'],
        'reminder_final' => ['days' => 1, 'template' => 'renewal_reminder_final'],
        'expiring_today' => ['days' => 0, 'template' => 'domain_expiring']
    ],
    'auto_renew' => [
        'enabled' => true,
        'attempt_payment_before_expiry' => 7,
        'payment_methods' => ['credit_card', 'paypal']
    ]
];
```

### Grace Period by Cycle

```php
// Grace period configuration
$gracePeriods = [
    'monthly' => [
        'grace_days' => 3,
        'redemption_days' => 30,
        'suspend_days' => 7
    ],
    'annually' => [
        'grace_days' => 14,
        'redemption_days' => 30,
        'suspend_days' => 21
    ]
];
```

## Reporting

### Cycle Distribution Report

```http
GET /billing/reports/cycle-distribution
```

**Response:**

```json
{
  "period": "2024-01",
  "distribution": [
    {"cycle" => "monthly", "count" => 500, "percentage" => 45, "revenue" => 4995.00},
    {"cycle" => "quarterly", "count" => 200, "percentage" => 18, "revenue" => 5598.00},
    {"cycle" => "annually", "count" => 300, "percentage" => 27, "revenue" => 29997.00},
    {"cycle" => "biennially", "count" => 100, "percentage" => 9, "revenue" => 9999.00}
  ],
  "summary": {
    "total_customers" => 1100,
    "total_mrr" => 11589.00,
    "total_arr" => 139068.00
  }
}
```

### Revenue by Cycle

```http
GET /billing/reports/revenue-by-cycle
```

## Troubleshooting

### Common Issues

| Issue | Cause | Solution |
|-------|-------|----------|
| Invoice not generating | Cron not running | Check cron configuration |
| Wrong price | Price not updated | Verify product pricing |
| Cycle not available | Cycle disabled | Enable in configuration |
| Prorate mismatch | Days calculation | Check prorate settings |

### Debug Commands

```bash
# List billing cycles
whmcscli billing cycles

# Check cycle pricing
whmcscli billing cycle-price --cycle=annually --product=100

# View upcoming renewals
whmcscli billing upcoming --days=7

# Force invoice generation
whmcscli billing generate --user_id=12345 --cycle=monthly
```

## See Also

- [Recurring Invoice](./whmcs-recurring-invoice.md)
- [Payment Terms](./whmcs-payment-terms.md)
- [Volume Discounts](./whmcs-volume-discounts.md)
