# WHMCS Payment Allocation

## Overview

Payment allocation determines how payments are applied to invoices and line items. WHMCS provides flexible allocation rules for handling partial payments, overpayments, and multi-invoice payments.

## Allocation Methods

### Manual Allocation
Admin assigns payment to specific invoice.

```php
[
    'method' => 'manual',
    'transaction_id' => 12345,
    'invoice_id' => 5678,
    'amount' => 100.00,
    'allocated_by' => 'admin_id'
]
```

### Automatic Allocation
System allocates payment based on rules.

```php
[
    'method' => 'automatic',
    'transaction_id' => 12345,
    'amount' => 150.00,
    'auto_rules' => [
        'invoice_priority' => 'oldest_first',
        'apply_fully' => true,
        'split_if_needed' => false
    ]
]
```

## Allocation Rules Configuration

### Default Rules

**Configuration > Billing > Payment Allocation**

```php
[
    'allocation_method' => 'oldest_first',
    'allow_overpayment_allocation' => true,
    'allow_partial_allocation' => true,
    'credit_priority' => 'expiry_first',
    'minimum_allocation' => 0.01
]
```

### Invoice Priority

```php
// Order invoices are paid
[
    'oldest_first' => [
        // Pay oldest due date first
        // Invoice due Jan 1
        // Invoice due Jan 15
        // First: Jan 1, Then: Jan 15
    ],
    'highest_first' => [
        // Pay largest amount first
        // $500, then $200, then $100
    ],
    'lowest_first' => [
        // Pay smallest first
        // $50, $100, $500
    ],
    'manual_priority' => [
        // Admin-defined order
        ['invoice_id' => 5678, 'priority' => 1],
        ['invoice_id' => 5679, 'priority' => 2]
    ]
]
```

### Client Invoice Order

```php
// Custom order per client
[
    'userid' => 123,
    'custom_order' => true,
    'invoice_order' => [5678, 5679, 5680]
]
```

## Single Payment Allocation

### One Invoice Payment

```php
// Simple allocation
[
    'transaction_id' => 12345,
    'amount' => 100.00,
    'allocations' => [
        ['invoice_id' => 5678, 'amount' => 100.00]
    ]
]
```

### Payment with Credit

```php
// Combined payment + credit
[
    'transaction_id' => 12345,
    'amount' => 100.00,
    'credit_applied' => 25.00,
    'allocations' => [
        ['invoice_id' => 5678, 'amount' => 125.00]
    ],
    'payment_breakdown' => [
        'cash' => 100.00,
        'credit' => 25.00
    ]
]
```

## Partial Payment Allocation

### Split Payment

```php
// Partial payment to single invoice
[
    'transaction_id' => 12345,
    'amount' => 50.00,
    'invoice_id' => 5678,
    'invoice_total' => 100.00,
    'allocated' => 50.00,
    'remaining_balance' => 50.00,
    'payment_complete' => false
]
```

### Multiple Partial Payments

```php
// Multiple payments to one invoice
[
    'invoice_id' => 5678,
    'total' => 100.00,
    'payments' => [
        ['id' => 12345, 'amount' => 50.00, 'date' => '2024-05-01'],
        ['id' => 12346, 'amount' => 50.00, 'date' => '2024-05-10']
    ],
    'status' => 'paid'
]
```

## Multi-Invoice Allocation

### Payment Covers Multiple Invoices

```php
// Allocate to multiple invoices
[
    'transaction_id' => 12345,
    'amount' => 350.00,
    'allocations' => [
        ['invoice_id' => 5678, 'amount' => 100.00],
        ['invoice_id' => 5679, 'amount' => 150.00],
        ['invoice_id' => 5680, 'amount' => 100.00]
    ],
    'fully_paid' => [5678, 5680],
    'partially_paid' => [5679]
]
```

### Order-Based Allocation

```php
// Allocate to invoices in order
[
    'transaction_id' => 12345,
    'amount' => 350.00,
    'invoice_order' => 'oldest_first',
    'allocations' => [
        ['invoice_id' => 5670, 'amount' => 50.00, 'status' => 'paid'],
        ['invoice_id' => 5671, 'amount' => 100.00, 'status' => 'paid'],
        ['invoice_id' => 5672, 'amount' => 200.00, 'status' => 'partial', 'remaining' => 0]
    ],
    'remaining_unallocated' => 0
]
```

## Overpayment Handling

### Credit Overpayment

```php
// Excess payment to credit balance
[
    'transaction_id' => 12345,
    'amount' => 150.00,
    'invoices_paid' => 100.00,
    'overpayment' => 50.00,
    'overpayment_handling' => 'credit',
    'credit_added' => 50.00,
    'credit_description' => 'Overpayment on invoice(s)'
]
```

### Manual Overpayment Handling

```php
// Admin decides overpayment
[
    'transaction_id' => 12345,
    'amount' => 150.00,
    'overpayment' => 50.00,
    'handling' => 'manual',
    'options' => [
        'refund' => 'Issue refund to customer',
        'credit' => 'Add to credit balance',
        'apply_to_next' => 'Apply to next invoice'
    ],
    'decided_action' => 'credit',
    'decided_by' => 'admin_id'
]
```

## Allocation Display

### Invoice Payment Status

```php
// Show payment allocation on invoice
[
    'invoice_id' => 5678,
    'total' => 100.00,
    'payments' => [
        ['date' => '2024-05-01', 'method' => 'Credit', 'amount' => 25.00],
        ['date' => '2024-05-05', 'method' => 'Credit Card', 'amount' => 75.00]
    ],
    'paid' => 100.00,
    'balance' => 0.00,
    'status' => 'Paid'
]
```

### Transaction Allocation Record

```php
// Show where payment went
[
    'transaction_id' => 12345,
    'date' => '2024-05-05',
    'amount' => 150.00,
    'method' => 'Credit Card',
    'allocations' => [
        ['invoice' => 'INV-5678', 'amount' => 100.00, 'status' => 'paid'],
        ['invoice' => 'INV-5679', 'amount' => 50.00, 'status' => 'paid']
    ],
    'unallocated' => 0.00
]
```

## Pre-Payment Allocation

### Advance Payment

```php
// Client pays before invoice exists
[
    'transaction_id' => 12345,
    'amount' => 100.00,
    'type' => 'prepayment',
    'status' => 'unallocated',
    'applied_to' => null,
    'available_balance' => 100.00
]
```

### Auto-Apply Prepayments

```php
// Apply prepayments when invoice created
[
    'auto_apply_prepayments' => true,
    'apply_order' => 'oldest_invoice',
    'apply_amount' => 'full_prepayment'
]
```

## Allocation Reversal

### Reverse Allocation

```php
// Reverse payment allocation
[
    'allocation_id' => 789,
    'transaction_id' => 12345,
    'invoice_id' => 5678,
    'amount' => 100.00,
    'reversed_by' => 'admin_id',
    'reversal_date' => '2024-05-10',
    'reason' => 'Invoice cancelled'
]
```

### After Reversal

```php
// Invoice status update
[
    'invoice_id' => 5678,
    'status' => 'Unpaid',
    'payments_reversed' => 100.00,
    'new_balance' => 100.00
]
```

## Credit Allocation Priority

### Credit Selection

```php
// Order credits are applied
[
    'credit_order' => [
        'expiry_first',          // Expiring credits first
        'oldest_first',          // Oldest credit first
        'largest_first',         // Largest balance first
        'smallest_first'         // Smallest balance first
    ],
    'preserve_expiring' => false,
    'preserve_bonus' => true           // Don't use bonus credits first
]
```

### Multiple Credit Allocation

```php
// Use multiple credits
[
    'invoice_id' => 5678,
    'credit_amount_needed' => 50.00,
    'credits_used' => [
        ['credit_id' => 123, 'amount' => 30.00],
        ['credit_id' => 456, 'amount' => 20.00]
    ],
    'total_credit' => 50.00
]
```

## API Functions

```php
// Allocate payment
$params = [
    'transactionid' => 12345,
    'invoiceid' => 5678,
    'amount' => 100.00
];
$result = localAPI('AllocatePayment', $params);

// Allocate to multiple
$params = [
    'transactionid' => 12345,
    'amounts' => [
        ['invoiceid' => 5678, 'amount' => 50.00],
        ['invoiceid' => 5679, 'amount' => 50.00]
    ]
];
$result = localAPI('AllocatePaymentMultiple', $params);

// Get allocation details
$params = [
    'transactionid' => 12345
];
$result = localAPI('GetPaymentAllocations', $params);

// Reverse allocation
$params = [
    'allocationid' => 789
];
$result = localAPI('ReverseAllocation', $params);
```

## Hooks

```php
// Hook: PrePaymentAllocation
add_hook('PrePaymentAllocation', 1, function($vars) {
    // $vars['transactionid']
    // $vars['invoiceid']
    // $vars['amount']
    
    // Modify or prevent allocation
    return ['allow' => true, 'amount' => $vars['amount']];
});

// Hook: PaymentAllocated
add_hook('PaymentAllocated', 1, function($vars) {
    // $vars['transactionid']
    // $vars['invoiceid']
    // $vars['amount']
    // $vars['allocationid']
});
```

## Best Practices

1. **Clear rules**: Define default allocation behavior
2. **Document decisions**: Log manual allocations
3. **Monitor overpayments**: Process excess funds promptly
4. **Customer communication**: Show where payments go
5. **Regular reconciliation**: Verify allocations

## Related Documentation

- [Credit System](./whmcs-credit-system.md)
- [Applied Credits](./whmcs-applied-credits.md)
- [Invoice Generation](./whmcs-invoice-generation.md)
- [Payment Methods](./whmcs-payment-methods.md)