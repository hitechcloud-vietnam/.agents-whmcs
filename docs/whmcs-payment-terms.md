# WHMCS Payment Terms Configuration Documentation

## Overview

Payment terms define when invoices are due and what payment methods are available. WHMCS supports flexible payment term configurations.

## Configuration

### Enable Payment Terms

Navigate to: **Configuration > General Settings > Invoices**

```php
// Payment Terms Configuration
$paymentTermsConfig = [
    // Basic Settings
    'default_due_days' => 7,
    'min_due_days' => 0,
    'max_due_days' => 90,

    // Due Date Calculation
    'due_date_calculation' => 'invoice_date',  // invoice_date, end_of_month
    'payment_terms_by_product' => true,
    'payment_terms_by_customer' => true,

    // Overdue Handling
    'allow_overdue' => false,
    'overdue_threshold_days' => 0
];
```

### Default Payment Terms

```php
// Default terms
$defaultTerms = [
    'name' => 'Net 7',
    'days' => 7,
    'description' => 'Payment due within 7 days',
    'late_fee_after_days' => 3,
    'suspend_after_days' => 14,
    'terminate_after_days' => 30
];
```

## Payment Term Types

### Standard Terms

```php
// Standard payment terms
$standardTerms = [
    'due_on_receipt' => [
        'name' => 'Due on Receipt',
        'days' => 0,
        'display_name' => 'Due Immediately'
    ],
    'net_7' => [
        'name' => 'Net 7',
        'days' => 7,
        'display_name' => 'Net 7 Days'
    ],
    'net_15' => [
        'name' => 'Net 15',
        'days' => 15,
        'display_name' => 'Net 15 Days'
    ],
    'net_30' => [
        'name' => 'Net 30',
        'days' => 30,
        'display_name' => 'Net 30 Days'
    ],
    'net_45' => [
        'name' => 'Net 45',
        'days' => 45,
        'display_name' => 'Net 45 Days'
    ],
    'net_60' => [
        'name' => 'Net 60',
        'days' => 60,
        'display_name' => 'Net 60 Days'
    ]
];
```

### Custom Terms

```php
// Custom payment terms
$customTerms = [
    'net_90' => [
        'name' => 'Net 90',
        'days' => 90,
        'display_name' => 'Net 90 Days',
        'requires_approval' => true,
        'credit_limit_required' => true,
        'available_for' => ['enterprise', 'preferred']
    ],
    'quarterly' => [
        'name' => 'Quarterly in Advance',
        'days' => 0,
        'billing_aligned' => true,
        'display_name' => 'Quarterly'
    ],
    'custom_15_eom' => [
        'name' => '15th of Following Month',
        'days' => 15,
        'calculation' => 'specific_day',
        'specific_day' => 15,
        'month_calculation' => 'next'
    ]
];
```

## Product Payment Terms

### Configure Product Terms

```php
// Product-specific payment terms
$productTerms = [
    'hosting' => [
        'default_term' => 'net_7',
        'available_terms' => ['due_on_receipt', 'net_7'],
        'require_auto_pay' => false
    ],
    'dedicated_server' => [
        'default_term' => 'net_15',
        'available_terms' => ['net_15', 'net_30'],
        'require_auto_pay' => true,
        'setup_fee_term' => 'due_on_receipt'
    ],
    'enterprise' => [
        'default_term' => 'net_30',
        'available_terms' => ['net_15', 'net_30', 'net_45', 'net_60', 'net_90'],
        'require_auto_pay' => false,
        'require_approval' => true
    ]
];
```

## Customer Payment Terms

### Customer-Specific Terms

```php
// Customer payment term assignment
$customerTerms = [
    'default' => [
        'term' => 'net_7',
        'credit_limit' => 0
    ],
    'preferred' => [
        'term' => 'net_15',
        'credit_limit' => 5000,
        'auto_approve' => true
    ],
    'enterprise' => [
        'term' => 'net_30',
        'credit_limit' => 50000,
        'require_purchase_order' => true,
        'approval_required_for' => ['over_1000']
    ],
    'vip' => [
        'term' => 'net_45',
        'credit_limit' => 100000,
        'auto_approve' => true,
        'payment_method_required' => false
    ]
];
```

### Credit Limits

```php
// Credit limit configuration
$creditLimits = [
    'enabled' => true,
    'check_credit_limit' => true,
    'allow_overdue_until' => 0,
    'limit_by_tier' => [
        'standard' => 0,
        'preferred' => 5000,
        'enterprise' => 50000,
        'vip' => 100000
    ],
    'auto_suspend_over_limit' => true
];
```

## Payment Terms API

### List Payment Terms

```http
GET /billing/payment-terms
```

**Response:**

```json
{
  "terms": [
    {
      "id": "TERM-001",
      "name" => "Net 7",
      "days" => 7,
      "display_name" => "Net 7 Days",
      "available_for" => "all",
      "enabled" => true
    },
    {
      "id": "TERM-002",
      "name" => "Net 30",
      "days" => 30,
      "display_name" => "Net 30 Days",
      "available_for" => ["enterprise", "preferred"],
      "enabled" => true
    }
  ]
}
```

### Create Payment Term

```http
POST /billing/payment-terms
```

**Request Body:**

```json
{
  "name": "Net 30",
  "days": 30,
  "display_name": "Net 30 Days",
  "available_for": ["all"],
  "late_fee_days" => 3,
  "suspend_days" => 14,
  "enabled": true
}
```

### Update Payment Term

```http
PATCH /billing/payment-terms/{term_id}
```

### Assign Term to Customer

```http
PATCH /clients/{client_id}/payment-term
```

**Request Body:**

```json
{
  "term_id": "TERM-002",
  "credit_limit": 10000,
  "effective_date": "2024-01-15"
}
```

## Due Date Calculation

### Invoice Date Basis

```php
// Calculate due date from invoice date
function calculateDueDate($invoiceDate, $days) {
    $dueDate = clone $invoiceDate;
    $dueDate->add(new DateInterval("P{$days}D"));
    return $dueDate;
}

// Example: Invoice dated Jan 1, Net 7 = Due Jan 8
```

### End of Month Calculation

```php
// Calculate due date at end of month
function calculateDueDateEOM($invoiceDate, $days) {
    $endOfMonth = clone $invoiceDate;
    $endOfMonth->modify('last day of this month');

    if ($invoiceDate->format('j') > $days) {
        // Next month's due date
        $endOfMonth->modify('first day of next month');
        $endOfMonth->modify("+{$days} days");
    }

    return $endOfMonth;
}

// Example: Invoice dated Jan 15, EOM + 15 = Due Feb 15
```

### Specific Day Calculation

```php
// Calculate due date on specific day
function calculateDueDateSpecificDay($invoiceDate, $dayOfMonth) {
    $dueDate = clone $invoiceDate;

    // If invoice day is past the due day, move to next month
    if ($invoiceDate->format('j') >= $dayOfMonth) {
        $dueDate->modify('first day of next month');
    } else {
        $dueDate->modify('first day of this month');
    }

    $dueDate->modify("+{$dayOfMonth} days");
    return $dueDate;
}
```

## Invoice Display

### Invoice Terms Display

```html
<!-- Invoice Header -->
<div class="invoice-header">
    <h1>INVOICE</h1>
    <p>Invoice #: INV-20240001</p>
    <p>Date: January 15, 2024</p>
    <p>Terms: <strong>Net 30</strong></p>
    <p>Due Date: <strong>February 14, 2024</strong></p>
</div>
```

### Terms on Invoice

```php
// Invoice terms configuration
$invoiceTermsDisplay = [
    'show_terms' => true,
    'show_due_date' => true,
    'show_payment_methods' => true,
    'terms_template' => [
        'due_on_receipt' => 'Payment due immediately',
        'net_7' => 'Payment due within 7 days',
        'net_30' => 'Payment due within 30 days'
    ]
];
```

## Payment Term Workflow

### Standard Flow

```
Invoice Created
    |
    v
Calculate Due Date (Invoice Date + Terms)
    |
    v
Send Invoice Notification
    |
    v
[D] - Reminder 1 (3 days before)
    |
    v
[D] - Reminder 2 (1 day before)
    |
    v
[D] - Due Date Reached
    |
    v
[0] - Send Overdue Notice
    |
    v
[3] - Apply Late Fee (if configured)
    |
    v
[7] - Suspend Service (if configured)
    |
    v
[14] - Send Termination Warning
    |
    v
[30] - Terminate Service (if configured)
```

## Credit Management

### Credit Check

```php
// Check customer credit before invoicing
function checkCustomerCredit($clientId, $invoiceAmount) {
    $client = WHMCS\User\Client::find($clientId);
    $creditLimit = $client->credit_limit;
    $availableCredit = $creditLimit - $client->outstanding_balance;

    if ($invoiceAmount > $availableCredit) {
        return [
            'approved' => false,
            'reason' => 'Credit limit exceeded',
            'available_credit' => $availableCredit,
            'required_credit' => $invoiceAmount
        ];
    }

    return [
        'approved' => true,
        'available_credit' => $availableCredit,
        'remaining_after' => $availableCredit - $invoiceAmount
    ];
}
```

### Overdue Invoice Handling

```php
// Handle overdue invoices
$overdueHandling = [
    'check_credit_on_overdue' => true,
    'allow_invoicing_over_limit' => false,
    'require_payment_for_new_order' => true,
    'suspend_existing_on_overdue' => true
];
```

## Webhooks

### Payment Terms Webhooks

| Event | Description |
|-------|-------------|
| `PaymentTermCreated` | New term added |
| `PaymentTermUpdated` | Term modified |
| `PaymentTermAssigned` | Term assigned to customer |
| `CreditLimitExceeded` | Customer over credit limit |

### Webhook Payload

```json
{
  "event": "PaymentTermAssigned",
  "timestamp": "2024-01-15T10:30:00Z",
  "data": {
    "client_id": 12345,
    "term_id": "TERM-002",
    "term_name": "Net 30",
    "credit_limit": 10000,
    "assigned_by": "admin"
  }
}
```

## Reporting

### Payment Terms Report

```http
GET /billing/reports/payment-terms
```

**Response:**

```json
{
  "period": "2024-01",
  "by_term": [
    {
      "term" => "Net 7",
      "invoices" => 500,
      "total_amount" => 25000.00,
      "avg_days_to_pay" => 5
    },
    {
      "term" => "Net 30",
      "invoices" => 200,
      "total_amount" => 50000.00,
      "avg_days_to_pay" => 28
    }
  ],
  "summary": {
    "total_invoices" => 700,
    "total_amount" => 75000.00,
    "on_time_payments" => 85,
    "overdue_payments" => 15
  }
}
```

## Troubleshooting

### Common Issues

| Issue | Cause | Solution |
|-------|-------|----------|
| Wrong due date | Terms not applied | Check customer/product terms |
| Credit exceeded | Limit too low | Adjust credit limit |
| Invoice on hold | Overdue invoices | Clear overdue first |
| Terms not available | Not assigned to customer | Update customer terms |

### Debug Commands

```bash
# Check customer payment terms
whmcscli client payment-terms --client_id=12345

# Calculate due date
whmcscli billing due-date --invoice_date=2024-01-01 --terms=net_30

# View credit status
whmcscli client credit --client_id=12345

# Update customer terms
whmcscli billing update-terms --client_id=12345 --term=net_30
```

## Best Practices

1. **Match terms to customer type** - Standard vs enterprise
2. **Set appropriate credit limits** - Based on payment history
3. **Automate reminders** - Reduce late payments
4. **Use auto-pay for short terms** - Ensure payment
5. **Monitor overdue rates** - Adjust terms accordingly
6. **Document special arrangements** - Keep records

## See Also

- [Recurring Invoice](./whmcs-recurring-invoice.md)
- [Late Fee Rules](./whmcs-late-fee-rules.md)
- [Credit Policy](./whmcs-credit-policy.md)
