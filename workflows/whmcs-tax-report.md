# WHMCS Tax Report Workflow

## Overview
This workflow generates tax liability and compliance reports.

## Prerequisites
- WHMCS with tax configuration
- Admin access for financial reports
- Tax rules properly configured

## Step-by-Step Process

### Step 1: Create Tax Report Generator
```php
<?php
// /includes/reports/TaxReportGenerator.php

class TaxReportGenerator
{
    public function generate(array $params = []): array
    {
        $startDate = $params['start_date'] ?? date('Y-m-01');
        $endDate = $params['end_date'] ?? date('Y-m-t');

        return [
            'summary' => $this->getTaxSummary($startDate, $endDate),
            'by_tax_rate' => $this->getByTaxRate($startDate, $endDate),
            'by_region' => $this->getByRegion($startDate, $endDate),
            'taxable_invoices' => $this->getTaxableInvoices($startDate, $endDate),
            'exempt_customers' => $this->getExemptCustomers($startDate, $endDate)
        ];
    }

    private function getTaxSummary(string $startDate, string $endDate): array
    {
        $data = Capsule::select("
            SELECT
                SUM(tax1 + tax2) as total_tax_collected,
                SUM(subtotal) as total_sales,
                SUM(tax1) as tax1_collected,
                SUM(tax2) as tax2_collected,
                COUNT(*) as taxable_transactions
            FROM tblinvoices
            WHERE datepaid BETWEEN ? AND ?
            AND status = 'Paid'
        ", [$startDate, $endDate]);

        return (array)$data[0];
    }

    private function getByTaxRate(string $startDate, string $endDate): array
    {
        return Capsule::select("
            SELECT
                ii.tax as tax_rate,
                COUNT(DISTINCT i.id) as invoices,
                SUM(ii.tax) as tax_collected,
                SUM(ii.amount - ii.tax) as net_sales
            FROM tblinvoiceitems ii
            JOIN tblinvoices i ON ii.invoiceid = i.id
            WHERE i.datepaid BETWEEN ? AND ?
            AND i.status = 'Paid'
            AND ii.tax > 0
            GROUP BY ii.tax
            ORDER BY tax_rate
        ", [$startDate, $endDate]);
    }

    private function getByRegion(string $startDate, string $endDate): array
    {
        return Capsule::select("
            SELECT
                c.country,
                c.state,
                SUM(ii.tax) as tax_collected,
                COUNT(DISTINCT i.id) as transactions
            FROM tblinvoices i
            JOIN tblclients c ON i.userid = c.id
            JOIN tblinvoiceitems ii ON i.id = ii.invoiceid
            WHERE i.datepaid BETWEEN ? AND ?
            AND i.status = 'Paid'
            AND ii.tax > 0
            GROUP BY c.country, c.state
            ORDER BY tax_collected DESC
        ", [$startDate, $endDate]);
    }

    private function getTaxableInvoices(string $startDate, string $endDate): array
    {
        return Capsule::select("
            SELECT
                i.id as invoice_id,
                i.datepaid,
                c.company,
                c.vatnumber,
                i.total,
                i.tax as tax_amount,
                (i.total - i.tax) as net_amount
            FROM tblinvoices i
            JOIN tblclients c ON i.userid = c.id
            WHERE i.datepaid BETWEEN ? AND ?
            AND i.status = 'Paid'
            AND i.tax > 0
            ORDER BY i.datepaid DESC
            LIMIT 100
        ", [$startDate, $endDate]);
    }

    private function getExemptCustomers(string $startDate, string $endDate): array
    {
        return Capsule::select("
            SELECT
                c.id,
                c.company,
                c.vatnumber,
                c.taxexempt,
                COUNT(i.id) as transactions,
                SUM(i.total) as total_sales
            FROM tblclients c
            LEFT JOIN tblinvoices i ON c.id = i.userid
                AND i.datepaid BETWEEN ? AND ?
                AND i.status = 'Paid'
            WHERE c.taxexempt = 1
            GROUP BY c.id
            HAVING transactions > 0
        ", [$startDate, $endDate]);
    }
}
```

### Step 2: Create Tax Report Hook
```php
<?php
// /includes/hooks/tax_report_hooks.php

add_hook('MonthlyCronJob', 1, function($vars) {
    $reportGenerator = new TaxReportGenerator();

    $report = $reportGenerator->generate([
        'start_date' => date('Y-m-01'),
        'end_date' => date('Y-m-t')
    ]);

    // Store report
    Capsule::table('mod_reports')->insert([
        'report_type' => 'tax_monthly',
        'data' => json_encode($report),
        'created_at' => date('Y-m-d H:i:s')
    ]);

    // Alert finance team
    sendEmail('admin', 'Monthly Tax Report', [
        'total_tax_collected' => formatCurrency($report['summary']['total_tax_collected']),
        'taxable_transactions' => $report['summary']['taxable_transactions'],
        'by_tax_rate' => $report['by_tax_rate']
    ]);

    return $report;
});
```

## Related Workflows
- [WHMCS Invoice Report](./whmcs-invoice-report.md)
- [WHMCS Revenue Report](./whmcs-revenue-report.md)