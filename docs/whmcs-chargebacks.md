# WHMCS Chargeback Handling

## Overview

Chargebacks occur when a customer disputes a payment with their bank or credit card company, resulting in a reversal of funds. WHMCS provides comprehensive chargeback management including tracking, response, and prevention tools.

## Chargeback Process Flow

```
Customer Dispute -> Bank Notification -> Merchant Response -> Resolution
     1. Customer contacts bank                4. Evidence submitted
     2. Bank places hold                     5. Bank reviews
     3. Merchant notified                    6. Win/Lose/Partial
```

## Chargeback Status Types

| Status | Description |
|--------|-------------|
| Pre-Arbitration | Warning before formal chargeback |
| Pending | Awaiting response |
| Under Review | Evidence submitted, awaiting decision |
| Won | Chargeback decided in merchant favor |
| Lost | Chargeback decided against merchant |
| Resolved | Settlement reached |

## Chargeback Configuration

### Settings

**Configuration > Billing > Chargeback Settings**

```php
[
    'chargeback_alerts' => true,
    'auto_respond_days' => 7,              // Days to respond
    'email_on_chargeback' => 'billing@example.com',
    'auto_suspend_service' => true,
    'suspend_after_days' => 3,
    'allow_respond' => true,
    'require_evidence' => true
]
```

### Notification Settings

```php
[
    'notify_admin' => true,
    'notify_sales' => false,
    'notify_account_manager' => true,
    'slack_webhook' => null,
    'alert_threshold_amount' => 100.00
]
```

## Chargeback Detection

### Automatic Detection

```php
// Detect new chargebacks via gateway webhook
[
    'event' => 'chargeback.created',
    'transaction_id' => 'ch_xxxx',
    'amount' => 100.00,
    'reason_code' => 'fraud',
    'customer_email' => 'customer@example.com',
    'dispute_url' => 'https://...'
]
```

### Manual Import

```php
// Import chargeback from gateway
[
    'action' => 'import_chargeback',
    'gateway' => 'stripe',
    'transaction_id' => 'ch_xxxx',
    'amount' => 100.00,
    'reason' => 'fraud'
]
```

## Responding to Chargebacks

### Evidence Collection

**Admin: Billing > Chargebacks > Respond**

```php
// Response types
[
    'evidence_type' => 'retrieval_request',
    'evidence' => [
        ['type' => 'invoice', 'file' => 'invoice_5678.pdf'],
        ['type' => 'receipt', 'file' => 'receipt.pdf'],
        ['type' => 'shipping', 'file' => 'shipping_proof.pdf'],
        ['type' => 'communication', 'file' => 'emails.pdf']
    ],
    'description' => 'Customer authorized charge',
    'submit_by' => '2024-05-22'
]
```

### Evidence Types

| Type | Description | Required |
|------|-------------|----------|
| Invoice | Copy of invoice | Yes |
| Receipt | Payment receipt | Yes |
| Shipping Proof | Delivery confirmation | If physical |
| Service Confirmation | Service was provided | If applicable |
| Communication | Email/chat with customer | If applicable |
| Cancellation Policy | Policy customer agreed to | If applicable |

### Writing Response

```php
// Chargeback response
[
    'chargeback_id' => 789,
    'response_type' => 'evidence',
    'amount_in_dispute' => 100.00,
    'evidence' => [
        'invoice_provided' => true,
        'customer_email_confirmed' => true,
        'service_delivered' => true,
        'cancellation_policy_presented' => true
    ],
    'explanation' => 'Customer agreed to terms and received service.
                     This is a fraudulent claim as customer has 
                     already used service for 30 days.',
    'submit_by' => '2024-05-22'
]
```

## Chargeback Fees

### Fee Configuration

```php
// Gateway chargeback fees
[
    'stripe' => ['fee' => 15.00],
    'paypal' => ['fee' => 20.00],
    'authorizenet' => ['fee' => 25.00]
]
```

### Fee Handling

```php
// Charge fee to client
[
    'charge_client' => true,
    'charge_fee_account' => 'Chargeback Fees',
    'invoice_chargeback_fee' => true,
    'include_in_dunning' => true,
    'fee_amount' => 15.00
]
```

### Fee Invoice

```
+--------------------------------------------------+
| Chargeback Fee - Transaction #12345               |
+--------------------------------------------------+
| Original Chargeback Amount:        $100.00       |
| Chargeback Fee:                     $15.00        |
+--------------------------------------------------+
| Total Due:                       $115.00        |
+--------------------------------------------------+
```

## Service Suspension

### Auto Suspension

```php
// Suspend service on chargeback
[
    'auto_suspend' => true,
    'suspend_days' => 3,                    // Days after chargeback
    'suspend_reason' => 'Chargeback - Payment dispute',
    'notify_customer' => true,
    'suspension_template' => 'chargeback_suspension'
]
```

### Manual Suspension

```php
// Manual suspension decision
[
    'service_id' => 123,
    'action' => 'suspend',
    'reason' => 'Chargeback received',
    'preserve_until_resolution' => true
]
```

### Suspension During Resolution

```php
// Keep suspended until resolved
[
    'keep_suspended' => true,
    'restore_on_win' => true,
    'restore_on_partial' => 'retain_partial',
    'termination_on_loss' => true,
    'termination_days' => 7
]
```

## Chargeback Prevention

### Prevention Strategies

```php
// Proactive measures
[
    'clear_invoice_descriptions' => true,
    'customer_verification' => true,
    'fraud_detection' => true,
    'chargeback_disclaimer' => true,
    'dispute_resolution_first' => true
]
```

### Clear Billing Descriptors

```php
// Ensure clear charge descriptions
[
    'billing_descriptor' => 'YOUR COMPANY*',
    'support_phone_on_statement' => true,
    'descriptor_url' => 'https://yourdomain.com'
]
```

### Customer Communication

```php
// Before chargeback
[
    'send_receipts' => true,
    'clear_cancellation_policy' => true,
    'easy_dispute_channel' => true,
    'proactive_refunds' => true
]
```

## Chargeback Tracking

### Chargeback Record

```php
// Complete chargeback data
[
    'chargeback_id' => 789,
    'original_transaction_id' => 12345,
    'invoice_id' => 5678,
    'client_id' => 123,
    'amount' => 100.00,
    'reason_code' => 'fraud',
    'reason_description' => 'Customer claims unauthorized',
    'status' => 'under_review',
    'created_at' => '2024-05-15',
    'respond_by' => '2024-05-22',
    'service_suspended' => true,
    'fee_charged' => 15.00
]
```

### History Log

```php
// Chargeback events
[
    ['date' => '2024-05-15', 'event' => 'chargeback_created', 'by' => 'stripe'],
    ['date' => '2024-05-15', 'event' => 'service_suspended', 'by' => 'system'],
    ['date' => '2024-05-16', 'event' => 'admin_notified', 'by' => 'system'],
    ['date' => '2024-05-17', 'event' => 'evidence_submitted', 'by' => 'admin'],
    ['date' => '2024-05-20', 'event' => 'bank_reviewing', 'by' => 'stripe']
]
```

## Chargeback Reports

### Chargeback Report

**Reports > Billing > Chargeback Report**

```php
// Report data
[
    'period' => 'May 2024',
    'total_chargebacks' => 15,
    'total_amount' => 1500.00,
    'won' => 8,
    'lost' => 5,
    'pending' => 2,
    'win_rate' => 61.5,
    'total_fees' => 225.00,
    'total_recovered' => 800.00
]
```

### Reason Analysis

```php
// Chargeback reasons breakdown
[
    'fraud' => ['count' => 5, 'amount' => 500.00, 'win_rate' => 80],
    'product_not_received' => ['count' => 4, 'amount' => 400.00, 'win_rate' => 75],
    'product_unacceptable' => ['count' => 3, 'amount' => 300.00, 'win_rate' => 33],
    'credit_not_processed' => ['count' => 3, 'amount' => 300.00, 'win_rate' => 100]
]
```

## API Functions

```php
// Get chargebacks
$result = localAPI('GetChargebacks', [
    'status' => 'pending'
]);

// Submit evidence
$result = localAPI('SubmitChargebackEvidence', [
    'chargeback_id' => 789,
    'evidence' => [...],
    'description' => '...'
]);

// Update chargeback status
$result = localAPI('UpdateChargebackStatus', [
    'chargeback_id' => 789,
    'status' => 'won'
]);
```

## Hooks

```php
// Hook: ChargebackCreated
add_hook('ChargebackCreated', 1, function($vars) {
    // $vars['chargeback_id']
    // $vars['transaction_id']
    // $vars['amount']
    // $vars['reason_code']
    
    // Auto-suspend service, notify admin, etc.
});

// Hook: ChargebackResolved
add_hook('ChargebackResolved', 1, function($vars) {
    // $vars['chargeback_id']
    // $vars['status']  // won, lost, resolved
    // $vars['amount']
    // $vars['fee_amount']
    
    // Restore service, charge fees, etc.
});
```

## Best Practices

1. **Respond quickly**: Meet deadlines for evidence
2. **Document everything**: Keep records of all interactions
3. **Provide strong evidence**: Invoices, receipts, communication
4. **Monitor patterns**: Track chargeback reasons
5. **Prevent proactively**: Address issues before chargebacks

## Related Documentation

- [Refund Processing](./whmcs-refund-processing.md)
- [Service Suspension](./whmcs-service-suspension.md)
- [Transaction Fees](./whmcs-transaction-fees.md)
- [Gateway Fees](./whmcs-gateway-fees.md)