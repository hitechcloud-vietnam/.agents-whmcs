# WHMCS Credit Notes

## Overview

Credit notes are accounting documents that reduce the amount owed by a customer. In WHMCS, credit notes are used to record refunds, discounts, billing corrections, and other credits that reduce client debt.

## Credit Note Overview

### When to Use Credit Notes

| Scenario | Description |
|----------|-------------|
| Refunds | Return payment to customer |
| Billing Corrections | Fix overcharging error |
| Discounts | Apply promotional discounts |
| Service Adjustments | Credit for service issues |
| Canceled Orders | Credit canceled order amounts |
| Promotional Credits | Add promotional value |

## Credit Note Configuration

### Settings

**Configuration > Billing > Credit Note Settings**

```php
[
    'credit_notes_enabled' => true,
    'auto_create' => true,                 // Auto-create on refund
    'require_approval' => true,
    'approval_threshold' => 100.00,
    'number_format' => 'CN-{YYYY}-{SEQ}',
    'default_expiry_days' => 365
]
```

### Number Format

```php
// Configure credit note numbering
[
    'prefix' => 'CN',
    'year' => true,
    'month' => false,
    'sequence' => true,
    'reset_frequency' => 'yearly',
    'starting_number' => 1
]
```

## Creating Credit Notes

### Manual Creation

**Admin: Billing > Credit Notes > Create Credit Note**

```php
// Credit note creation
[
    'client_id' => 123,
    'number' => 'CN-2024-0123',
    'date' => '2024-05-15',
    'items' => [
        ['description' => 'Billing correction', 'amount' => -25.00],
        ['description' => 'Service credit', 'amount' => -10.00]
    ],
    'subtotal' => -35.00,
    'tax' => -7.00,
    'total' => -42.00,
    'status' => 'Pending'
]
```

### Line Items

```php
// Add items to credit note
[
    'description' => 'Service outage compensation',
    'quantity' => 1,
    'unit_price' => -50.00,
    'amount' => -50.00,
    'taxable' => false
]
```

## Credit Note Status

### Status Flow

```
Draft -> Pending -> Redeemed -> Expired
                     -> Refunded
```

| Status | Description |
|--------|-------------|
| Draft | Not finalized, can be edited |
| Pending | Issued, available for use |
| Redeemed | Applied to invoice/payment |
| Expired | Credit validity period ended |
| Refunded | Paid out in cash |

### Status Transitions

```php
// Draft to Pending
[
    'action' => 'finalize',
    'from_status' => 'Draft',
    'to_status' => 'Pending',
    'notify_client' => true
]

// Pending to Redeemed
[
    'action' => 'redeem',
    'from_status' => 'Pending',
    'to_status' => 'Redeemed',
    'applied_to' => 'invoice_5678'
]
```

## Credit Note vs Refund

### Credit Note Benefits

| Credit Note | Refund |
|-------------|--------|
| Keeps funds in system | Returns cash to customer |
| Can be partially used | Full amount returned |
| Less processing time | Requires payment processing |
| Customer can choose | Pre-determined |
| Encourages repeat business | No further engagement |

### When to Use Each

```php
// Credit note: Customer wants credit
[
    'type' => 'credit_note',
    'reason' => 'Customer prefers credit',
    'add_to_balance' => true
]

// Refund: Customer wants cash back
[
    'type' => 'refund',
    'reason' => 'Customer requests refund',
    'return_to_payment' => true
]
```

## Applying Credit Notes

### Apply to Invoice

```php
// Apply credit note to invoice
[
    'credit_note_id' => 789,
    'invoice_id' => 5678,
    'amount' => 35.00,
    'applied_at' => '2024-05-15',
    'applied_by' => 'admin_id'
]
```

### Partial Application

```php
// Use portion of credit note
[
    'credit_note_id' => 789,
    'total' => 100.00,
    'applied_amount' => 35.00,
    'remaining' => 65.00,
    'remaining_status' => 'Pending'
]
```

### Auto-Apply to Next Invoice

```php
// Automatically apply to next invoice
[
    'credit_note_id' => 789,
    'auto_apply' => true,
    'apply_to_invoice_status' => 'Pending',
    'minimum_balance' => 1.00
]
```

## Credit Note Display

### Invoice Integration

```
Credit Note CN-2024-0123 Applied

Original Invoice Total: $100.00
Credit Note Applied:    -$35.00
---------------------------------
New Balance Due: $65.00
```

### Credit Note Document

```php
// Standard credit note format
[
    'header' => [
        'number' => 'CN-2024-0123',
        'date' => 'May 15, 2024',
        'original_invoice' => 'INV-5678'
    ],
    'client' => [...],
    'items' => [...],
    'totals' => [
        'subtotal' => -35.00,
        'tax' => -7.00,
        'total' => -42.00
    ],
    'footer' => 'Valid for 12 months'
]
```

## Refund vs Redeem Credit Note

### Redeem (Apply to Invoice)

```php
// Apply as credit to invoice
[
    'credit_note_id' => 789,
    'action' => 'redeem',
    'apply_to_invoice' => 5678,
    'amount' => 42.00
]
```

### Refund (Pay Out)

```php
// Pay out credit note
[
    'credit_note_id' => 789,
    'action' => 'refund',
    'refund_method' => 'original_payment',
    'amount' => 42.00,
    'process_fee' => false
]
```

## Credit Note Expiry

### Expiry Configuration

```php
// Set credit validity period
[
    'expiry_enabled' => true,
    'expiry_days' => 365,
    'notify_before_expiry_days' => 30,
    'expired_action' => 'refund',          // refund, expire, notify
    'allow_extension' => true
]
```

### Expiry Notification

```smarty
Subject: Credit Note Expiring Soon - {$credit_note_number}

Dear {$client_first_name},

Your credit note {$credit_note_number} for {$credit_amount} will expire 
on {$expiry_date}.

Please use this credit on your next invoice.

{$company_name}
```

## Credit Note Reporting

### Credit Note Report

**Reports > Billing > Credit Note Report**

```php
// Report structure
[
    'period' => 'May 2024',
    'total_credit_notes' => 30,
    'total_value' => 3000.00,
    'redeemed' => 2000.00,
    'pending' => 800.00,
    'expired' => 200.00,
    'by_reason' => [
        'refund' => 1500.00,
        'billing_correction' => 800.00,
        'service_credit' => 500.00,
        'promotion' => 200.00
    ]
]
```

## API Functions

```php
// Create credit note
$params = [
    'clientid' => 123,
    'amount' => 42.00,
    'reason' => 'Billing correction',
    'related_invoice' => 5678
];
$result = localAPI('CreateCreditNote', $params);

// Get credit notes
$params = [
    'clientid' => 123
];
$result = localAPI('GetCreditNotes', $params);

// Redeem credit note
$params = [
    'creditnoteid' => 789,
    'invoiceid' => 5679,
    'amount' => 35.00
];
$result = localAPI('RedeemCreditNote', $params);

// Refund credit note
$params = [
    'creditnoteid' => 789,
    'method' => 'original_payment'
];
$result = localAPI('RefundCreditNote', $params);
```

## Hooks

```php
// Hook: CreditNoteCreated
add_hook('CreditNoteCreated', 1, function($vars) {
    // $vars['creditnoteid']
    // $vars['clientid']
    // $vars['total']
});

// Hook: CreditNoteRedeemed
add_hook('CreditNoteRedeemed', 1, function($vars) {
    // $vars['creditnoteid']
    // $vars['invoiceid']
    // $vars['amount']
});
```

## Best Practices

1. **Clear documentation**: Record reason for credit note
2. **Client communication**: Inform client of available credit
3. **Expiry management**: Send reminders before expiry
4. **Regular reconciliation**: Verify credit note usage
5. **Offer options**: Let customers choose refund or credit

## Related Documentation

- [Debit Notes](./whmcs-debit-notes.md)
- [Credit System](./whmcs-credit-system.md)
- [Applied Credits](./whmcs-applied-credits.md)
- [Refund Processing](./whmcs-refund-processing.md)