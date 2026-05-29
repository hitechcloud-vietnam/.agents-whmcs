# WHMCS Gateway Fees

## Overview

Gateway fees are transaction costs charged by payment processors for processing credit card and other electronic payments. WHMCS can track, display, and optionally pass these fees to customers.

## Gateway Fee Configuration

### Accessing Settings

**Configuration > Payments > Payment Gateways > Gateway Fees**

### Basic Configuration

```php
// Enable gateway fees
[
    'enable_gateway_fees' => true,
    'fee_structure' => 'pass_through',  // pass_through, absorbed, both
    'display_to_customer' => true
]
```

### Fee Models

#### Pass-Through Model
Customer pays the full gateway fee.

```php
[
    'model' => 'pass_through',
    'invoice_includes_fee' => true,
    'fee_label' => 'Payment Processing Fee'
]
```

#### Absorbed Model
Business absorbs the gateway fee.

```php
[
    'model' => 'absorbed',
    'fees_from_revenue' => true,
    'record_in_reports' => true
]
```

#### Hybrid Model
Fees above threshold passed to customer.

```php
[
    'model' => 'hybrid',
    'minimum_transaction' => 20.00,
    'fee_threshold' => 2.50,
    'pass_above_threshold' => true
]
```

## Fee Calculation

### Percentage + Fixed Fee

```php
// Standard fee structure
[
    'gateway' => 'stripe',
    'percentage' => 2.9,    // 2.9%
    'fixed' => 0.30,        // $0.30 per transaction
    'currency' => 'USD',
    'cap' => 10.00          // Maximum fee
]

// Example: $100 charge
// Fee: ($100 * 0.029) + $0.30 = $3.20
// Customer pays: $103.20
```

### Tiered Fees

```php
// Volume-based fees
[
    'gateway' => 'stripe',
    'tiers' => [
        ['min' => 0, 'max' => 100, 'percentage' => 3.5, 'fixed' => 0.30],
        ['min' => 101, 'max' => 500, 'percentage' => 2.9, 'fixed' => 0.30],
        ['min' => 501, 'max' => null, 'percentage' => 2.5, 'fixed' => 0.20]
    ]
]
```

### Currency-Based Fees

```php
// Different rates per currency
[
    'USD' => ['percentage' => 2.9, 'fixed' => 0.30],
    'EUR' => ['percentage' => 2.9, 'fixed' => 0.25],
    'GBP' => ['percentage' => 2.5, 'fixed' => 0.20]
]
```

## Per-Gateway Configuration

### Stripe Fees

```php
[
    'gateway' => 'stripe',
    'card_percentage' => 2.9,
    'card_fixed' => 0.30,
    'amex_percentage' => 3.5,
    'international_percentage' => 3.5,
    'currency_conversion_fee' => 1.0
]
```

### PayPal Fees

```php
[
    'gateway' => 'paypal',
    'domestic_percentage' => 3.49,
    'international_percentage' => 4.49,
    'fixed' => 0.30,
    'micropayment_fee' => 0.05
]
```

### Authorize.net Fees

```php
[
    'gateway' => 'authorizenet',
    'card_percentage' => 3.05,
    'fixed' => 0.10,
    'charge_back_fee' => 25.00,
    'monthly_fee' => 25.00,
    'gateway_fee_approval' => 0.005
]
```

## Fee Display Options

### Invoice Display

```php
// Add fee as separate line item
[
    'show_on_invoice' => true,
    'line_item_format' => [
        'description' => 'Payment Processing Fee',
        'taxable' => false,
        'ledger_entry' => 'gateway_fees'
    ]
]
```

### Invoice Example

```
+--------------------------------------------------+
| Item                          | Amount           |
+--------------------------------------------------+
| Cloud Hosting - Monthly       | $100.00          |
+--------------------------------------------------+
|                                |                  |
| Subtotal:                     | $100.00          |
| Tax:                          | $20.00           |
| Payment Processing Fee (2.9%):| $3.20            |
+--------------------------------------------------+
| Total Due:                    | $123.20          |
+--------------------------------------------------+
```

### Client Notification

```php
// Payment confirmation includes fee
{
    'email_fee_breakdown' => true,
    'include_fee_in_total' => true,
    'fee_description' => 'Includes 2.9% + $0.30 processing fee'
}
```

## Fee Exceptions

### Amount-Based Exceptions

```php
// No fee for small transactions
[
    'minimum_for_fee' => 5.00,
    'absorb_minimum' => true
]

// Cap on fees
[
    'maximum_fee' => 50.00,
    'absorb_excess' => true
]
```

### Client-Based Exceptions

```php
// Waive fees for premium clients
[
    'client_groups_exempt' => ['premium', 'enterprise'],
    'client_ids_exempt' => [123, 456]
]
```

### Product-Based Exceptions

```php
// No fees on certain products
[
    'products_exempt' => [10, 15, 20]
]
```

## Fee Accounting

### Journal Entries

```php
// Record gateway fees
[
    'entries' => [
        ['account' => 'Revenue', 'debit' => 100.00, 'credit' => 0],
        ['account' => 'Accounts Receivable', 'debit' => 0, 'credit' => 100.00],
        ['account' => 'Gateway Fees', 'debit' => 3.20, 'credit' => 0],
        ['account' => 'Payment Processing Liability', 'debit' => 0, 'credit' => 3.20]
    ]
]
```

### Reporting

**Reports > Billing > Gateway Fee Report**

```php
// Fee summary
[
    'period' => 'May 2024',
    'total_transactions' => 500,
    'total_volume' => 50000.00,
    'total_fees' => 1450.00,
    'effective_rate' => 2.9,
    'by_gateway' => [
        'stripe' => ['fees' => 850.00, 'transactions' => 300],
        'paypal' => ['fees' => 600.00, 'transactions' => 200]
    ]
]
```

## Refund Fee Handling

### Fee on Refunds

```php
// Refund fee policy
[
    'refund_fee' => 'proportional',  // proportional, full, none
    'refund_percentage' => 2.9,
    'refund_fixed' => 0.30
]

// Example
// Original: $100 + $3.20 fee = $103.20
// Refund: $100 + (proportional fee return) = $100 + $2.90 = $102.90
```

### Chargeback Fees

```php
// Chargeback fee handling
[
    'chargeback_fee' => 15.00,
    'chargeback_fee_pass_through' => true,
    'record_in_separate_account' => true
]
```

## API Functions

```php
// Calculate gateway fee
$params = [
    'gateway' => 'stripe',
    'amount' => 100.00,
    'currency' => 'USD'
];
$result = localAPI('CalculateGatewayFee', $params);

// Response
{
    "result" => "success",
    "fee_amount" => 3.20,
    "percentage_fee" => 2.90,
    "fixed_fee" => 0.30,
    "total_charge" => 103.20
}
```

## Hooks

```php
// Hook: GatewayFeeCalculate
add_hook('GatewayFeeCalculate', 1, function($vars) {
    // $vars['gateway']
    // $vars['amount']
    // $vars['calculated_fee']
    
    // Modify fee if needed
    if ($vars['amount'] > 1000) {
        return ['calculated_fee' => $vars['calculated_fee'] * 0.9];
    }
});

// Hook: GatewayFeeCharged
add_hook('GatewayFeeCharged', 1, function($vars) {
    // $vars['invoiceid']
    // $vars['gateway']
    // $vars['fee_amount']
    // $vars['transaction_id']
});
```

## Best Practices

1. **Transparent fees**: Clearly show fees to customers
2. **Consistent policy**: Apply fees consistently across gateways
3. **Regular review**: Update fees as gateway rates change
4. **Record tracking**: Maintain accurate fee accounting
5. **Exception management**: Handle special cases properly

## Configuration Checklist

- [ ] Enable gateway fees
- [ ] Configure fee structure
- [ ] Set per-gateway rates
- [ ] Define exceptions
- [ ] Enable invoice display
- [ ] Configure refund handling
- [ ] Set up reporting

## Related Documentation

- [Payment Methods](./whmcs-payment-methods.md)
- [Transaction Fees](./whmcs-transaction-fees.md)
- [Invoice Generation](./whmcs-invoice-generation.md)
- [Refund Processing](./whmcs-refund-processing.md)