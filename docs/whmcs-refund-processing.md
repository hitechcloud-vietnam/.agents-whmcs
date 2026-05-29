# WHMCS Refund Processing

## Overview

Refund processing in WHMCS handles returning funds to customers for cancelled services, returned products, billing errors, or service disputes. The system supports various refund methods and provides comprehensive tracking.

## Refund Types

### Full Refund
Complete return of payment for cancelled service.

```php
[
    'type' => 'full',
    'original_transaction_id' => 12345,
    'amount' => 100.00,
    'reason' => 'Service cancelled before activation'
]
```

### Partial Refund
Return of portion of payment.

```php
[
    'type' => 'partial',
    'original_transaction_id' => 12345,
    'amount' => 50.00,
    'remaining' => 50.00,
    'reason' => 'Unused days refund'
]
```

### Prorated Refund
Time-based refund calculation.

```php
[
    'type' => 'prorated',
    'original_amount' => 100.00,
    'days_used' => 10,
    'days_total' => 30,
    'refund_amount' => 66.67,
    'reason' => 'Cancelled mid-cycle'
]
```

### Credit Refund
Refund added to client credit balance instead of payment.

```php
[
    'type' => 'credit',
    'original_transaction_id' => 12345,
    'amount' => 100.00,
    'credit_balance' => 100.00,
    'reason' => 'Client requested credit instead'
]
```

## Refund Configuration

### Settings

**Configuration > Billing > Refund Settings**

```php
[
    'allow_refunds' => true,
    'require_approval' => true,
    'approval_threshold' => 100.00,
    'auto_approve_under' => 10.00,
    'refund_window_days' => 30,         // Days to request refund
    'restock_fee_percent' => 0,
    'allow_partial' => true,
    'allow_credit_instead' => true
]
```

### Refund Methods

```php
// Available refund methods
[
    'original_payment' => true,         // Refund to original payment method
    'credit_balance' => true,           // Add to client credit
    'bank_transfer' => true,           // Bank transfer refund
    'check' => true,                   // Check refund
    'store_credit' => true              // Store credit only
]
```

## Refund Processing

### Manual Refund

**Admin: Billing > Refunds > New Refund**

```php
// Process refund
[
    'transaction_id' => 12345,
    'type' => 'full',
    'amount' => 100.00,
    'method' => 'original_payment',
    'reason' => 'Service cancellation',
    'notes' => 'Customer cancelled within 30 days',
    'approved_by' => 'admin_id',
    'processed_at' => '2024-05-15'
]
```

### Refund via Original Gateway

```php
// Stripe refund
[
    'gateway' => 'stripe',
    'charge_id' => 'ch_xxxx',
    'amount' => 100.00,
    'reason' => 'requested_by_customer',
    'process_immediately' => true
]

// PayPal refund
[
    'gateway' => 'paypal',
    'transaction_id' => 'PAY-xxxx',
    'amount' => 100.00,
    'note' => 'Service cancellation refund'
]
```

### Refund to Credit

```php
// Add refund to credit balance
[
    'transaction_id' => 12345,
    'amount' => 100.00,
    'method' => 'credit',
    'credit_description' => 'Refund - Invoice #5678',
    'credit_expiry' => null,
    'add_to_balance' => true
]
```

## Partial Refunds

### Partial Refund Calculation

```php
// Calculate partial refund
$originalAmount = 100.00;
$daysUsed = 10;
$daysInCycle = 30;
$proratedRefund = $originalAmount * (($daysInCycle - $daysUsed) / $daysInCycle);
// $66.67 refund, $33.33 retained
```

### Multiple Partial Refunds

```php
// Split refund into payments
[
    'transaction_id' => 12345,
    'original_amount' => 100.00,
    'refunds' => [
        ['amount' => 50.00, 'date' => '2024-05-15', 'reason' => 'First installment'],
        ['amount' => 50.00, 'date' => '2024-05-20', 'reason' => 'Final installment']
    ],
    'total_refunded' => 100.00,
    'remaining' => 0.00
]
```

## Refund Approval Workflow

### Approval Thresholds

```php
// Auto-approve small refunds
[
    'auto_approve_under' => 10.00,
    'require_approval_above' => 10.00,
    'escalate_above' => 500.00
]
```

### Approval Process

```php
// Approval workflow
[
    'step_1' => [
        'amount_range' => '0-50',
        'approver' => 'billing_agent'
    ],
    'step_2' => [
        'amount_range' => '50-200',
        'approver' => 'billing_manager'
    ],
    'step_3' => [
        'amount_range' => '200+',
        'approver' => 'finance_director'
    ]
]
```

## Refund Reasons

### Predefined Reasons

| Code | Description |
|------|-------------|
| CANCELLATION | Service cancellation |
| DUPLICATE | Duplicate payment |
| FRAUD | Fraudulent transaction |
| SERVICE_ISSUE | Service quality issue |
| EARLY_TERMINATION | Early termination discount |
| PROMO_ERROR | Promotional error |
| BILLING_ERROR | Billing calculation error |
| CUSTOMER_REQUEST | Customer request |

### Custom Reasons

```php
// Add custom refund reasons
[
    'reason' => 'LOYALTY_REFUND',
    'description' => 'Customer loyalty adjustment',
    'requires_approval' => true,
    'auto_approve' => false
]
```

## Refund Logging

### Transaction Records

```php
// Refund transaction record
[
    'refund_id' => 789,
    'original_transaction_id' => 12345,
    'amount' => 100.00,
    'method' => 'original_payment',
    'reason' => 'CANCELLATION',
    'notes' => 'Customer cancelled within trial',
    'status' => 'completed',
    'processed_by' => 'admin_id',
    'processed_at' => '2024-05-15',
    'gateway_ref' => 're_xxxx'
]
```

### Audit Trail

```php
// Complete history
[
    'original_payment' => [
        'id' => 12345,
        'amount' => 100.00,
        'date' => '2024-05-01'
    ],
    'refund_request' => [
        'id' => 789,
        'amount' => 100.00,
        'requested_by' => 'customer',
        'requested_at' => '2024-05-10'
    ],
    'approval' => [
        'approved_by' => 'admin_id',
        'approved_at' => '2024-05-12'
    ],
    'refund_processed' => [
        'processed_at' => '2024-05-15',
        'gateway_ref' => 're_xxxx',
        'completed_at' => '2024-05-15'
    }
]
```

## Refund to Different Payment Method

### Bank Transfer Refund

```php
// Refund via bank transfer
[
    'method' => 'bank_transfer',
    'amount' => 100.00,
    'bank_details' => [
        'account_name' => 'John Doe',
        'account_number' => '****1234',
        'routing_number' => '****5678',
        'bank_name' => 'Chase'
    ],
    'processing_days' => 3-5,
    'fees' => 0.00
]
```

### Check Refund

```php
// Check refund
[
    'method' => 'check',
    'amount' => 100.00,
    'payable_to' => 'John Doe',
    'mailing_address' => '123 Main St, City, ST 12345',
    'processing_days' => 5-7
]
```

## Refund Email Notifications

### Refund Processed Email

```smarty
Subject: Refund Processed - {$refund_amount}

Dear {$client_first_name},

Your refund of {$refund_amount} has been processed.

Original Transaction: {$original_transaction_id}
Refund Amount: {$refund_amount}
Refund Method: {$refund_method}
Refund Date: {$refund_date}

{if $refund_method == 'credit'}
This amount has been added to your credit balance.
{else}
The refund will appear on your {$payment_method} statement 
within 5-10 business days.
{/if}

{$company_name}
```

## Refund Reports

### Refund Summary Report

**Reports > Billing > Refund Report**

```php
[
    'period' => 'May 2024',
    'total_refunds' => 50,
    'total_refunded' => 5000.00,
    'average_refund' => 100.00,
    'by_reason' => [
        'CANCELLATION' => 30,
        'SERVICE_ISSUE' => 15,
        'BILLING_ERROR' => 5
    ],
    'by_method' => [
        'original_payment' => 40,
        'credit' => 8,
        'bank_transfer' => 2
    ]
]
```

## API Functions

```php
// Process refund
$params = [
    'transactionid' => 12345,
    'amount' => 100.00,
    'method' => 'original',
    'reason' => 'CANCELLATION'
];
$result = localAPI('ProcessRefund', $params);

// Get refund history
$params = [
    'clientid' => 123
];
$result = localAPI('GetRefunds', $params);

// Get pending refunds
$result = localAPI('GetPendingRefunds');

// Approve refund
$params = [
    'refundid' => 789,
    'approve' => true
];
$result = localAPI('ApproveRefund', $params);
```

## Hooks

```php
// Hook: PreRefund
add_hook('PreRefund', 1, function($vars) {
    // $vars['transactionid']
    // $vars['amount']
    // $vars['method']
    
    // Validate or prevent
    return ['allow' => true];
});

// Hook: RefundProcessed
add_hook('RefundProcessed', 1, function($vars) {
    // $vars['refundid']
    // $vars['transactionid']
    // $vars['amount']
    // $vars['method']
    // $vars['gateway_ref']
});
```

## Best Practices

1. **Clear policies**: Define refund eligibility clearly
2. **Timely processing**: Process refunds within timeframe
3. **Document reasons**: Record all refund reasons
4. **Maintain audit trail**: Keep complete records
5. **Monitor trends**: Track refund rates and reasons

## Related Documentation

- [Chargebacks](./whmcs-chargebacks.md)
- [Credit System](./whmcs-credit-system.md)
- [Service Cancellation](./whmcs-service-cancellation.md)
- [Applied Credits](./whmcs-applied-credits.md)