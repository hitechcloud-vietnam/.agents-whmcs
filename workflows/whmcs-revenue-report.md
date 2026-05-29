# WHMCS Revenue Report Workflow

## Overview
This workflow generates and automates revenue reporting for WHMCS.

## Prerequisites
- WHMCS with database access
- Admin access for reports
- Email configuration for report delivery

## Step-by-Step Process

### Step 1: Create Revenue Report Generator
```php
<?php
// /includes/reports/RevenueReportGenerator.php

class RevenueReportGenerator
{
    /**
     * Generate revenue report
     */
    public function generate(array $params = []): array
    {
        $startDate = $params['start_date'] ?? date('Y-m-01');
        $endDate = $params['end_date'] ?? date('Y-m-t');
        $groupBy = $params['group_by'] ?? 'day';

        return [
            'summary' => $this->getSummary($startDate, $endDate),
            'daily' => $this->getDailyBreakdown($startDate, $endDate),
            'by_payment_method' => $this->getByPaymentMethod($startDate, $endDate),
            'by_product' => $this->getByProduct($startDate, $endDate),
            'trends' => $this->getTrends($startDate, $endDate),
            'period' => ['start' => $startDate, 'end' => $endDate]
        ];
    }

    private function getSummary(string $startDate, string $endDate): array
    {
        $data = Capsule::select("
            SELECT
                COUNT(*) as total_transactions,
                SUM(total) as gross_revenue,
                SUM(total - (total * 0.029 + 0.30)) as net_revenue,
                AVG(total) as average_transaction,
                MIN(total) as smallest_transaction,
                MAX(total) as largest_transaction
            FROM tblinvoices
            WHERE datepaid BETWEEN ? AND ?
            AND status = 'Paid'
        ", [$startDate, $endDate]);

        return (array)$data[0];
    }

    private function getDailyBreakdown(string $startDate, string $endDate): array
    {
        return Capsule::select("
            SELECT
                DATE(datepaid) as date,
                COUNT(*) as transactions,
                SUM(total) as revenue,
                AVG(total) as avg_transaction
            FROM tblinvoices
            WHERE datepaid BETWEEN ? AND ?
            AND status = 'Paid'
            GROUP BY DATE(datepaid)
            ORDER BY date DESC
        ", [$startDate, $endDate]);
    }

    private function getByPaymentMethod(string $startDate, string $endDate): array
    {
        return Capsule::select("
            SELECT
                paymentmethod,
                COUNT(*) as transactions,
                SUM(total) as revenue,
                AVG(total) as avg_transaction
            FROM tblinvoices
            WHERE datepaid BETWEEN ? AND ?
            AND status = 'Paid'
            GROUP BY paymentmethod
            ORDER BY revenue DESC
        ", [$startDate, $endDate]);
    }

    private function getByProduct(string $startDate, string $endDate): array
    {
        return Capsule::select("
            SELECT
                p.name as product_name,
                COUNT(DISTINCT i.userid) as unique_customers,
                COUNT(*) as transactions,
                SUM(ii.amount) as revenue
            FROM tblinvoices i
            JOIN tblinvoiceitems ii ON i.id = ii.invoiceid
            JOIN tblproducts p ON ii.relid = p.id
            WHERE i.datepaid BETWEEN ? AND ?
            AND i.status = 'Paid'
            AND ii.type = 'Hosting'
            GROUP BY p.id
            ORDER BY revenue DESC
        ", [$startDate, $endDate]);
    }

    private function getTrends(string $startDate, string $endDate): array
    {
        $currentPeriod = Capsule::select("
            SELECT SUM(total) as revenue
            FROM tblinvoices
            WHERE datepaid BETWEEN ? AND ?
            AND status = 'Paid'
        ", [$startDate, $endDate]);

        $periodDays = (strtotime($endDate) - strtotime($startDate)) / 86400;
        $previousStart = date('Y-m-d', strtotime("-{$periodDays} days", strtotime($startDate)));
        $previousEnd = date('Y-m-d', strtotime('-1 day', strtotime($startDate)));

        $previousPeriod = Capsule::select("
            SELECT SUM(total) as revenue
            FROM tblinvoices
            WHERE datepaid BETWEEN ? AND ?
            AND status = 'Paid'
        ", [$previousStart, $previousEnd]);

        $currentRevenue = $currentPeriod[0]->revenue ?? 0;
        $previousRevenue = $previousPeriod[0]->revenue ?? 0;
        $growth = $previousRevenue > 0 ? (($currentRevenue - $previousRevenue) / $previousRevenue) * 100 : 0;

        return [
            'current_period' => $currentRevenue,
            'previous_period' => $previousRevenue,
            'growth_percentage' => round($growth, 2)
        ];
    }
}
```

### Step 2: Create Scheduled Report
```php
<?php
// /includes/hooks/revenue_report_hooks.php

add_hook('MonthlyCronJob', 1, function($vars) {
    $reportGenerator = new RevenueReportGenerator();

    $report = $reportGenerator->generate([
        'start_date' => date('Y-m-01'),
        'end_date' => date('Y-m-t'),
        'group_by' => 'day'
    ]);

    // Send to admin
    sendEmail('admin', 'Monthly Revenue Report', [
        'summary' => $report['summary'],
        'trends' => $report['trends'],
        'period' => $report['period']
    ]);

    // Store for historical reference
    Capsule::table('mod_reports')->insert([
        'report_type' => 'revenue_monthly',
        'period_start' => $report['period']['start'],
        'period_end' => $report['period']['end'],
        'data' => json_encode($report),
        'created_at' => date('Y-m-d H:i:s')
    ]);

    return $report;
});
```

## Related Workflows
- [WHMCS Report Automation](./whmcs-report-automation.md)
- [WHMCS ARR Report](./whmcs-arr-report.md)