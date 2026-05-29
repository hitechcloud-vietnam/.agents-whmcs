# WHMCS Late Fees

## Overview

Late fees are charges applied to invoices that remain unpaid after a specified grace period. WHMCS provides flexible late fee configuration including fixed amounts, percentage-based fees, and custom rules.

## Late Fee Configuration

### Accessing Settings

**Configuration > Billing > Late Fees**

### Basic Configuration

```php
[
    'late_fees_enabled' => true,
    'apply_after_days' => 7,            // Days after due date
    'fee_amount' => 5.00,               // Fee amount
    'fee_type' => 'fixed',              // fixed, percentage
    'recurring' => false,               // Apply fee repeatedly
    'maximum_fees' => 50.00             // Cap total fees
]
```

## Fee Types

### Fixed Fee

```php
// Single flat fee
[
    'fee_type' => 'fixed',
    'amount' => 5.00,
    'apply_once' => true
]

// Example: $100 invoice + $5 late fee = $105 total
```

### Percentage Fee

```php
// Percentage of invoice amount
[
    'fee_type' => 'percentage',
    'percentage' => 2.0,
    'minimum_fee' => 1.00,
    'maximum_fee' => 25.00
]

// Example: $100 invoice + 2% = $2.00 fee
// Minimum $1.00, Maximum $25.00
```

### Tiered Fee

```php
// Fee based on days overdue
[
    'fee_type' => 'tiered',
    'tiers' => [
        ['days' => 7, 'fee' => 5.00],
        ['days' => 14, 'fee' => 10.00],
        ['days' => 30, 'fee' => 25.00]
    ]
]

// 7 days overdue: $5
// 14 days overdue: $10
// 30 days overdue: $25
```

## Fee Application Rules

### When to Apply

```php
// Configure application conditions
[
    'apply_after_days' => 7,            // Days past due
    'exclude_weekends' => false,        // Skip weekends
    'minimum_invoice_amount' => 10.00, // Don't apply to small invoices
    'maximum_invoice_amount' => 5000.00
]
```

### Who to Apply

```php
// Client exclusions
[
    'exclude_client_groups' => ['premium', 'enterprise', 'vip'],
    'exclude_client_ids' => [123, 456],
    'exclude_new_clients_days' => 30   // New client grace period
]
```

### What to Apply

```php
// Invoice exclusions
[
    'exclude_invoice_types' => ['proforma', 'quote', 'credit'],
    'exclude_already_overdue' => false,  // Don't double-charge
    'exclude_zero_balance' => true
]
```

## Recurring Late Fees

### Weekly Recurring

```php
// Apply fee weekly until paid
[
    'recurring' => true,
    'recurring_interval' => 7,           // Every 7 days
    'maximum_recurring' => 4,           // Max 4 times
    'cap_total_fees' => 20.00
]

// Day 7: $5 fee
// Day 14: $5 fee (if still unpaid)
// Day 21: $5 fee
// Day 28: $5 fee
// Total: $20 maximum
```

### Monthly Recurring

```php
[
    'recurring' => true,
    'recurring_interval' => 30,
    'maximum_recurring' => 3,
    'cap_total_fees' => 15.00
]
```

## Late Fee Invoice

### Separate Invoice

```php
// Create dedicated invoice for fees
[
    'create_separate_invoice' => true,
    'invoice_group' => 'Late Fees',
    'invoice_duedays' => 0,
    'include_in_reminders' => true
]
```

### Add to Original Invoice

```php
// Add as line item
[
    'add_to_original' => true,
    'line_item_description' => 'Late Payment Fee',
    'separate_line_item' => true,
    'taxable' => false
]
```

### Invoice Line Item Format

```
+--------------------------------------------------+
| Late Payment Fee - Invoice #1234                 |  $5.00
+--------------------------------------------------+
```

## Custom Late Fee Rules

### Product-Specific Rules

```php
// Different fee for different products
[
    'product_id' => 1,
    'custom_late_fee' => true,
    'fee_amount' => 10.00,
    'fee_type' => 'fixed'
]
```

### Client-Specific Rules

```php
// Custom fee for specific clients
[
    'client_id' => 123,
    'custom_fee' => true,
    'fee_amount' => 0,             // No fee for VIP
    'extend_grace_period_days' => 14
]
```

## Late Fee Calculation Examples

### Fixed Fee Example

```php
// Invoice: $100, Due date passed 10 days
// Fee: $5 fixed

$lateFee = 5.00;
$newTotal = 100.00 + 5.00;  // $105.00
```

### Percentage Fee Example

```php
// Invoice: $500, 14 days overdue
// Fee: 2% of invoice total

$lateFee = 500.00 * 0.02;  // $10.00
$newTotal = 500.00 + 10.00;  // $510.00
```

### Tiered Fee Example

```php
// Invoice: $200, 21 days overdue
// Fee tiers: 7 days=$5, 14 days=$10, 21 days=$15

$lateFee = 15.00;  // Based on 21 days
$newTotal = 200.00 + 15.00;  // $215.00
```

## Late Fee Notification

### Email Notification

```php
// Notify client when fee applied
[
    'send_notification' => true,
    'template' => 'late_fee_applied',
    'include_fee_details' => true
]
```

### Notification Template

```smarty
Subject: Late Payment Fee Applied - Invoice {$invoice_number}

Dear {$client_first_name},

A late payment fee of {$late_fee_amount} has been applied to invoice 
{$invoice_number} due to non-payment.

Invoice Amount: {$invoice_total}
Late Fee: {$late_fee_amount}
New Total Due: {$new_total}

Please make payment immediately to avoid additional fees.

Pay Now: {$payment_link}

{$company_name}
```

## Late Fee Waivers

### Manual Waiver

**Admin: Invoice > Late Fee > Waive**

```php
// Waive fee
[
    'invoice_id' => 5678,
    'late_fee_id' => 123,
    'action' => 'waive',
    'reason' => 'Customer good standing',
    'waived_by' => 'admin_id',
    'waived_at' => '2024-05-15'
]
```

### Automatic Waiver Rules

```php
// Auto-waive for certain conditions
[
    'waive_for_client_groups' => ['premium'],
    'waive_for_first_offense' => true,
    'waive_if_amount_under' => 2.00
]
```

## Late Fee Reporting

### Late Fee Report

**Reports > Billing > Late Fee Report**

```php
// Report data
[
    'period' => 'May 2024',
    'total_late_fees_raised' => 1500.00,
    'total_fees_waived' => 200.00,
    'total_fees_collected' => 1200.00,
    'invoices_with_fees' => 150,
    'average_fee' => 10.00
]
```

### Fee Breakdown

```php
// By fee type
[
    'fixed_fees' => ['count' => 100, 'total' => 500.00],
    'percentage_fees' => ['count' => 50, 'total' => 1000.00]
]
```

## API Functions

```php
// Calculate late fee
$params = [
    'invoiceid' => 5678
];
$result = localAPI('CalculateLateFee', $params);

// Apply late fee
$params = [
    'invoiceid' => 5678,
    'fee_amount' => 5.00,
    'description' => 'Late payment fee'
];
$result = localAPI('ApplyLateFee', $params);

// Waive late fee
$params = [
    'invoiceid' => 5678,
    'fee_id' => 123,
    'reason' => 'Customer loyalty'
];
$result = localAPI('WaiveLateFee', $params);

// Get late fee settings
$result = localAPI('GetLateFeeSettings');
```

## Hooks

```php
// Hook: PreLateFeeApply
add_hook('PreLateFeeApply', 1, function($vars) {
    // $vars['invoiceid']
    // $vars['amount']
    // $vars['days_overdue']
    
    // Modify fee or prevent application
    return [
        'apply' => true,
        'amount' => $vars['amount']
    ];
});

// Hook: LateFeeApplied
add_hook('LateFeeApplied', 1, function($vars) {
    // $vars['invoiceid']
    // $vars['fee_id']
    // $vars['amount']
    // $vars['fee_type']
});
```

## Best Practices

1. **Be reasonable**: Fees shouldn't exceed invoice value
2. **Communicate clearly**: Make fee policy known upfront
3. **Apply consistently**: Same rules for all clients
4. **Document decisions**: Log all fee applications/waivers
5. **Track effectiveness**: Monitor if fees help collections

## Configuration Checklist

- [ ] Enable/disable late fees
- [ ] Set fee type (fixed/percentage)
- [ ] Configure amount
- [ ] Set grace period
- [ ] Configure recurring
- [ ] Set maximum cap
- [ ] Configure exclusions
- [ ] Set up notifications

## Related Documentation

- [Dunning Settings](./whmcs-dunning-settings.md)
- [Invoice Reminders](./whmcs-invoice-reminders.md)
- [Write Offs](./whmcs-write-offs.md)
- [Credit Notes](./whmcs-credit-notes.md)