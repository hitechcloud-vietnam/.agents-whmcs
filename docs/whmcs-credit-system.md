# WHMCS Credit System

## Overview

The WHMCS credit system allows clients to maintain a credit balance that can be applied toward invoices, refunds, or service payments. Credits can be added manually by admins, automatically through overpayments, or through promotional campaigns.

## Accessing Credit Management

### Admin Area
**Billing > Client Credits**

### Client View
**Clients > Select Client > Summary > Credit Balance**

## Credit Balance Display

```php
// Client credit information
[
    'id' => 1,
    'userid' => 123,
    'description' => 'Account credit',
    'amount' => 50.00,
    'currency' => 'USD',
    'created_at' => '2024-05-01',
    'remaining' => 35.00
]
```

## Adding Credits

### Manual Credit Addition

**Admin: Clients > Credit > Add Credit**

```php
// Add credit form
[
    'userid' => 123,
    'amount' => 50.00,
    'description' => 'Account credit - promotional',
    'type' => 'add',
    'admin_id' => 1,
    'transaction_id' => null
]
```

### Automatic Credit Creation

Credits are automatically created when:
- Overpayment occurs on invoice
- Refund processed
- Credit note issued
- Promotional credit applied

```php
// Auto-credit on overpayment
[
    'userid' => 123,
    'amount' => 15.00,
    'description' => 'Overpayment on invoice #1234',
    'type' => 'overpayment',
    'transaction_id' => 5678
]
```

## Credit Application

### Automatic Application

**Configuration > Billing > Credit Settings**

```php
[
    'auto_credit_invoice' => true,
    'credit_order' => 'oldest_first',  // or 'newest_first'
    'credit_threshold' => 0.01          // Minimum credit to apply
]
```

### Manual Application

**Admin: Invoice > Apply Credit**

```php
// Apply credit to invoice
[
    'userid' => 123,
    'invoiceid' => 5678,
    'amount' => 25.00,
    'description' => 'Manual credit application'
]
```

### Credit Application Order

```php
// By default, oldest credit applied first
// Invoice #100: $50 due
// Credit 1: $20 (created Jan 1)
// Credit 2: $30 (created Jan 15)

// Applied: Credit 1 fully, then Credit 2 partially
// Result: $20 from Credit 1, $30 from Credit 2
// Invoice paid in full
```

## Credit vs Prepayment

### Credit (Positive Balance)
- Money owed to client
- Applied to future invoices
- Can be refunded

### Prepayment (Negative Balance)
- Advance payment made
- Reserved for services
- Applied to specific invoices

## Credit Types

### Standard Credit
```php
[
    'type' => 'credit',
    'description' => 'Account credit',
    'amount' => 100.00,
    'expiry' => null  // Never expires
]
```

### Promotional Credit
```php
[
    'type' => 'promotional',
    'description' => 'Welcome bonus credit',
    'amount' => 10.00,
    'expiry' => '2024-12-31',
    'min_purchase' => 50.00
]
```

### Refund Credit
```php
[
    'type' => 'refund',
    'description' => 'Refund for cancelled service',
    'amount' => 25.00,
    'expiry' => null,
    'transaction_id' => 12345
]
```

## Credit Limits

### Client Credit Limit

```php
// Configure per client
[
    'userid' => 123,
    'credit_limit' => 500.00,
    'credit_used' => 150.00,
    'credit_available' => 350.00
]
```

### Credit Limit Enforcement

```php
// Prevent adding credit beyond limit
if ($currentBalance + $amount > $creditLimit) {
    throw new Exception('Credit limit exceeded');
}
```

## Credit Transactions

### Transaction Log

```php
// Credit transaction history
[
    ['date' => '2024-05-01', 'type' => 'add', 'amount' => 50.00, 'balance' => 50.00],
    ['date' => '2024-05-05', 'type' => 'apply', 'amount' => -25.00, 'balance' => 25.00],
    ['date' => '2024-05-10', 'type' => 'add', 'amount' => 30.00, 'balance' => 55.00]
]
```

### Transaction Entry

```php
// Log credit action
$transaction = [
    'userid' => 123,
    'type' => 'add',           // add, apply, refund, expire
    'amount' => 50.00,
    'balance_before' => 0.00,
    'balance_after' => 50.00,
    'description' => 'Manual credit add',
    'admin_id' => 1,
    'reference' => 'INV-1234',
    'created_at' => date('Y-m-d H:i:s')
];
```

## Credit on Invoice

### Display on Invoice

```
+--------------------------------------------------+
| Credits Applied                                  |
+--------------------------------------------------+
| Credit #1234  | -$25.00 | Invoice #5678         |
+--------------------------------------------------+
| Total Credits Applied:     -$25.00               |
+--------------------------------------------------+
| Balance Due:               $25.00                |
+--------------------------------------------------+
```

### Invoice Payment with Credit

```php
// Payment with credit
$invoiceTotal = 100.00;
$creditBalance = 35.00;

$creditToApply = min($invoiceTotal, $creditBalance);  // $35
$remainingDue = $invoiceTotal - $creditToApply;        // $65

// Invoice paid with $35 credit + $65 payment
```

## Credit Expiry

### Expiring Credits

```php
// Promotional credits with expiry
[
    'amount' => 10.00,
    'expiry_date' => '2024-12-31',
    'days_before_expiry_notify' => 14
]
```

### Expiry Notification

```php
// Email notification before expiry
[
    'subject' => 'Your credit is expiring soon',
    'body' => 'Your credit of $10.00 will expire 
               on December 31, 2024. Use it soon!'
]
```

### Automatic Expiry Processing

```php
// Cron: expires credits daily
// Process on expiry date
// Transfer to revenue or hold based on policy
```

## Credit Reports

### Admin Reports

**Reports > Billing > Credit Report**

```php
// Credit summary
[
    'total_credits' => 5000.00,
    'credits_used' => 3500.00,
    'credits_available' => 1500.00,
    'pending_expiry' => 100.00
]
```

### Client Credit History

**Client Area > Billing > Credit History**

```php
// Client sees their credit activity
[
    ['date' => '2024-05-01', 'description' => 'Credit added', 'amount' => '+50.00'],
    ['date' => '2024-05-05', 'description' => 'Applied to invoice', 'amount' => '-25.00'],
    ['date' => '2024-05-10', 'description' => 'Credit added', 'amount' => '+30.00']
]
```

## API Functions

```php
// Add credit
$params = [
    'userid' => 123,
    'amount' => 50.00,
    'description' => 'Account credit',
    'type' => 'add'
];
$result = localAPI('AddCredit', $params);

// Get credit balance
$params = [
    'clientid' => 123
];
$result = localAPI('GetCreditBalance', $params);

// Apply credit to invoice
$params = [
    'invoiceid' => 5678,
    'amount' => 25.00
];
$result = localAPI('ApplyCredit', $params);
```

## Hooks

```php
// Hook: CreditAdded
add_hook('CreditAdded', 1, function($vars) {
    // $vars['userid']
    // $vars['amount']
    // $vars['creditid']
});

// Hook: CreditApplied
add_hook('CreditApplied', 1, function($vars) {
    // $vars['userid']
    // $vars['invoiceid']
    // $vars['amount']
});
```

## Configuration Settings

**Configuration > Billing > Credit Settings**

| Setting | Description | Default |
|---------|-------------|---------|
| Auto Credit Application | Apply credit to invoices automatically | Yes |
| Credit Order | Order of credit application | Oldest first |
| Minimum Credit | Minimum amount to apply | 0.01 |
| Credit Limit Enforce | Enforce client credit limits | No |
| Allow Manual Credit | Allow admin manual credit adds | Yes |
| Credit Expiry | Enable credit expiry | No |

## Best Practices

1. **Track all credits**: Maintain complete transaction history
2. **Clear descriptions**: Document credit source and reason
3. **Set limits**: Prevent credit abuse
4. **Monitor expiries**: Process expiring credits promptly
5. **Regular reconciliation**: Match credit totals

## Related Documentation

- [Invoice Generation](./whmcs-invoice-generation.md)
- [Payment Allocation](./whmcs-payment-allocation.md)
- [Credit Notes](./whmcs-credit-notes.md)
- [Refund Processing](./whmcs-refund-processing.md)