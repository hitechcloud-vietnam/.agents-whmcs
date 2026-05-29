# WHMCS Pro-Rated Invoicing Documentation

## Overview

Pro-rated invoicing calculates charges based on the actual time a service is used, enabling accurate billing when customers upgrade, downgrade, or change services mid-billing cycle.

## How Pro-Rated Billing Works

### Basic Concept

```
Customer upgrades on day 15 of a 30-day billing cycle:

Old Plan: $20/month (days 1-15 = 15 days used)
New Plan: $30/month (days 15-30 = 15 days remaining)

Credit for unused old plan: (20/30) * 15 = $10
Charge for remaining new plan: (30/30) * 15 = $15
Net amount due: $5
```

## Configuration

### Enable Pro-Ration

Navigate to: **Configuration > General Settings > Billing**

```php
// Pro-Ration Configuration
$prorateConfig = [
    // General Settings
    'enabled' => true,
    'default_method' => 'daily',      // daily, exact, monthly
    'round_precision' => 2,           // Decimal places

    // Prorate Settings
    'prorate_on_upgrade' => true,
    'prorate_on_downgrade' => true,
    'prorate_on_cancel' => true,
    'prorate_on_add' => true,

    // Credit Settings
    'credit_existing_period' => true,
    'credit_method' => 'immediate',    // immediate, next_invoice
    'show_credit_line' => true,

    // Tax Settings
    'include_tax_in_prorate' => true,
    'tax_method' => 'same_as_original',

    // Minimum Charges
    'minimum_charge' => 0.01,
    'minimum_days' => 1,              // Charge for at least N days
    'waive_below_minimum' => true
];
```

### Pro-Rate Methods

```php
// Daily Pro-Ration (Most Common)
$dailyMethod = [
    'type' => 'daily',
    'description' => 'Divide monthly price by days in month',
    'formula' => 'price / days_in_month * days_used',
    'example' => '$30 / 30 * 15 = $15'
];

// Exact Pro-Ration
$exactMethod = [
    'type' => 'exact',
    'description' => 'Calculate exact seconds/days',
    'formula' => 'price / seconds_in_period * seconds_used',
    'example' => 'Precise to the second'
];

// Monthly Average
$monthlyMethod = [
    'type' => 'monthly',
    'description' => 'Use 30-day month standard',
    'formula' => 'price / 30 * days_used',
    'example' => '$30 / 30 * 15 = $15 (always)'
];
```

## Pro-Ration Scenarios

### Upgrade

When a customer upgrades to a higher-tier plan:

```php
// Upgrade pro-ration calculation
$upgrade = [
    'old_plan_price' => 19.99,
    'old_plan_remaining_days' => 15,
    'old_plan_total_days' => 30,
    'new_plan_price' => 29.99,
    'new_plan_remaining_days' => 15,
    'new_plan_total_days' => 30
];

$credit = ($upgrade['old_plan_price'] / $upgrade['old_plan_total_days'])
          * $upgrade['old_plan_remaining_days'];
// Credit: $9.99

$charge = ($upgrade['new_plan_price'] / $upgrade['new_plan_total_days'])
          * $upgrade['new_plan_remaining_days'];
// Charge: $14.99

$net_due = $charge - $credit;
// Net: $5.00
```

### Downgrade

When a customer downgrades to a lower-tier plan:

```php
// Downgrade pro-ration calculation
$downgrade = [
    'old_plan_price' => 29.99,
    'old_plan_remaining_days' => 15,
    'new_plan_price' => 19.99,
    'new_plan_remaining_days' => 15
];

$credit = ($downgrade['old_plan_price'] / 30) * $downgrade['old_plan_remaining_days'];
// Credit: $14.99

$charge = ($downgrade['new_plan_price'] / 30) * $downgrade['new_plan_remaining_days'];
// Charge: $9.99

$credit_to_account = $credit - $charge;
// Credit to account: $5.00
// Applied to next invoice
```

### Mid-Month Add-On

Adding a new service mid-cycle:

```php
// Add-on pro-ration
$addon = [
    'addon_price' => 9.99,
    'days_in_month' => 30,
    'days_remaining' => 15
];

$charge = ($addon['addon_price'] / $addon['days_in_month'])
          * $addon['days_remaining'];
// Charge: $4.99
```

### Cancellation with Prorate

```php
// Cancellation with prorate refund
$cancellation = [
    'plan_price' => 29.99,
    'days_used' => 10,
    'days_in_period' => 30,
    'billing_cycle' => 'monthly',
    'cancel_type' => 'immediate'     // immediate, end_of_period
];

$unused_days = $cancellation['days_in_period'] - $cancellation['days_used'];
$refund = ($cancellation['plan_price'] / $cancellation['days_in_period'])
          * $unused_days;
// Refund: $19.99
```

## API Reference

### Calculate Pro-Rate

```http
POST /billing/prorate/calculate
```

**Request Body:**

```json
{
  "user_id": 12345,
  "action": "upgrade",
  "old_product_id": 100,
  "new_product_id": 101,
  "effective_date": "2024-01-15"
}
```

**Response:**

```json
{
  "calculation": {
    "old_product": {
      "id": 100,
      "name": "Basic Plan",
      "price": 19.99,
      "remaining_days": 15,
      "total_days": 30,
      "credit": 9.99
    },
    "new_product": {
      "id": 101,
      "name": "Premium Plan",
      "price": 29.99,
      "remaining_days": 15,
      "total_days": 30,
      "charge": 14.99
    },
    "tax": {
      "rate": 0.09,
      "credit_tax": 0.90,
      "charge_tax": 1.35,
      "net_tax": 0.45
    },
    "totals": {
      "credit": 9.99,
      "charge": 14.99,
      "tax": 0.45,
      "net_amount": 5.45
    }
  }
}
```

### Apply Pro-Rate Change

```http
POST /billing/prorate/apply
```

**Request Body:**

```json
{
  "user_id": 12345,
  "action": "upgrade",
  "old_product_id": 100,
  "new_product_id": 101,
  "effective_date": "2024-01-15",
  "create_invoice": true,
  "invoice_due_days": 7
}
```

### Preview Pro-Rate

```http
GET /billing/prorate/preview/{user_id}
```

## Invoice Display

### Pro-Rate Invoice Line Items

```json
{
  "line_items": [
    {
      "description": "Credit: Basic Plan (Jan 15-30) - Unused",
      "type": "credit",
      "amount": -9.99,
      "taxed": true
    },
    {
      "description": "Premium Plan Upgrade Proration (Jan 15-30)",
      "type": "charge",
      "amount": 14.99,
      "taxed": true
    },
    {
      "description": "Tax on Prorated Amount",
      "type": "tax",
      "amount": 0.45,
      "taxed": false
    }
  ],
  "totals": {
    "subtotal": 5.00,
    "tax": 0.45,
    "total": 5.45
  }
}
```

## Configuration Options

### Threshold Settings

```php
// Pro-ration threshold settings
$thresholds = [
    // Minimum amount to charge
    'minimum_charge_amount' => 0.50,

    // Minimum days to prorate
    'minimum_days' => 1,

    // Maximum credit cap
    'maximum_credit' => null,     // null = unlimited

    // Waive small charges
    'waive_below' => 1.00
];
```

### Grace Period

```php
// Grace period for mid-cycle changes
$gracePeriod = [
    'enabled' => true,
    'days_before_cycle_end' => 3,
    'upgrade_behavior' => 'wait_next_cycle',
    'downgrade_behavior' => 'prorate'
];
```

## Customer Portal

### Upgrade with Pro-Rate

**Client Area > My Services > Upgrade Plan**

```
+------------------------------------------------------------------+
|  Upgrade Your Plan                                               |
+------------------------------------------------------------------+
|                                                                  |
|  Current Plan: Basic ($19.99/month)                             |
|  New Plan: Premium ($29.99/month)                               |
|                                                                  |
|  Prorated Billing:                                              |
|  +--------------------------------------------------------------+|
|  | Credit for unused Basic Plan:      -$9.99                    ||
|  | Charge for Premium (15 days):     +$14.99                    ||
|  | ------------------------------------------------              ||
|  | Subtotal:                       $5.00                         ||
|  | Tax (9%):                      +$0.45                         ||
|  | ====================================                          ||
|  | Amount Due Today:               $5.45                         ||
|  | Next Invoice (Feb 1):         $29.99                         ||
|  +--------------------------------------------------------------+|
|                                                                  |
|  [Confirm Upgrade] [Cancel]                                       |
+------------------------------------------------------------------+
```

### Downgrade with Credit

**Client Area > My Services > Downgrade Plan**

```
+------------------------------------------------------------------+
|  Downgrade Your Plan                                              |
+------------------------------------------------------------------+
|                                                                  |
|  Current Plan: Premium ($29.99/month)                           |
|  New Plan: Basic ($19.99/month)                                  |
|                                                                  |
|  Prorated Credit:                                               |
|  +--------------------------------------------------------------+|
|  | Credit for unused Premium:       $14.99                      ||
|  | Less: Basic Plan charge:        -$9.99                       ||
|  | ======================================                         ||
|  | Credit to Account:             $5.00                         ||
|  | Applied to next invoice         ($5.00 on Feb 1)            ||
|  +--------------------------------------------------------------+|
|                                                                  |
|  Future Billing:                                                 |
|  Starting Feb 1: $19.99/month                                   |
|                                                                  |
|  [Confirm Downgrade] [Cancel]                                    |
+------------------------------------------------------------------+
```

## Webhooks

### Pro-Ration Webhooks

| Event | Description |
|-------|-------------|
| `ProrateCalculated` | Pro-rate amount calculated |
| `ProrateApplied` | Pro-rate change completed |
| `CreditApplied` | Credit applied to account |

### Webhook Payload

```json
{
  "event": "ProrateApplied",
  "timestamp": "2024-01-15T10:30:00Z",
  "data": {
    "user_id": 12345,
    "action": "upgrade",
    "old_product_id": 100,
    "new_product_id": 101,
    "credit_amount": 9.99,
    "charge_amount": 14.99,
    "net_amount": 5.45,
    "invoice_id": "INV-12345"
  }
}
```

## Troubleshooting

### Common Issues

| Issue | Cause | Solution |
|-------|-------|----------|
| Wrong amount calculated | Incorrect days | Check billing cycle |
| Credit not appearing | Config disabled | Enable credit_existing_period |
| Tax mismatch | Tax calculation error | Verify tax settings |
| Negative invoice | Over-credit | Check maximum_credit |

### Debug Commands

```bash
# Calculate pro-rate
whmcscli billing prorate --action=upgrade --user_id=12345 \
    --old_product=100 --new_product=101 --date=2024-01-15

# View pro-rate settings
whmcscli billing prorate-config

# Test calculation
whmcscli billing prorate-test --price=29.99 --days=15 --total=30
```

## Best Practices

1. **Be transparent** - Show credit and charge separately
2. **Set minimum thresholds** - Avoid micro-transactions
3. **Communicate clearly** - Explain prorated amounts to customers
4. **Test calculations** - Verify accuracy with sample data
5. **Handle edge cases** - Grace period, leap years, etc.
6. **Document policies** - Make proration rules clear in ToS

## See Also

- [Recurring Invoice](./whmcs-recurring-invoice.md)
- [Tiered Pricing](./whmcs-tiered-pricing.md)
- [Volume Discounts](./whmcs-volume-discounts.md)
