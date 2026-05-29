# WHMCS Invoice Report Workflow

## Overview
This workflow generates invoice and billing reports.

## Step-by-Step Process

```php
<?php
// /includes/reports/InvoiceReportGenerator.php

class InvoiceReportGenerator
{
    public function generate(array $params = []): array
    {
        $startDate = $params['start_date'] ?? date('Y-m-01');
        $endDate = $params['end_date'] ?? date('Y-m-t');

        return [
            'summary' => $this->getSummary($startDate, $endDate),
            'by_status' => $this->getByStatus($startDate, $endDate),
            'aging' => $this->getAgingReport(),
            'outstanding' => $this->getOutstandingAmounts()
        ];
    }

    private function getSummary(string $startDate, string $endDate): array
    {
        $data = Capsule::select("
            SELECT
                COUNT(*) as total_invoices,
                SUM(total) as total_amount,
                SUM(CASE WHEN status = 'Paid' THEN total ELSE 0 END) as paid_amount,
                SUM(CASE WHEN status = 'Unpaid' THEN total ELSE 0 END) as unpaid_amount,
                AVG(total) as avg_invoice
            FROM tblinvoices
            WHERE date BETWEEN ? AND ?
        ", [$startDate, $endDate]);

        return (array)$data[0];
    }

    private function getByStatus(string $startDate, string $endDate): array
    {
        return Capsule::select("
            SELECT
                status,
                COUNT(*) as count,
                SUM(total) as amount
            FROM tblinvoices
            WHERE date BETWEEN ? AND ?
            GROUP BY status
        ", [$startDate, $endDate]);
    }

    private function getAgingReport(): array
    {
        return Capsule::select("
            SELECT
                CASE
                    WHEN DATEDIFF(CURDATE(), duedate) <= 0 THEN 'not_yet_due'
                    WHEN DATEDIFF(CURDATE(), duedate) <= 30 THEN '1_30_days'
                    WHEN DATEDIFF(CURDATE(), duedate) <= 60 THEN '31_60_days'
                    WHEN DATEDIFF(CURDATE(), duedate) <= 90 THEN '61_90_days'
                    ELSE 'over_90_days'
                END as aging_bucket,
                COUNT(*) as count,
                SUM(total) as amount
            FROM tblinvoices
            WHERE status IN ('Unpaid', 'Overdue')
            GROUP BY aging_bucket
        ");
    }

    private function getOutstandingAmounts(): array
    {
        return [
            'total_outstanding' => Capsule::table('tblinvoices')
                ->whereIn('status', ['Unpaid', 'Overdue'])
                ->sum('total'),
            'overdue_count' => Capsule::table('tblinvoices')
                ->where('status', 'Overdue')
                ->count(),
            'overdue_amount' => Capsule::table('tblinvoices')
                ->where('status', 'Overdue')
                ->sum('total')
        ];
    }
}
```

## Related Workflows
- [WHMCS Payment Report](./whmcs-payment-report.md)
- [WHMCS Revenue Report](./whmcs-revenue-report.md)