# WHMCS Late Fee Rules Documentation

## Overview

Late fee rules define automatic charges applied to overdue invoices, encouraging timely payment and compensating for collection efforts.

## Configuration

### Enable Late Fees

Navigate to: **Configuration > General Settings > Late Fees**

```php
// Late Fee Configuration
$lateFeeConfig = [
    'enabled' => true,
    'auto_apply' => true,
    'fee_after_days' => 3,
    'calculate_on_tax' => false,

    // Fee Types
    'fee_type' => 'fixed',          // fixed, percentage, tiered
    'rounding' => 'nearest',       // nearest, up, down
    'round_precision' => 2,

    // Limits
    'minimum_fee' => 0.00,
    'maximum_fee' => 100.00,
    'maximum_fee_percentage' => 25,

    // Exceptions
    'exclude_first_invoice' => true,
    'exclude_credit_invoices' => true,
    'exclude_specific_products' => true
];
```

## Fee Types

### Fixed Fee

```php
// Fixed late fee
$fixedFee = [
    'type' => 'fixed',
    'amount' => 5.00,
    'applies_once' => true,        // One-time fee
    // OR
    'applies_daily' => false,       // Per occurrence
    'applies_weekly' => false
];
```

### Percentage Fee

```php
// Percentage of invoice amount
$percentageFee = [
    'type' => 'percentage',
    'rate' => 2.0,                // 2% of invoice
    'minimum' => 5.00,
    'maximum' => 50.00,
    'applies_to_total' => true,     // Total or subtotal
    'applies_to_tax' => false
];
```

### Tiered Fee

```php
// Tiered late fees based on overdue days
$tieredFee = [
    'type' => 'tiered',
    'tiers' => [
        ['days_overdue' => 1, 'type' => 'fixed', 'amount' => 5.00],
        ['days_overdue' => 7, 'type' => 'fixed', 'amount' => 10.00],
        ['days_overdue' => 14, 'type' => 'percentage', 'rate' => 1.0],
        ['days_overdue' => 30, 'type' => 'percentage', 'rate' => 2.0]
    ]
];
```

### Compound Fee

```php
// Fixed fee plus percentage
$compoundFee = [
    'type' => 'compound',
    'fixed_amount' => 5.00,
    'percentage_rate' => 1.0,
    'minimum' => 5.00,
    'maximum' => 100.00
];
```

## Late Fee Rules

### Standard Rules

```php
// Standard late fee configuration
$standardLateFeeRules = [
    'name' => 'Standard Late Fee',
    'days_after_due' => 3,
    'fee_type' => 'fixed',
    'fee_amount' => 5.00,
    'applies_recurring' => true,
    'recurring_interval' => 7,      // Every 7 days
    'maximum_applications' => 4,    // Max 4 times
    'cap_amount' => 25.00          // Max total fee
];
```

### Percentage-Based Rules

```php
// Percentage late fee
$percentageLateFeeRules = [
    'name' => 'Percentage Late Fee',
    'days_after_due' => 7,
    'fee_type' => 'percentage',
    'fee_percentage' => 2.0,
    'minimum_fee' => 5.00,
    'maximum_fee' => 50.00,
    'apply_once' => true
];
```

### Escalating Rules

```php
// Escalating late fees
$escalatingRules = [
    'name' => 'Escalating Late Fee',
    'escalation_enabled' => true,
    'escalation_schedule' => [
        ['days' => 3, 'fixed_fee' => 5.00],
        ['days' => 7, 'fixed_fee' => 10.00],
        ['days' => 14, 'fixed_fee' => 15.00, 'percentage' => 1.0],
        ['days' => 30, 'fixed_fee' => 25.00, 'percentage' => 2.0]
    ],
    'cap_type' => 'per_escalation',  // per_escalation, total
    'cap_amount' => 100.00
];
```

## Customer-Specific Fees

### By Customer Group

```php
// Customer group late fees
$groupLateFees = [
    'standard' => [
        'days_after_due' => 3,
        'fee_type' => 'fixed',
        'fee_amount' => 5.00
    ],
    'preferred' => [
        'days_after_due' => 7,
        'fee_type' => 'fixed',
        'fee_amount' => 2.50
    ],
    'enterprise' => [
        'days_after_due' => 14,
        'fee_type' => 'fixed',
        'fee_amount' => 0.00,
        'reminder_only' => true
    ]
];
```

### By Invoice Amount

```php
// Invoice amount-based fees
$amountBasedFees = [
    'small_invoice' => [
        'min_amount' => 0,
        'max_amount' => 50.00,
        'fee_type' => 'fixed',
        'fee_amount' => 2.50
    ],
    'medium_invoice' => [
        'min_amount' => 50.01,
        'max_amount' => 500.00,
        'fee_type' => 'fixed',
        'fee_amount' => 5.00
    ],
    'large_invoice' => [
        'min_amount' => 500.01,
        'max_amount' => null,
        'fee_type' => 'percentage',
        'fee_percentage' => 1.0,
        'minimum' => 10.00,
        'maximum' => 100.00
    ]
];
```

## Product Exemptions

### Exempt Products

```php
// Products exempt from late fees
$exemptProducts = [
    'product_ids' => [],
    'category_ids' => ['domain', 'addons-free'],
    'by_remaining_value' => false
];
```

### Custom Exemptions

```php
// Custom exemption rules
$customExemptions = [
    'first_invoice' => true,        // Waive for first invoice
    'first_overdue' => true,         // Waive for first overdue
    'good_standing_customers' => [
        'enabled' => true,
        'payment_history_months' => 6,
        'on_time_percentage' => 95,
        'fee_reduction' => 50        // 50% reduction
    ],
    'manual_exemptions' => [
        'enabled' => true,
        'require_approval' => true,
        'reason_required' => true
    ]
];
```

## API Reference

### Configure Late Fee Rule

```http
POST /billing/late-fee/rules
```

**Request Body:**

```json
{
  "name": "Standard Late Fee",
  "days_after_due": 3,
  "fee_type": "fixed",
  "fee_amount": 5.00,
  "minimum_fee": 0,
  "maximum_fee": 25.00,
  "applies_recurring": true,
  "recurring_interval": 7,
  "maximum_applications": 4,
  "enabled": true
}
```

### Update Late Fee Rule

```http
PATCH /billing/late-fee/rules/{rule_id}
```

### Calculate Late Fee

```http
POST /billing/late-fee/calculate
```

**Request Body:**

```json
{
  "invoice_id": "INV-12345",
  "apply_to_invoice": false
}
```

**Response:**

```json
{
  "invoice_id": "INV-12345",
  "invoice_amount": 100.00,
  "due_date": "2024-01-01",
  "days_overdue": 5,
  "late_fee": {
    "rule_id": "RULE-001",
    "rule_name": "Standard Late Fee",
    "fee_type": "fixed",
    "fee_amount": 5.00,
    "total_late_fee": 5.00
  },
  "invoice_total_with_fee": 105.00
}
```

### Apply Late Fee

```http
POST /billing/late-fee/apply/{invoice_id}
```

**Request Body:**

```json
{
  "waive_fee" => false,
  "reason" => "Late fee applied per terms"
}
```

### Waive Late Fee

```http
POST /billing/late-fee/waive/{invoice_id}
```

**Request Body:**

```json
{
  "reason": "Customer contacted us, good payment history"
}
```

## Late Fee Invoice Line

### Display Format

```php
// Late fee line item format
$lateFeeLine = [
    'description' => 'Late Payment Fee - Invoice INV-12345',
    'amount' => 5.00,
    'taxable' => false,
    'type' => 'late_fee',
    'reference_invoice' => 'INV-12345',
    'days_overdue' => 5,
    'fee_rule' => 'Standard Late Fee'
];
```

### Invoice Display

```html
<!-- Invoice Late Fee Section -->
<div class="invoice-late-fee">
    <h3>Late Fee Applied</h3>
    <p>Invoice INV-12345 was due on January 1, 2024</p>
    <p>5 days overdue as of January 6, 2024</p>
    <p class="fee-amount">Late Fee: $5.00</p>
    <p class="fee-notice">Late fees are non-refundable unless otherwise noted.</p>
</div>
```

## Automation

### Auto-Apply Configuration

```php
// Automation settings
$lateFeeAutomation = [
    'enabled' => true,
    'run_frequency' => 'daily',
    'run_time' => '00:00',
    'apply_rules' => true,
    'notify_before_apply' => false,
    'notify_after_apply' => true,

    // Notification
    'email_template' => 'late_fee_applied',
    'admin_notification' => true
];
```

### Notification Schedule

```php
// Late fee notifications
$lateFeeNotifications = [
    'before_apply' => [
        'enabled' => false,
        'days_before' => 1,
        'template' => 'late_fee_warning'
    ],
    'after_apply' => [
        'enabled' => true,
        'template' => 'late_fee_applied'
    ],
    'recurring_notice' => [
        'enabled' => true,
        'after_applications' => 2,
        'template' => 'late_fee_recurring_notice'
    ]
];
```

## Reporting

### Late Fee Report

```http
GET /billing/reports/late-fees
```

**Query Parameters:**
- `date_from`: Start date
- `date_to`: End date
- `group_by`: day, week, month

**Response:**

```json
{
  "period": "2024-01",
  "summary": {
    "total_late_fees" => 1500.00,
    "total_invoices_with_fees" => 150,
    "average_fee" => 10.00,
    "waived_fees" => 50.00
  },
  "by_day": [
    {"date" => "2024-01-03", "fees" => 50.00, "invoices" => 10},
    {"date" => "2024-01-04", "fees" => 75.00, "invoices" => 15}
  ],
  "revenue_impact" => {
    "gross_late_fees" => 1500.00,
    "waived" => 50.00,
    "collected" => 1450.00,
    "collection_rate" => 96.7
  }
}
```

## Customer Portal

### View Late Fees

**Client Area > Billing > Invoices > Invoice Details**

```
+------------------------------------------------------------------+
|  Invoice INV-12345                                               |
+------------------------------------------------------------------+
|  Status: OVERDUE (5 days)                                        |
|  Original Due Date: January 1, 2024                              |
|  Amount Due: $100.00                                            |
|                                                                  |
|  Late Fee Applied:                                               |
|  +--------------------------------------------------------------+|
|  | Date       | Description            | Amount                ||
|  |------------|------------------------|-----------------------||
|  | Jan 4      | Late Payment Fee        | $5.00               ||
|  +--------------------------------------------------------------+|
|                                                                  |
|  Total Due: $105.00                                             |
|                                                                  |
|  Note: Late fees are applied automatically for overdue invoices |
|        and are non-refundable.                                  |
+------------------------------------------------------------------+
```

## Waiving Fees

### Waive Criteria

```php
// Waiving criteria
$waiveCriteria = [
    'good_standing' => [
        'enabled' => true,
        'payment_history_months' => 6,
        'on_time_rate' => 90
    ],
    'first_occurrence' => true,         // Waive first late fee
    'circumstances' => [
        'bank_error' => true,
        'system_issue' => true,
        'customer_contact' => true,
        'payment_arrangement' => true
    ],
    'approval_required' => [
        'amount_over' => 25.00,
        'approver_role' => 'billing_manager'
    ]
];
```

### Manual Waiver

```http
POST /billing/late-fee/waive/{invoice_id}
```

**Request Body:**

```json
{
  "reason": "Good payment history, first late fee",
  "waived_by" => "admin",
  "approval_required" => false
}
```

## Troubleshooting

### Common Issues

| Issue | Cause | Solution |
|-------|-------|----------|
| Fee not applied | Automation not running | Check cron |
| Wrong amount | Rule misconfigured | Verify rule settings |
| Fee waived incorrectly | Over-permissive rules | Tighten waiver criteria |
| Customer disputes | Unclear policy | Update terms/conditions |

### Debug Commands

```bash
# Check late fee rules
whmcscli latefee rules

# Calculate fee for invoice
whmcscli latefee calculate --invoice_id=INV-12345

# Apply fee manually
whmcscli latefee apply --invoice_id=INV-12345

# View applied fees
whmcscli latefee list --client_id=12345

# Waive fee
whmcscli latefee waive --invoice_id=INV-12345 --reason="Good standing"
```

## Best Practices

1. **Start low** - Begin with small fees
2. **Communicate clearly** - Display late fee policy prominently
3. **Be consistent** - Apply fees uniformly
4. **Allow waivers** - For legitimate circumstances
5. **Monitor impact** - Track payment behavior changes
6. **Automate fairly** - Send warnings before fees

## See Also

- [Payment Terms](./whmcs-payment-terms.md)
- [Grace Period](./whmcs-grace-period.md)
- [Credit Policy](./whmcs-credit-policy.md)
