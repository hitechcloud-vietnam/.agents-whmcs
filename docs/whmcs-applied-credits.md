# WHMCS Applied Credits

## Overview

Applied credits are amounts taken from a client's credit balance and applied to an invoice or order. WHMCS provides flexible credit application mechanisms with automatic and manual options.

## Credit Application Overview

### Types of Credit Application

| Type | Description | Trigger |
|------|-------------|---------|
| Manual | Admin-applied credit | Admin action |
| Automatic | System-applied credit | Invoice generation/payment |
| Pre-payment | Advance payment applied | Client initiates |
| Refund Credit | Credit from refund | System or admin |

## Automatic Credit Application

### Configuration

**Configuration > Billing > Credit Settings**

```php
[
    'auto_credit_enabled' => true,
    'auto_credit_threshold' => 0.01,     // Minimum to apply
    'auto_credit_order' => 'oldest_first', // oldest, newest, highest
    'credit_invoice_auto' => true,
    'credit_payment_auto' => true
]
```

### Application Order

```php
// Order credit is applied
[
    'oldest_first' => [
        // Apply oldest credit first
        // Credit created Jan 1
        // Credit created Jan 15
        // First: Jan 1, Then: Jan 15
    ],
    'newest_first' => [
        // Apply newest credit first
        // Newest first
    ],
    'highest_first' => [
        // Apply largest credit first
        // $50, then $30, then $20
    ],
    'expiry_first' => [
        // Apply expiring credits first
        // Expires sooner first
    ]
]
```

## Manual Credit Application

### Apply Credit to Invoice

**Admin: Billing > Invoices > Apply Credit**

```php
// Manual credit application
[
    'userid' => 123,
    'invoiceid' => 5678,
    'amount' => 25.00,
    'credit_id' => null,           // null = use oldest available
    'description' => 'Customer request',
    'applied_by' => 'admin_id',
    'applied_at' => '2024-05-15'
]
```

### Apply to Specific Credit

```php
// Apply specific credit balance
[
    'credit_id' => 789,            // Specific credit entry
    'invoiceid' => 5678,
    'amount' => 25.00,
    'description' => 'Applying promotional credit'
]
```

### Partial Application

```php
// Apply portion of credit
[
    'credit_id' => 789,
    'credit_balance' => 100.00,
    'apply_amount' => 25.00,
    'remaining_credit' => 75.00
]
```

## Credit Application Rules

### Amount Limits

```php
// Control credit application amounts
[
    'max_credit_per_invoice' => 500.00,
    'minimum_credit_apply' => 0.01,
    'require_credit_threshold' => 1.00,   // Min credit balance to use
    'round_to' => 0.01                    // Round to cents
]
```

### Client Restrictions

```php
// Exclude certain clients
[
    'exclude_client_groups' => ['prepaid'],
    'exclude_client_ids' => [123],
    'require_credit_age_days' => 1        // Credit must be X days old
]
```

### Invoice Restrictions

```php
// Control which invoices get credit
[
    'apply_to_status' => ['Overdue', 'Pending'],
    'exclude_invoice_types' => ['proforma', 'quote'],
    'exclude_amount_min' => 1.00,
    'exclude_amount_max' => 10000.00
]
```

## Credit Application Process

### Invoice Generation Credit

```php
// Apply credit when invoice created
[
    'invoice_id' => 5678,
    'client_id' => 123,
    'invoice_total' => 100.00,
    'credit_available' => 50.00,
    'auto_applied' => true,
    'amount_applied' => 50.00,
    'remaining_invoice_due' => 50.00
]
```

### Payment Credit Application

```php
// Apply credit during payment
[
    'invoice_id' => 5678,
    'payment_amount' => 50.00,
    'credit_applied' => 25.00,
    'actual_payment' => 25.00,
    'payment_method' => 'credit_only'
]
```

## Credit Display

### Invoice Line Item

```
+--------------------------------------------------+
| Credits Applied                                  |
+--------------------------------------------------+
| Credit #1234 | -$25.00 | Applied: May 15, 2024  |
+--------------------------------------------------+
| Total Credits:              -$25.00               |
+--------------------------------------------------+
| Balance Due:                $75.00               |
+--------------------------------------------------+
```

### Client Credit History

```php
// Credit application history
[
    ['date' => '2024-05-15', 'type' => 'applied', 'amount' => -25.00, 
     'invoice' => '5678', 'balance' => 25.00],
    ['date' => '2024-05-01', 'type' => 'add', 'amount' => 50.00,
     'description' => 'Account credit', 'balance' => 50.00]
]
```

## Credit Reversal

### Reversing Credit Application

```php
// Reverse credit application
[
    'credit_application_id' => 456,
    'invoice_id' => 5678,
    'credit_id' => 789,
    'amount' => 25.00,
    'reversed_by' => 'admin_id',
    'reversal_reason' => 'Invoice cancelled'
]
```

### After Reversal

```php
// Credit restored to balance
[
    'credit_id' => 789,
    'original_balance' => 75.00,
    'credit_restored' => 25.00,
    'new_balance' => 100.00,
    'restored_at' => '2024-05-16'
]
```

## Multi-Credit Application

### Applying Multiple Credits

```php
// Apply multiple credits to one invoice
[
    'invoice_id' => 5678,
    'credits' => [
        ['credit_id' => 123, 'amount' => 25.00],
        ['credit_id' => 456, 'amount' => 15.00]
    ],
    'total_credit' => 40.00,
    'invoice_total' => 100.00,
    'remaining_due' => 60.00
]
```

### Credit Priority

```php
// Order multiple credits applied
[
    'apply_order' => ['promotional_first', 'oldest_first'],
    'prefer_smaller' => false,         // Apply smaller credits first
    'preserve_expiring' => true        // Don't use expiring credits
]
```

## Credit Application Emails

### Notification Template

```smarty
Subject: Credit Applied to Invoice {$invoice_number}

Dear {$client_first_name},

A credit of {$credit_amount} has been applied to invoice {$invoice_number}.

Invoice Original: {$invoice_total}
Credit Applied:   -{$credit_amount}
Amount Due:       {$amount_due}

You can view your updated invoice here:
{$invoice_url}

{$company_name}
```

## API Functions

```php
// Apply credit
$params = [
    'invoiceid' => 5678,
    'amount' => 25.00,
    'creditid' => null
];
$result = localAPI('ApplyCredit', $params);

// Get credit balance
$params = [
    'clientid' => 123
];
$result = localAPI('GetCreditBalance', $params);

// Get credit history
$params = [
    'clientid' => 123
];
$result = localAPI('GetClientCredits', $params);

// Reverse credit
$params = [
    'invoiceid' => 5678,
    'creditid' => 789,
    'amount' => 25.00
];
$result = localAPI('ReverseCredit', $params);
```

## Hooks

```php
// Hook: PreCreditApplication
add_hook('PreCreditApplication', 1, function($vars) {
    // $vars['invoiceid']
    // $vars['creditid']
    // $vars['amount']
    
    // Validate or modify
    return ['allow' => true, 'amount' => $vars['amount']];
});

// Hook: CreditApplied
add_hook('CreditApplied', 1, function($vars) {
    // $vars['invoiceid']
    // $vars['creditid']
    // $vars['amount']
    // $vars['balance_remaining']
});
```

## Best Practices

1. **Set clear rules**: Define when credit auto-applies
2. **Communicate to clients**: Show credit balance clearly
3. **Track application history**: Maintain complete audit trail
4. **Manage limits**: Prevent over-application
5. **Regular reconciliation**: Verify credit balances

## Related Documentation

- [Credit System](./whmcs-credit-system.md)
- [Credit Notes](./whmcs-credit-notes.md)
- [Invoice Generation](./whmcs-invoice-generation.md)
- [Payment Allocation](./whmcs-payment-allocation.md)