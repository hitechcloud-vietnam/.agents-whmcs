# WHMCS Write-Offs

## Overview

Write-offs are accounting entries that remove uncollectible invoice amounts from accounts receivable. WHMCS provides mechanisms to write off bad debts while maintaining proper accounting records and audit trails.

## Write-Off Configuration

### Accessing Write-Off Settings

**Configuration > Billing > Write-Off Settings**

### Basic Configuration

```php
[
    'write_off_enabled' => true,
    'require_approval' => true,
    'approval_role' => 'billing_admin',
    'auto_write_off_days' => 90,        // Auto write off after days
    'minimum_write_off' => 0.01,
    'require_reason' => true,
    'notify_client' => true
]
```

## Write-Off Reasons

### Predefined Reasons

| Reason Code | Description |
|-------------|-------------|
| BAD_DEBT | Uncollectible debt |
| CUSTOMER_DISPUTE | Valid dispute |
| ECONOMIC_LOSS | Economic conditions |
| BANKRUPTCY | Customer bankruptcy |
| SMALL_BALANCE | Not worth collecting |
| COURT_JUDGMENT | Legal judgment against |
| STATUTE_LIMITED | Collection barred |
| GOODWILL | Customer retention |

### Custom Reasons

```php
// Add custom write-off reasons
[
    'custom_reasons' => [
        ['code' => 'LOYALTY', 'description' => 'Customer loyalty write-off'],
        ['code' => 'MARKETING', 'description' => 'Marketing promotion'],
        ['code' => 'SERVICE_ISSUE', 'description' => 'Service quality issue']
    ]
]
```

## Writing Off Invoices

### Manual Write-Off

**Admin: Billing > Invoices > Write Off**

```php
// Write off invoice
[
    'invoice_id' => 5678,
    'amount' => 100.00,              // Full or partial
    'reason' => 'BAD_DEBT',
    'notes' => 'Collection attempts exhausted',
    'write_off_date' => '2024-05-15',
    'approved_by' => 'admin_id'
]
```

### Partial Write-Off

```php
// Write off portion of invoice
[
    'invoice_id' => 5678,
    'amount' => 50.00,               // Partial write-off
    'remaining_balance' => 50.00,
    'reason' => 'SMALL_BALANCE',
    'notes' => 'Write off small balance',
    'record_payment_plan' => false
]
```

### Bulk Write-Off

```php
// Write off multiple invoices
[
    'action' => 'bulk_write_off',
    'filter' => [
        'status' => 'Overdue',
        'age_days' => '>=90',
        'amount_max' => 10.00
    ],
    'reason' => 'SMALL_BALANCE',
    'approve_all' => true
]
```

## Write-Off Accounting

### Journal Entries

```php
// Accounting entries for write-off
[
    'date' => '2024-05-15',
    'entries' => [
        ['account' => 'Accounts Receivable', 'debit' => 0, 'credit' => 100.00],
        ['account' => 'Bad Debt Expense', 'debit' => 100.00, 'credit' => 0]
    ],
    'reference' => 'WO-2024-0123',
    'invoice_id' => 5678
]
```

### Tax Treatment

```php
// Tax implications
[
    'tax_adjustment' => true,
    'write_off_tax' => 20.00,         // Tax previously collected
    'adjust_vat_return_period' => '2024-05',
    'record_as_bad_debt_vat' => true
]
```

## Invoice Status After Write-Off

### Status Change

```php
// When written off
[
    'original_status' => 'Overdue',
    'new_status' => 'Written Off',
    'preserve_history' => true,
    'show_on_reports' => 'historical'
]
```

### Invoice Display

```
Status: Written Off
Original Amount: $100.00
Write-Off Amount: $100.00
Remaining Balance: $0.00
Written Off Date: May 15, 2024
Write-Off Reason: BAD_DEBT
```

## Client Communication

### Notification Email

```php
// Notify client of write-off
[
    'send_notification' => true,
    'template' => 'invoice_written_off',
    'include_amount' => true,
    'include_reason' => true
]
```

### Template Variables

```smarty
{$invoice_number}
{$original_amount}
{$write_off_amount}
{$reason}
{$write_off_date}
{$client_name}
{$company_name}
```

## Write-Off Recovery

### Recording Recovery

```php
// If client pays after write-off
[
    'write_off_id' => 123,
    'recovery_date' => '2024-06-01',
    'recovered_amount' => 100.00,
    'recovery_method' => 'payment',
    'reverse_journal_entries' => true
]
```

### Recovery Accounting

```php
// Reverse original write-off
[
    'entries' => [
        ['account' => 'Accounts Receivable', 'debit' => 100.00, 'credit' => 0],
        ['account' => 'Bad Debt Recovered', 'debit' => 0, 'credit' => 100.00]
    ]
]
```

## Write-Off Reports

### Bad Debt Report

**Reports > Billing > Write-Off Report**

```php
// Report structure
[
    'period' => 'May 2024',
    'total_written_off' => 5000.00,
    'total_recovered' => 500.00,
    'net_bad_debt' => 4500.00,
    'by_reason' => [
        'BAD_DEBT' => ['count' => 20, 'amount' => 3000.00],
        'SMALL_BALANCE' => ['count' => 50, 'amount' => 2000.00]
    ]
]
```

### Age Analysis

```php
// Invoices by age at write-off
[
    '30_days' => ['count' => 10, 'amount' => 1000.00],
    '60_days' => ['count' => 15, 'amount' => 2000.00],
    '90_days' => ['count' => 25, 'amount' => 5000.00],
    '120_plus_days' => ['count' => 10, 'amount' => 3000.00]
]
```

## Automation

### Auto Write-Off Rules

```php
// Automatic write-off configuration
[
    'auto_write_off_enabled' => true,
    'criteria' => [
        'invoice_age_days' => 90,
        'amount_max' => 5.00,
        'client_status' => 'inactive'
    ],
    'notify_before' => true,
    'require_approval' => true
]
```

### Scheduled Write-Offs

```bash
# Cron for auto write-off
0 2 * * * php -q /whmcs/crons/auto_writeoff.php
```

## API Functions

```php
// Write off invoice
$params = [
    'invoiceid' => 5678,
    'amount' => 100.00,
    'reason' => 'BAD_DEBT',
    'notes' => 'Collection exhausted'
];
$result = localAPI('WriteOffInvoice', $params);

// Get write-off history
$params = [
    'invoiceid' => 5678
];
$result = localAPI('GetInvoiceWriteOffs', $params);

// Reverse write-off
$params = [
    'writeoffid' => 123,
    'amount' => 100.00
];
$result = localAPI('ReverseWriteOff', $params);
```

## Hooks

```php
// Hook: PreWriteOff
add_hook('PreWriteOff', 1, function($vars) {
    // $vars['invoiceid']
    // $vars['amount']
    // $vars['reason']
    
    // Validate or modify
    return ['approve' => true];
});

// Hook: InvoiceWrittenOff
add_hook('InvoiceWrittenOff', 1, function($vars) {
    // $vars['invoiceid']
    // $vars['amount']
    // $vars['reason']
    // $vars['writeoffid']
    // $vars['written_off_at']
});
```

## Best Practices

1. **Document thoroughly**: Record all write-off reasons
2. **Get approvals**: Require authorization for write-offs
3. **Track recoveries**: Monitor payments after write-off
4. **Review policies**: Adjust criteria based on results
5. **Maintain audit trail**: Keep complete records

## Related Documentation

- [Late Fees](./whmcs-late-fees.md)
- [Credit Notes](./whmcs-credit-notes.md)
- [Invoice Generation](./whmcs-invoice-generation.md)
- [Dunning Settings](./whmcs-dunning-settings.md)