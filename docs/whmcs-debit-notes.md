# WHMCS Debit Notes

## Overview

Debit notes are accounting documents that increase the amount owed by a customer. In WHMCS, debit notes are used to record additional charges, service adjustments, or corrections that increase client debt.

## Debit Note Overview

### When to Use Debit Notes

| Scenario | Description |
|----------|-------------|
| Additional Charges | Add services not on original invoice |
| Price Adjustments | Increase from agreed price |
| Fee Application | Apply additional fees |
| Credit Recovery | Recover previously credited amounts |
| Currency Adjustments | Exchange rate corrections |
| Correction | Fix undercharging error |

## Debit Note Configuration

### Settings

**Configuration > Billing > Credit Note Settings**

```php
[
    'debit_notes_enabled' => true,
    'auto_create' => false,
    'require_approval' => true,
    'approval_threshold' => 100.00,
    'auto_apply_to_invoice' => false,
    'number_format' => 'DN-{YYYY}-{SEQ}'
]
```

### Number Format

```php
// Configure debit note numbering
[
    'prefix' => 'DN',
    'year' => true,
    'month' => false,
    'sequence' => true,
    'reset_frequency' => 'yearly',
    'starting_number' => 1
]
```

## Creating Debit Notes

### Manual Creation

**Admin: Billing > Debit Notes > Create Debit Note**

```php
// Debit note creation
[
    'client_id' => 123,
    'number' => 'DN-2024-0123',
    'date' => '2024-05-15',
    'due_date' => '2024-05-22',
    'items' => [
        ['description' => 'Additional bandwidth', 'amount' => 50.00],
        ['description' => 'Overage charges', 'amount' => 25.00]
    ],
    'subtotal' => 75.00,
    'tax' => 15.00,
    'total' => 90.00,
    'status' => 'Pending'
]
```

### Line Items

```php
// Add items to debit note
[
    'description' => 'Additional storage - 100GB',
    'quantity' => 1,
    'unit_price' => 10.00,
    'amount' => 10.00,
    'taxable' => true,
    'tax_rate' => 20
]
```

## Debit Note Status

### Status Flow

```
Draft -> Pending -> Applied -> Paid
               -> Cancelled
```

| Status | Description |
|--------|-------------|
| Draft | Not finalized, can be edited |
| Pending | Issued, awaiting payment |
| Applied | Applied to client account |
| Paid | Payment received |
| Cancelled | Debit note cancelled |

### Status Transitions

```php
// Draft to Pending
[
    'action' => 'finalize',
    'from_status' => 'Draft',
    'to_status' => 'Pending',
    'notify_client' => true
]

// Pending to Applied
[
    'action' => 'apply',
    'from_status' => 'Pending',
    'to_status' => 'Applied',
    'create_invoice' => true
]
```

## Debit Note vs Invoice

### Key Differences

| Aspect | Debit Note | Invoice |
|--------|-----------|---------|
| Purpose | Additional charges | Initial billing |
| Numbering | Separate sequence | Separate sequence |
| Creation | Manual trigger | Auto or manual |
| Tax | Can include tax | Standard tax |
| Application | Applied to account | Payment required |

### When to Use Debit Note

```
Invoice (Original): $100
Debit Note (Additional): +$25
Total Due: $125

Debit note increases amount owed.
Credit note decreases amount owed.
```

## Applying Debit Notes

### Apply to Client Account

```php
// Apply debit note to client
[
    'debit_note_id' => 789,
    'client_id' => 123,
    'amount' => 90.00,
    'applied_at' => '2024-05-15',
    'applied_by' => 'admin_id'
]
```

### Apply to Invoice

```php
// Apply to existing invoice
[
    'debit_note_id' => 789,
    'invoice_id' => 5678,
    'amount' => 90.00,
    'application_type' => 'deduction'
]

// Invoice adjustment
[
    'invoice_id' => 5678,
    'original_total' => 100.00,
    'debit_note_applied' => 25.00,
    'new_total' => 125.00
]
```

### Partial Application

```php
// Apply portion of debit note
[
    'debit_note_id' => 789,
    'total' => 90.00,
    'applied_amount' => 50.00,
    'remaining' => 40.00
]
```

## Debit Note Display

### Debit Note Document

```php
// Standard debit note format
[
    'header' => [
        'number' => 'DN-2024-0123',
        'date' => 'May 15, 2024',
        'due_date' => 'May 22, 2024'
    ],
    'client' => [...],
    'items' => [...],
    'totals' => [...],
    'footer' => 'Payment due within 7 days'
]
```

### Invoice Integration

```
Debit Note Applied to Invoice #5678

Original Invoice Total: $100.00
Debit Note DN-2024-0123: +$25.00
---------------------------------
New Balance Due: $125.00
```

## Refunds on Debit Notes

### Full Refund

```php
// Cancel debit note
[
    'debit_note_id' => 789,
    'action' => 'cancel',
    'reason' => 'Error in billing',
    'cancelled_by' => 'admin_id',
    'cancelled_at' => '2024-05-16'
]
```

### Partial Refund

```php
// Reduce debit note amount
[
    'debit_note_id' => 789,
    'original_amount' => 90.00,
    'refund_amount' => 25.00,
    'remaining' => 65.00,
    'reason' => 'Partial service not delivered'
]
```

## Reporting

### Debit Note Report

**Reports > Billing > Debit Note Report**

```php
// Report structure
[
    'period' => 'May 2024',
    'total_debit_notes' => 50,
    'total_value' => 5000.00,
    'applied' => 4000.00,
    'pending' => 1000.00,
    'cancelled' => 0.00,
    'by_reason' => [
        'additional_service' => 2000.00,
        'overage' => 1500.00,
        'correction' => 1000.00,
        'fee' => 500.00
    ]
]
```

## API Functions

```php
// Create debit note
$params = [
    'clientid' => 123,
    'date' => '2024-05-15',
    'items' => [
        ['description' => 'Additional service', 'amount' => 50.00]
    ]
];
$result = localAPI('CreateDebitNote', $params);

// Get debit notes
$params = [
    'clientid' => 123
];
$result = localAPI('GetDebitNotes', $params);

// Apply debit note
$params = [
    'debitnoteid' => 789,
    'amount' => 90.00
];
$result = localAPI('ApplyDebitNote', $params);

// Cancel debit note
$params = [
    'debitnoteid' => 789,
    'reason' => 'Error'
];
$result = localAPI('CancelDebitNote', $params);
```

## Hooks

```php
// Hook: DebitNoteCreated
add_hook('DebitNoteCreated', 1, function($vars) {
    // $vars['debitnoteid']
    // $vars['clientid']
    // $vars['total']
});

// Hook: DebitNoteApplied
add_hook('DebitNoteApplied', 1, function($vars) {
    // $vars['debitnoteid']
    // $vars['invoiceid']
    // $vars['amount']
});
```

## Best Practices

1. **Clear documentation**: Record reason for debit note
2. **Approval workflow**: Require approval for large amounts
3. **Client communication**: Notify clients of additional charges
4. **Regular review**: Monitor debit note patterns
5. **Accurate recording**: Ensure proper accounting entries

## Related Documentation

- [Credit Notes](./whmcs-credit-notes.md)
- [Invoice Generation](./whmcs-invoice-generation.md)
- [Transaction Fees](./whmcs-transaction-fees.md)
- [Billing Configuration](./whmcs-billing-config.md)