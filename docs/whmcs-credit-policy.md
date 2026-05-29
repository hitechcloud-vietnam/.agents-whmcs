# WHMCS Credit Policy Documentation

## Overview

Credit policies manage store credit, credit limits, and credit-related operations for customer accounts.

## Configuration

### Enable Credit System

Navigate to: **Configuration > General Settings > Credit Settings**

```php
// Credit Configuration
$creditConfig = [
    'enabled' => true,

    // Credit Limits
    'enable_credit_limit' => true,
    'default_credit_limit' => 0,
    'max_credit_limit' => 10000,

    // Credit Operations
    'allow_negative_balance' => false,
    'allow_credit_purchase' => true,
    'allow_credit_withdrawal' => true,
    'auto_apply_credit' => false,

    // Expiration
    'credit_expires' => true,
    'credit_expiry_days' => 365,
    'notify_before_expiry_days' => [30, 7],
    'expire_credit_on_account_close' => false
];
```

## Credit Balance

### View Credit Balance

```http
GET /clients/{client_id}/credit
```

**Response:**

```json
{
  "client_id": 12345,
  "balance": 150.00,
  "credit_limit": 500.00,
  "available_credit" => 650.00,
  "used_credit" => 0,
  "pending_credit" => 0,
  "currency" => "USD",
  "credit_history" => [
    {
      "id": "CR-001",
      "type" => "credit",
      "amount" => 100.00,
      "balance_after" => 100.00,
      "created_at" => "2024-01-01T10:00:00Z",
      "description" => "Refund for Invoice INV-12345"
    },
    {
      "id": => "CR-002",
      "type" => "debit",
      "amount" => -50.00,
      "balance_after" => 50.00,
      "created_at" => "2024-01-05T10:00:00Z",
      "description" => "Applied to Invoice INV-12346"
    }
  ]
}
```

## Credit Operations

### Add Credit

```http
POST /clients/{client_id}/credit
```

**Request Body:**

```json
{
  "amount": 100.00,
  "type" => "manual",
  "description" => "Account credit for overpayment",
  "expires_at" => "2025-01-15",
  "notify_customer" => true
}
```

### Deduct Credit

```http
POST /clients/{client_id}/credit/deduct
```

**Request Body:**

```json
{
  "amount": 50.00,
  "description" => "Credit applied to invoice",
  "reference_invoice" => "INV-12345",
  "notify_customer" => true
}
```

### Transfer Credit

```http
POST /clients/{client_id}/credit/transfer
```

**Request Body:**

```json
{
  "to_client_id": 67890,
  "amount": 50.00,
  "description" => "Transfer to account for refund"
}
```

## Credit Limits

### Set Credit Limit

```php
// Credit limit per customer
$clientCreditLimit = [
    'credit_limit' => 500.00,
    'limit_type' => 'fixed',    // fixed, variable
    'approval_required_for' => 1000.00,
    'review_date' => '2024-06-01'
];
```

### Credit Limit Tiers

```php
// Credit limit by customer tier
$creditLimitTiers = [
    'standard' => [
        'default_limit' => 0,
        'max_limit' => 100,
        'require_approval' => false
    ],
    'preferred' => [
        'default_limit' => 500,
        'max_limit' => 1000,
        'require_approval' => true,
        'approval_threshold' => 500
    ],
    'enterprise' => [
        'default_limit' => 5000,
        'max_limit' => 10000,
        'require_approval' => true,
        'approval_threshold' => 2000
    ]
];
```

## Automatic Credit Application

### Auto-Apply Configuration

```php
// Auto-apply credit settings
$autoApplyConfig = [
    'enabled' => true,
    'apply_order' => 'oldest_first',    // oldest_first, largest_first

    // When to apply
    'apply_on_invoice_create' => true,
    'apply_on_invoice_view' => false,
    'apply_on_due_date' => true,
    'apply_before_suspension' => true,

    // Amount limits
    'minimum_apply_amount' => 1.00,
    'maximum_apply_percentage' => 100,   // Apply up to 100% of invoice
    'leave_minimum_balance' => 0,

    // Notifications
    'notify_before_apply' => true,
    'notify_after_apply' => true
];
```

### Customer Preference

```php
// Per-customer auto-apply settings
$customerAutoApply = [
    'auto_apply_enabled' => true,
    'apply_percentage' => 100,           // Apply up to 100%
    'minimum_invoice_amount' => 10.00,
    'exclude_first_invoice' => false
];
```

## Credit Expiration

### Expiration Settings

```php
// Credit expiration configuration
$creditExpiration = [
    'enabled' => true,
    'expiry_days' => 365,
    'expiry_notification_days' => [30, 7, 1],
    'auto_expire' => true,

    // Exceptions
    'never_expire_for' => [
        'preferred_customers',
        'positive_balance_customers'
    ],

    // On expiry
    'notify_before_expiry' => true,
    'notify_on_expiry' => true,
    'convert_to_accounting' => true
];
```

### Credit Expiry Calculation

```php
// Calculate credit expiration
$expiryCalculation = [
    'credit_added' => '2024-01-15',
    'expiry_days' => 365,
    'expires_at' => '2025-01-15',
    'notify_days' => [30, 7, 1],
    'notification_dates' => [
        '2024-12-16',    // 30 days
        '2025-01-08',    // 7 days
        '2025-01-14'     // 1 day
    ]
];
```

## Credit Purchase

### Purchase Credit

```php
// Credit purchase settings
$creditPurchase = [
    'enabled' => true,
    'minimum_purchase' => 10.00,
    'maximum_purchase' => 1000.00,
    'payment_methods' => ['credit_card', 'paypal'],

    // Pricing
    'credit_conversion_rate' => 1.0,     // 1:1 ratio
    'bonus_credit' => [
        'enabled' => true,
        'thresholds' => [
            ['amount' => 100, 'bonus' => 5],    // 5% bonus
            ['amount' => 500, 'bonus' => 10],   // 10% bonus
            ['amount' => 1000, 'bonus' => 15]   // 15% bonus
        ]
    ],

    // Expiration
    'purchased_credit_expiry_days' => 730   // 2 years
];
```

### Purchase API

```http
POST /clients/{client_id}/credit/purchase
```

**Request Body:**

```json
{
  "amount": 500.00,
  "payment_method_id": "PM-12345",
  "add_bonus_credit" => true,
  "notify_customer" => true
}
```

## Credit History

### Get Credit History

```http
GET /clients/{client_id}/credit/history
```

**Query Parameters:**
- `type`: all, credit, debit
- `from_date`: Start date
- `to_date`: End date
- `limit`: Results limit

**Response:**

```json
{
  "client_id": 12345,
  "balance" => 150.00,
  "history" => [
    {
      "id": "CR-001",
      "type" => "credit",
      "amount" => 100.00,
      "balance_before" => 0,
      "balance_after" => 100.00,
      "description" => "Refund",
      "reference_type" => "refund",
      "reference_id" => "REF-12345",
      "expires_at" => "2025-01-15",
      "created_at" => "2024-01-01T10:00:00Z"
    }
  ],
  "pagination": {...}
}
```

## Credit Policies

### Policy Types

```php
// Credit policies
$policies = [
    'refund_to_credit' => [
        'enabled' => true,
        'default' => true,
        'customer_optional' => true
    ],
    'overpayment_to_credit' => [
        'enabled' => true,
        'default' => true
    ],
    'service_credit' => [
        'enabled' => true,
        'approval_required' => true,
        'max_single_credit' => 1000.00
    ],
    'promotional_credit' => [
        'enabled' => true,
        'expiry_days' => 90,
        'requires_approval' => false
    ],
    'goodwill_credit' => [
        'enabled' => true,
        'requires_approval' => true,
        'approval_threshold' => 500.00
    ]
];
```

## Customer Portal

### View Credit Balance

**Client Area > Account > Credit Balance**

```
+------------------------------------------------------------------+
|  Credit Balance                                                  |
+------------------------------------------------------------------+
|                                                                  |
|  Available Credit: $150.00                                      |
|  Credit Limit: $500.00                                         |
|  Total Available: $650.00                                       |
|                                                                  |
|  +--------------------------------------------------------------+|
|  | Credit History                                              ||
|  |--------------------------------------------------------------||
|  | Date       | Description              | Amount    | Balance  ||
|  |------------|--------------------------|-----------|----------||
|  | Jan 15    | Refund                   | +$100.00  | $100.00  ||
|  | Jan 10    | Applied to Invoice       | -$50.00   | $50.00   ||
|  | Jan 5     | Promotional Credit       | +$100.00  | $100.00  ||
|  +--------------------------------------------------------------+|
|                                                                  |
|  [Purchase Credit] [Auto-Apply Settings] [View All History]     |
+------------------------------------------------------------------+
```

### Purchase Credit

**Client Area > Account > Purchase Credit**

```
+------------------------------------------+
|  Purchase Credit                         |
+------------------------------------------+
|  Amount:  [100___________]             |
|                                          |
|  Bonus Credit Available:                  |
|  $100 - $499: 5% bonus ($5.00)         |
|  $500 - $999: 10% bonus ($50.00)      |
|  $1000+: 15% bonus ($150.00)          |
|                                          |
|  Payment Method:                         |
|  [Credit Card ending 4242____]         |
|                                          |
|  Total Credit: $110.00 (with 10% bonus)|
|                                          |
|  [Purchase Credit]                       |
+------------------------------------------+
```

## Reporting

### Credit Report

```http
GET /billing/reports/credit
```

**Response:**

```json
{
  "period": "2024-01",
  "summary": {
    "total_credit_issued" => 10000.00,
    "total_credit_used" => 8500.00,
    "current_outstanding" => 1500.00,
    "credit_expired" => 500.00,
    "credit_forfeited" => 100.00
  },
  "by_type" => [
    {"type" => "refund", "amount" => 5000.00},
    {"type" => "service_credit", "amount" => 3000.00},
    {"type" => "promotional", "amount" => 2000.00}
  ],
  "by_customer" => [
    {"client_id" => 12345, "balance" => 500.00},
    {"client_id" => 67890, "balance" => 300.00}
  ]
}
```

## API Reference

### Credit Transaction Types

| Type | Description |
|------|-------------|
| `credit` | Credit added |
| `debit` | Credit deducted |
| `refund` | Refund to credit |
| `transfer_in` | Transfer from another account |
| `transfer_out` | Transfer to another account |
| `expired` | Credit expired |
| `forfeited` | Credit forfeited |

## Troubleshooting

### Common Issues

| Issue | Cause | Solution |
|-------|-------|----------|
| Credit not applying | Auto-apply disabled | Enable in settings |
| Credit expired | Expiry policy | Check expiry settings |
| Negative balance | Allow negative disabled | Check credit limit |
| Purchase failed | Gateway issue | Check payment settings |

### Debug Commands

```bash
# View client credit
whmcscli client credit --client_id=12345

# Add credit
whmcscli credit add --client_id=12345 --amount=100 --description="Manual credit"

# Check credit history
whmcscli credit history --client_id=12345
```

## Best Practices

1. **Set clear policies** - Document credit terms
2. **Monitor expiration** - Notify before credit expires
3. **Track usage** - Monitor credit patterns
4. **Automate fairly** - Apply credits appropriately
5. **Maintain records** - Keep detailed audit trail
6. **Review limits** - Adjust based on history

## See Also

- [Refund Policy](./whmcs-refund-policy.md)
- [Loyalty Program](./whmcs-loyalty-program.md)
- [Reward Points](./whmcs-reward-points.md)
