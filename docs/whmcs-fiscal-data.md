# WHMCS Fiscal Data

## Overview

Fiscal data management in WHMCS handles financial information required for accounting compliance, tax reporting, and business analytics. This includes fiscal year settings, tax period reporting, and financial record keeping.

## Fiscal Year Configuration

### Setting Fiscal Year

**Configuration > General > Fiscal Year**

```php
// Fiscal year definition
[
    'fiscal_year_type' => 'calendar',  // calendar, tax, custom
    'start_month' => 1,                // January
    'start_day' => 1,
    'end_month' => 12,
    'end_day' => 31
]
```

### Fiscal Year Types

| Type | Description | Example |
|------|-------------|---------|
| Calendar | Standard calendar year | Jan 1 - Dec 31 |
| Tax Year | Tax reporting year | Apr 1 - Mar 31 (UK) |
| Custom | Business-defined year | Jul 1 - Jun 30 |

### Custom Fiscal Year

```php
// July - June fiscal year
[
    'fiscal_year_type' => 'custom',
    'start_month' => 7,               // July
    'start_day' => 1,
    'end_month' => 6,                 // June
    'end_day' => 30
]

// Current fiscal year
// FY 2024: Jul 1, 2023 - Jun 30, 2024
// FY 2025: Jul 1, 2024 - Jun 30, 2025
```

## Tax Period Configuration

### Tax Period Setup

```php
// Configure tax reporting periods
[
    'tax_period' => 'monthly',         // monthly, quarterly, annually
    'tax_year_basis' => 'fiscal_year', // fiscal_year, calendar_year
    'carry_forward_vat' => true,
    'vat_return_day' => 15,            // Day of month for VAT return
    'next_vat_return' => '2024-06-15'
]
```

### Quarterly Tax Periods

```php
// UK quarterly VAT periods
[
    'period_1' => ['start' => 'Jan 1', 'end' => 'Mar 31', 'return_due' => 'May 7'],
    'period_2' => ['start' => 'Apr 1', 'end' => 'Jun 30', 'return_due' => 'Aug 7'],
    'period_3' => ['start' => 'Jul 1', 'end' => 'Sep 30', 'return_due' => 'Nov 7'],
    ['start' => 'Oct 1', 'end' => 'Dec 31', 'return_due' => 'Feb 7']
]
```

## Revenue Recognition

### Accrual vs Cash Basis

```php
// Revenue recognition method
[
    'accounting_basis' => 'accrual',  // accrual, cash
    'revenue_allocation' => 'service_period',
    'deferred_revenue_account' => 'Deferred Revenue'
]

// Accrual: Revenue recognized when earned
// Cash: Revenue recognized when payment received
```

### Revenue by Period

```php
// Monthly revenue allocation
[
    'invoice_date' => '2024-05-15',
    'service_period' => 'Jun 1 - Jun 30',
    'revenue_days' => 30,
    'amount' => 30.00,
    'recognized_in_month' => 'June',
    'deferred' => 0.00
]

// Annual invoice
[
    'invoice_date' => '2024-05-15',
    'amount' => 360.00,
    'service_period' => 'May 2024 - Apr 2025',
    'recognized_this_month' => 30.00,
    'deferred' => 330.00
]
```

## Financial Reports

### Income Statement

**Reports > Financial > Income Statement**

```php
// Income statement report
[
    'period' => 'May 2024',
    'revenue' => [
        'hosting_services' => 50000.00,
        'domain_registration' => 10000.00,
        'ssl_certificates' => 5000.00,
        'other' => 2000.00
    ],
    'total_revenue' => 67000.00,
    'expenses' => [
        'cost_of_goods' => 20000.00,
        'gateway_fees' => 2000.00,
        'other_expenses' => 10000.00
    ],
    'total_expenses' => 32000.00,
    'net_income' => 35000.00
]
```

### Balance Sheet Data

```php
// Balance sheet components
[
    'assets' => [
        'cash' => 100000.00,
        'accounts_receivable' => 50000.00,
        'prepaid_expenses' => 5000.00
    ],
    'liabilities' => [
        'accounts_payable' => 20000.00,
        'deferred_revenue' => 30000.00,
        'tax_payable' => 5000.00
    ],
    'equity' => [
        'retained_earnings' => 100000.00,
        'net_income' => 35000.00
    ]
]
```

## Tax Reporting Data

### VAT/Sales Tax Report

```php
// Tax collected report
[
    'period' => 'Q1 2024',
    'taxable_sales' => [
        'standard_rate' => 50000.00,
        'reduced_rate' => 10000.00
    ],
    'tax_collected' => [
        'standard_20%' => 10000.00,
        'reduced_5%' => 500.00
    ],
    'tax_exempt_sales' => 5000.00,
    'total_output_tax' => 10500.00,
    'input_tax' => 2000.00,
    'net_tax_due' => 8500.00
]
```

### Tax by Jurisdiction

```php
// Breakdown by tax jurisdiction
[
    'US-California' => [
        'sales' => 20000.00,
        'tax_rate' => 7.5,
        'tax_collected' => 1500.00
    ],
    'US-New_York' => [
        'sales' => 15000.00,
        'tax_rate' => 8.0,
        'tax_collected' => 1200.00
    ],
    'UK' => [
        'sales' => 30000.00,
        'tax_rate' => 20,
        'tax_collected' => 6000.00
    ]
]
```

## Invoice Fiscal Data

### Invoice Level Fiscal Info

```php
// Fiscal data on invoice
[
    'invoice_id' => 5678,
    'fiscal_period' => '2024-05',
    'fiscal_year' => '2024',
    'revenue_category' => 'hosting',
    'tax_jurisdiction' => 'US-CA',
    'tax_rate' => 7.5,
    'tax_collected' => 7.50,
    'recognition_month' => 'May 2024'
]
```

### Revenue Categories

```php
// Define revenue categories
[
    'hosting' => 'Recurring Hosting Services',
    'domains' => 'Domain Registration/Transfer',
    'ssl' => 'SSL Certificates',
    'licenses' => 'Software Licenses',
    'setup' => 'One-time Setup Fees',
    'support' => 'Support Services'
]
```

## Audit Trail

### Financial Transaction Log

```php
// Complete audit trail
[
    'transaction_id' => 12345,
    'date' => '2024-05-15',
    'type' => 'invoice_payment',
    'amount' => 117.50,
    'fiscal_period' => '2024-05',
    'fiscal_year' => '2024',
    'revenue_category' => 'hosting',
    'client_id' => 123,
    'invoice_id' => 5678,
    'created_by' => 'system',
    'created_at' => '2024-05-15 10:30:00'
]
```

### Transaction Types

```php
// Financial transaction categories
[
    'invoice_created' => ['category' => 'revenue', 'type' => 'accrual'],
    'invoice_paid' => ['category' => 'cash', 'type' => 'receipt'],
    'credit_applied' => ['category' => 'adjustment', 'type' => 'reduction'],
    'refund_processed' => ['category' => 'refund', 'type' => 'outflow'],
    'write_off' => ['category' => 'bad_debt', 'type' => 'expense']
]
```

## Export and Integration

### Financial Export

```php
// Export for accounting software
[
    'format' => 'csv',                    // csv, xero, quickbooks, sage
    'date_range' => ['start' => '2024-01-01', 'end' => '2024-05-31'],
    'fiscal_year' => '2024',
    'include' => [
        'invoices' => true,
        'payments' => true,
        'refunds' => true,
        'credits' => true
    ],
    'tax_filter' => 'all'                // all, collected, paid
]
```

### Accounting Software Integration

```php
// Export to external accounting
[
    'export_type' => 'journal_entries',
    'software' => 'xero',
    'mapping' => [
        'revenue_account' => '200',
        'tax_account' => '210',
        'accounts_receivable' => '110'
    ],
    'auto_sync' => false,
    'sync_frequency' => 'daily'
]
```

## Fiscal Period Closing

### Period End Procedures

```php
// Close fiscal period
[
    'action' => 'close_period',
    'fiscal_period' => '2024-05',
    'close_date' => '2024-05-31',
    'lock_transactions' => true,
    'generate_reports' => true,
    'notify_accounting' => true
]
```

### Prevent Post-Period Edits

```php
// Lock closed periods
[
    'lock_closed_periods' => true,
    'allow_override' => true,
    'override_role' => 'finance_admin',
    'override_audit' => true
]
```

## API Functions

```php
// Get fiscal period data
$params = [
    'fiscal_year' => '2024',
    'period' => 'monthly'
];
$result = localAPI('GetFiscalData', $params);

// Get tax report
$params = [
    'start_date' => '2024-01-01',
    'end_date' => '2024-05-31',
    'tax_jurisdiction' => 'all'
];
$result = localAPI('GetTaxReport', $params);

// Export financial data
$params = [
    'format' => 'csv',
    'start_date' => '2024-01-01',
    'end_date' => '2024-05-31'
];
$result = localAPI('ExportFinancialData', $params);
```

## Best Practices

1. **Consistent fiscal settings**: Keep fiscal year configuration stable
2. **Regular reconciliation**: Match WHMCS data to accounting records
3. **Tax compliance**: Ensure proper tax calculation and reporting
4. **Audit trail**: Maintain complete transaction records
5. **Export planning**: Set up regular financial exports

## Related Documentation

- [Tax Rules](./whmcs-tax-rules.md)
- [Invoice Generation](./whmcs-invoice-generation.md)
- [Transaction Fees](./whmcs-transaction-fees.md)
- [Credit Notes](./whmcs-credit-notes.md)