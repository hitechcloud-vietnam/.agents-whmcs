# WHMCS Recurring Invoice Documentation

## Overview

Recurring invoices in WHMCS automate the billing process for subscription-based products and services, ensuring timely billing and revenue consistency.

## Configuration

### Enable Recurring Invoices

Navigate to: **Configuration > System Settings > Automation Settings**

```php
// Recurring Invoice Configuration
$recurringConfig = [
    // Generation Settings
    'auto_generate' => true,
    'days_before_due' => 7,
    'generation_time' => '00:00',
    'invoice_prefix' => 'INV-',

    // Numbering
    'number_format' => '{year}{month}{sequence}',
    'sequence_reset' => 'monthly',
    'minimum_digits' => 4,

    // Payment Terms
    'default_due_days' => 7,
    'payment_methods' => ['credit_card', 'paypal', 'bank_transfer'],

    // Notifications
    'send_reminder_days' => [3, 1],
    'auto_charge' => true,
    'charge_attempts' => 3,
    'charge_attempt_days' => [0, 3, 5]
];
```

### Product Recurring Settings

```php
// Product recurring configuration
$productRecurring = [
    'recurring_enabled' => true,
    'billing_cycle' => 'monthly',
    'recurring_only' => false,
    'allow_custom_qty' => true,
    'setup_fee' => [
        'first_payment' => true,
        'amount' => 0
    ]
];
```

## Billing Cycles

### Supported Cycles

| Cycle | Code | Description |
|-------|------|-------------|
| One Time | onetime | No recurring |
| Monthly | monthly | Every month |
| Quarterly | quarterly | Every 3 months |
| Semi-Annually | semiannually | Every 6 months |
| Annually | annually | Every 12 months |
| Biennially | biennially | Every 24 months |
| Triennially | triennially | Every 36 months |

### Cycle Configuration

```php
// Billing cycle settings
$billingCycles = [
    'monthly' => [
        'days' => 30,
        'invoice_days_before' => 7,
        'prorate_on_upgrade' => true
    ],
    'quarterly' => [
        'days' => 90,
        'invoice_days_before' => 14,
        'prorate_on_upgrade' => true
    ],
    'annually' => [
        'days' => 365,
        'invoice_days_before' => 14,
        'prorate_on_upgrade' => true
    ]
];
```

## Invoice Generation

### Auto-Generation Process

```
1. Cron runs daily at configured time
2. Find all products due for renewal
3. Generate invoice for each
4. Apply any applicable credits
5. Send invoice to customer
6. Process auto-payment if enabled
7. Log generation results
```

### Manual Invoice Generation

```php
// Generate recurring invoice manually
$invoice = WHMCS\Billing\Invoice::generateRecurring([
    'user_id' => 12345,
    'product_id' => 67890,
    'billing_cycle' => 'monthly',
    'amount' => 19.99
]);
```

### Invoice Line Items

```php
// Invoice line item structure
$lineItem = [
    'description' => 'Premium Hosting - Monthly',
    'quantity' => 1,
    'unit_price' => 19.99,
    'tax' => 1.79,
    'total' => 21.78,
    'relid' => 67890,         // Related product ID
    'taxed' => true
];
```

## Recurring Billing Rules

### Grace Period Configuration

```php
// Grace period for overdue invoices
$gracePeriod = [
    'enabled' => true,
    'days' => 3,
    'suspend_on_expiry' => true,
    'suspend_days' => 14,
    'terminate_days' => 30,
    'retain_on_termination' => false
];
```

### Auto-Payment Rules

```php
// Auto-payment configuration
$autoPayment = [
    'enabled' => true,
    'payment_method_id' => 'default',
    'retry_failed' => true,
    'max_attempts' => 3,
    'retry_days' => [0, 3, 5, 7],
    'notify_before_attempt' => true,
    'notify_after_failure' => true,
    'suspend_on_failure' => true,
    'suspend_after_attempts' => 3
];
```

## API Reference

### List Recurring Invoices

```http
GET /billing/invoices/recurring
```

**Query Parameters:**

| Parameter | Type | Description |
|-----------|------|-------------|
| status | string | pending, paid, overdue, cancelled |
| user_id | int | Filter by customer |
| product_id | int | Filter by product |
| billing_cycle | string | Filter by cycle |

**Response:**

```json
{
  "invoices": [
    {
      "id": "INV-12345",
      "status": "pending",
      "user_id": 12345,
      "user_name": "John Doe",
      "amount": 19.99,
      "tax": 1.79,
      "total": 21.78,
      "billing_cycle": "monthly",
      "due_date": "2024-02-01",
      "next_invoice_date": "2024-03-01",
      "product": {
        "id": 67890,
        "name": "Premium Hosting"
      }
    }
  ],
  "pagination": {...}
}
```

### Create Recurring Invoice

```http
POST /billing/invoices
```

**Request Body:**

```json
{
  "user_id": 12345,
  "items": [
    {
      "description": "Premium Hosting - Monthly",
      "quantity": 1,
      "unit_price": 19.99,
      "taxed": true
    }
  ],
  "billing_cycle": "monthly",
  "auto_payment": true,
  "send_invoice": true
}
```

### Update Recurring Amount

```http
PATCH /billing/invoices/{invoice_id}/recurring
```

**Request Body:**

```json
{
  "amount": 24.99,
  "reason": "Price increase"
}
```

### Cancel Recurring Invoice

```http
POST /billing/invoices/{invoice_id}/recurring/cancel
```

**Request Body:**

```json
{
  "reason": "Customer requested cancellation",
  "immediate_effect" => true,
  "prorate_refund" => false
}
```

## Webhooks

### Recurring Invoice Webhooks

| Event | Description |
|-------|-------------|
| `InvoiceCreated` | New recurring invoice generated |
| `InvoicePaid` | Invoice payment received |
| `InvoiceOverdue` | Invoice past due date |
| `RecurringCancelled` | Recurring billing cancelled |
| `RecurringUpdated` | Recurring amount changed |

### Webhook Payload

```json
{
  "event": "InvoiceCreated",
  "timestamp": "2024-01-15T00:00:00Z",
  "data": {
    "invoice_id": "INV-12345",
    "user_id": 12345,
    "amount": 19.99,
    "billing_cycle": "monthly",
    "due_date": "2024-02-01",
    "next_invoice_date": "2024-03-01"
  }
}
```

## Customer Management

### View Recurring Invoices

**Client Area > Billing > Recurring Invoices**

```
+------------------------------------------------------------------+
|  Recurring Invoices                                              |
+------------------------------------------------------------------+
|                                                                  |
|  +--------------------------------------------------------------+|
|  | Product            | Amount    | Next Date  | Status         ||
|  |--------------------|-----------|------------|----------------||
|  | Premium Hosting    | $21.78/mo | Feb 1, 2024| Active         ||
|  | SSL Certificate    | $79.99/yr | Mar 15, 2024| Active         ||
|  | Domain Renewal     | $12.99/yr | Apr 1, 2024 | Active         ||
|  +--------------------------------------------------------------+|
|                                                                  |
|  Total Monthly: $21.78                                            |
|  Total Annually: $92.98                                          |
|                                                                  |
|  [Update Payment Method] [View All Invoices]                      |
+------------------------------------------------------------------+
```

### Update Recurring Amount

**Client Area > My Services > Manage > Billing Settings**

```
+------------------------------------------+
| Recurring Billing                         |
+------------------------------------------+
| Current Plan: Premium Hosting            |
| Billing Cycle: Monthly                   |
| Amount: $19.99 + $1.79 tax = $21.78     |
| Next Invoice: February 1, 2024          |
|                                          |
| New Amount: [24.99____________]         |
| Effective: [x] Immediately               |
|         [ ] Next billing cycle           |
|                                          |
| Reason: [Price increase notification____]|
|                                          |
| [Confirm Update]                         |
+------------------------------------------+
```

## Prorated Billing

### Prorate on Upgrade

```php
// Prorated billing for upgrades
$prorateConfig = [
    'enabled' => true,
    'method' => 'daily',          // daily, exact
    'credit_existing' => true,
    'charge_new' => true,
    'include_tax' => true
];

// Example calculation
$daysRemaining = 15;
$newPriceMonthly = 29.99;
$oldPriceMonthly = 19.99;

$creditAmount = ($oldPriceMonthly / 30) * $daysRemaining;
$chargeAmount = ($newPriceMonthly / 30) * $daysRemaining;
$netAmount = $chargeAmount - $creditAmount;
```

### Prorate on Downgrade

```php
// Prorated refund for downgrades
$downgradeConfig = [
    'enabled' => true,
    'credit_remaining' => true,
    'apply_to_next_invoice' => true,
    'refund_method' => 'credit'     // credit, refund, both
];
```

## Reporting

### Recurring Revenue Report

```http
GET /billing/reports/recurring-revenue
```

**Response:**

```json
{
  "period": "2024-01",
  "summary": {
    "total_recurring_mrr": 45230.00,
    "total_recurring_arr": 542760.00,
    "new_mrr" => 2150.00,
    "churned_mrr" => 890.00,
    "net_new_mrr" => 1260.00
  },
  "by_cycle": {
    "monthly": 28500.00,
    "quarterly": 8500.00,
    "annually": 8230.00
  },
  "by_product": [
    {"product" => "Premium Hosting", "mrr" => 12500.00},
    {"product" => "Basic Hosting", "mrr" => 8900.00}
  ]
}
```

## Troubleshooting

### Common Issues

| Issue | Cause | Solution |
|-------|-------|----------|
| Invoice not generated | Cron not running | Check cron configuration |
| Wrong amount | Price not updated | Verify product pricing |
| Auto-payment failed | Invalid payment method | Update customer payment |
| Missing line item | Product not recurring | Check product settings |

### Debug Commands

```bash
# Check pending invoices
whmcscli invoice list --status=pending --recurring=true

# Force generate invoice
whmcscli invoice generate --user_id=12345 --product_id=67890

# View recurring settings
whmcscli invoice recurring-info --invoice_id=INV-12345
```

## Best Practices

1. **Send reminders** before due date
2. **Enable auto-payment** for reliable revenue
3. **Use prorated billing** for mid-cycle changes
4. **Monitor churn** through recurring reports
5. **Communicate price changes** before they take effect
6. **Offer annual discounts** to improve cash flow

## See Also

- [Billing Cycles](./whmcs-billing-cycles.md)
- [Payment Terms](./whmcs-payment-terms.md)
- [Invoice Codes](./whmcs-invoice-codes.md)
