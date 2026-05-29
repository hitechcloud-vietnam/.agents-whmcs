# WHMCS Sales Report Workflow

## Overview
This workflow generates comprehensive sales analytics reports for WHMCS.

## Step-by-Step Process

### Step 1: Create Sales Report Generator
```php
<?php
// /includes/reports/SalesReportGenerator.php

class SalesReportGenerator
{
    public function generate(array $params = []): array
    {
        $startDate = $params['start_date'] ?? date('Y-m-01');
        $endDate = $params['end_date'] ?? date('Y-m-t');

        return [
            'new_orders' => $this->getNewOrders($startDate, $endDate),
            'order_status' => $this->getOrderStatusBreakdown($startDate, $endDate),
            'top_products' => $this->getTopProducts($startDate, $endDate),
            'conversion' => $this->getConversionMetrics($startDate, $endDate),
            'abandoned_carts' => $this->getAbandonedCarts($startDate, $endDate)
        ];
    }

    private function getNewOrders(string $startDate, string $endDate): array
    {
        return Capsule::select("
            SELECT
                DATE(date) as date,
                COUNT(*) as orders,
                SUM(totaldue) as revenue,
                COUNT(DISTINCT userid) as unique_customers
            FROM tblorders
            WHERE date BETWEEN ? AND ?
            GROUP BY DATE(date)
            ORDER BY date DESC
        ", [$startDate, $endDate]);
    }

    private function getOrderStatusBreakdown(string $startDate, string $endDate): array
    {
        return Capsule::select("
            SELECT
                status,
                COUNT(*) as count,
                SUM(totaldue) as revenue
            FROM tblorders
            WHERE date BETWEEN ? AND ?
            GROUP BY status
        ", [$startDate, $endDate]);
    }

    private function getTopProducts(string $startDate, string $endDate): array
    {
        return Capsule::select("
            SELECT
                p.name,
                COUNT(o.id) as orders,
                SUM(o.totaldue) as revenue
            FROM tblorders o
            JOIN tblproducts p ON o.products = p.id
            WHERE o.date BETWEEN ? AND ?
            AND o.status IN ('Active', 'Pending')
            GROUP BY p.id
            ORDER BY revenue DESC
            LIMIT 10
        ", [$startDate, $endDate]);
    }

    private function getConversionMetrics(string $startDate, string $endDate): array
    {
        $carts = Capsule::table('tblcart')
            ->where('created_at', 'BETWEEN', [$startDate, $endDate])
            ->count();

        $completed = Capsule::table('tblorders')
            ->where('date', 'BETWEEN', [$startDate, $endDate])
            ->count();

        $abandoned = $carts - $completed;
        $conversionRate = $carts > 0 ? ($completed / $carts) * 100 : 0;

        return [
            'carts_created' => $carts,
            'orders_completed' => $completed,
            'abandoned_carts' => $abandoned,
            'conversion_rate' => round($conversionRate, 2)
        ];
    }
}
```

### Step 2: Create Scheduled Report Hook
```php
<?php
// /includes/hooks/sales_report_hooks.php

add_hook('DailyCronJob', 1, function($vars) {
    $reportGenerator = new SalesReportGenerator();

    $report = $reportGenerator->generate([
        'start_date' => date('Y-m-d', strtotime('-7 days')),
        'end_date' => date('Y-m-d')
    ]);

    // Store report
    Capsule::table('mod_reports')->insert([
        'report_type' => 'sales_weekly',
        'data' => json_encode($report),
        'created_at' => date('Y-m-d H:i:s')
    ]);

    return $report;
});
```

## Related Workflows
- [WHMCS Revenue Report](./whmcs-revenue-report.md)
- [WHMCS Report Automation](./whmcs-report-automation.md)