# WHMCS Transaction Fees

## Overview

Transaction fees encompass all costs associated with processing payments, including gateway fees, bank charges, currency conversion costs, and processing surcharges. WHMCS provides comprehensive tracking and management of these fees.

## Fee Types

### Gateway Fees
Credit card processing fees from payment providers.

```php
[
    'type' => 'gateway',
    'source' => 'stripe',
    'percentage' => 2.9,
    'fixed' => 0.30,
    'calculated_on' => 'gross_amount'
]
```

### Bank Fees
Charges from financial institutions for transfers and settlements.

```php
[
    'type' => 'bank',
    'description' => 'Wire transfer fee',
    'amount' => 15.00,
    'per_transaction' => true
]
```

### Currency Conversion Fees
Costs when accepting payments in different currencies.

```php
[
    'type' => 'currency_conversion',
    'rate_markup' => 1.0,    // 1% above market rate
    'minimum_fee' => 1.00
]
```

### Chargeback Fees
Fees charged for disputed or reversed transactions.

```php
[
    'type' => 'chargeback',
    'amount' => 15.00,       // Standard fee
    'disputed_amount_fee' => 0  // Additional % on disputed amount
]
```

## Transaction Fee Configuration

### Global Settings

**Configuration > Billing > Transaction Fees**

```php
[
    'track_fees' => true,
    'fee_account' => 'Gateway Fees Expense',
    'record_individually' => true,
    'include_in_reports' => true
]
```

### Per-Transaction Tracking

```php
// Record each fee
[
    'transaction_id' => 12345,
    'invoice_id' => 5678,
    'amount' => 100.00,
    'fee_type' => 'gateway',
    'fee_amount' => 3.20,
    'fee_percentage' => 2.9,
    'fee_fixed' => 0.30,
    'currency' => 'USD',
    'net_received' => 96.80,
    'gateway' => 'stripe'
]
```

## Fee Calculation

### Standard Calculation

```php
function calculateTransactionFee($amount, $percentage, $fixed) {
    $percentageFee = $amount * ($percentage / 100);
    $totalFee = $percentageFee + $fixed;
    return [
        'percentage_fee' => $percentageFee,
        'fixed_fee' => $fixed,
        'total_fee' => round($totalFee, 2),
        'net_amount' => $amount - $totalFee
    ];
}
```

### Example

```php
// $100 transaction at 2.9% + $0.30
$amount = 100.00;
$percentage = 2.9;
$fixed = 0.30;

$fees = calculateTransactionFee($amount, $percentage, $fixed);
// Result:
// percentage_fee: $2.90
// fixed_fee: $0.30
// total_fee: $3.20
// net_amount: $96.80
```

## Fee Reporting

### Transaction Fee Report

**Reports > Billing > Transaction Fee Report**

```php
// Report parameters
[
    'start_date' => '2024-05-01',
    'end_date' => '2024-05-31',
    'gateway' => 'all',       // or specific gateway
    'group_by' => 'gateway'   // gateway, month, client
]
```

### Report Output

```php
// Monthly fee summary
{
    'period' => 'May 2024',
    'total_transactions' => 500,
    'gross_volume' => 50000.00,
    'total_fees' => 1450.00,
    'net_revenue' => 48550.00,
    'effective_fee_rate' => 2.9,
    'by_gateway' => [
        'stripe' => [
            'transactions' => 300,
            'volume' => 30000.00,
            'fees' => 900.00,
            'effective_rate' => 3.0
        ],
        'paypal' => [
            'transactions' => 200,
            'volume' => 20000.00,
            'fees' => 550.00,
            'effective_rate' => 2.75
        ]
    ]
}
```

## Fee Allocation

### By Product/Service

```php
// Track fees per product
[
    'service_id' => 123,
    'transaction_id' => 456,
    'gross_amount' => 100.00,
    'fee_amount' => 3.20,
    'net_amount' => 96.80,
    'fee_percentage' => 3.2
]
```

### By Client

```php
// Aggregate fees per client
[
    'client_id' => 789,
    'period' => 'May 2024',
    'total_transactions' => 5,
    'total_volume' => 500.00,
    'total_fees' => 16.00
]
```

### By Invoice

```php
// Fees per invoice
[
    'invoice_id' => 101,
    'transactions' => [
        ['id' => 1, 'amount' => 75.00, 'fee' => 2.48],
        ['id' => 2, 'amount' => 25.00, 'fee' => 1.03]
    ],
    'total_fees' => 3.51
]
```

## Fee Adjustments

### Manual Adjustments

```php
// Adjust fee if needed
[
    'transaction_id' => 123,
    'adjustment_type' => 'refund',    // refund, waive, correct
    'original_fee' => 3.20,
    'adjusted_fee' => 1.60,
    'reason' => 'Customer loyalty adjustment',
    'admin_id' => 1
]
```

### Fee Waivers

```php
// Waive fees for special cases
[
    'reason' => 'Customer service recovery',
    'approved_by' => 'admin',
    'transaction_id' => 123,
    'waived_amount' => 3.20
]
```

## Currency Conversion Fees

### Conversion Rate

```php
// When customer pays in non-base currency
[
    'customer_currency' => 'EUR',
    'base_currency' => 'USD',
    'invoice_amount' => 100.00,
    'market_rate' => 1.10,       // 1 EUR = 1.10 USD
    'markup_rate' => 1.11,        // +1% markup
    'converted_amount' => 90.09,  // EUR equivalent
    'conversion_fee' => 0.90
]
```

### Multi-Currency Handling

```php
// Accept payment in customer's currency
[
    'invoice_currency' => 'GBP',
    'invoice_amount' => 80.00,
    'exchange_rate' => 1.25,
    'base_currency_amount' => 100.00,
    'exchange_rate_fee' => 1.00
]
```

## Fee Reconciliation

### Daily Reconciliation

```php
// Compare gateway reports to WHMCS
[
    'date' => '2024-05-15',
    'gateway' => 'stripe',
    'gateway_total' => 5000.00,
    'whmcs_total' => 5000.00,
    'difference' => 0.00,
    'transactions_matched' => 50,
    'transactions_unmatched' => 0
]
```

### Discrepancy Handling

```php
// Unmatched transactions
[
    'gateway_txn' => 'ch_123',
    'amount' => 100.00,
    'whmcs_status' => 'not_found',
    'action' => 'investigate',
    'resolution' => 'awaiting'
]
```

## API Functions

```php
// Get transaction fees for period
$params = [
    'start_date' => '2024-05-01',
    'end_date' => '2024-05-31',
    'gateway' => 'stripe'
];
$result = localAPI('GetTransactionFees', $params);

// Calculate fee for amount
$params = [
    'amount' => 100.00,
    'gateway' => 'stripe',
    'currency' => 'USD'
];
$result = localAPI('CalculateTransactionFee', $params);
```

## Hooks

```php
// Hook: TransactionFeeCalculated
add_hook('TransactionFeeCalculated', 1, function($vars) {
    // $vars['transaction_id']
    // $vars['amount']
    // $vars['gateway']
    // $vars['fee_amount']
    
    // Modify or add additional fees
});

// Hook: TransactionFeeRecorded
add_hook('TransactionFeeRecorded', 1, function($vars) {
    // Record to external accounting system
});
```

## Best Practices

1. **Track all fees**: Record every transaction fee
2. **Use automation**: Integrate with gateway reports
3. **Regular reconciliation**: Check for discrepancies
4. **Fee analysis**: Identify high-fee gateways
5. **Negotiate rates**: Review rates periodically

## Related Documentation

- [Gateway Fees](./whmcs-gateway-fees.md)
- [Payment Methods](./whmcs-payment-methods.md)
- [Currency Conversion](./whmcs-currency-conversion.md)
- [Multi-Currency](./whmcs-multi-currency.md)