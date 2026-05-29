# WHMCS Product Performance Report Workflow

## Overview
This workflow generates product analytics and performance reports.

## Prerequisites
- WHMCS with product catalog
- Admin access for reports
- Order history

## Step-by-Step Process

### Step 1: Create Product Performance Report Generator
```php
<?php
// /includes/reports/ProductPerformanceReportGenerator.php

class ProductPerformanceReportGenerator
{
    public function generate(array $params = []): array
    {
        $startDate = $params['start_date'] ?? date('Y-m-01');
        $endDate = $params['end_date'] ?? date('Y-m-t');

        return [
            'summary' => $this->getProductSummary($startDate, $endDate),
            'top_products' => $this->getTopProducts($startDate, $endDate),
            'by_category' => $this->getByCategory($startDate, $endDate),
            'pricing_analysis' => $this->getPricingAnalysis($startDate, $endDate),
            'product_lifecycle' => $this->getProductLifecycle()
        ];
    }

    private function getProductSummary(string $startDate, string $endDate): array
    {
        return [
            'total_products' => Capsule::table('tblproducts')->count(),
            'active_products' => Capsule::table('tblproducts')->where('disabled', 0)->count(),
            'total_orders' => Capsule::table('tblorders')
                ->whereBetween('date', [$startDate, $endDate])
                ->count(),
            'total_revenue' => Capsule::table('tblinvoiceitems')
                ->join('tblinvoices', 'tblinvoiceitems.invoiceid', '=', 'tblinvoices.id')
                ->whereBetween('tblinvoices.datepaid', [$startDate, $endDate])
                ->where('tblinvoices.status', 'Paid')
                ->sum('tblinvoiceitems.amount')
        ];
    }

    private function getTopProducts(string $startDate, string $endDate): array
    {
        return Capsule::select("
            SELECT
                p.id,
                p.name,
                p.type,
                COUNT(DISTINCT oi.invoiceid) as orders,
                SUM(oi.amount) as revenue,
                AVG(oi.amount) as avg_price,
                SUM(oi.quantity) as units_sold,
                COUNT(DISTINCT o.userid) as unique_customers
            FROM tblproducts p
            LEFT JOIN tblinvoiceitems oi ON p.id = oi.relid AND oi.type = 'Hosting'
            LEFT JOIN tblinvoices o ON oi.invoiceid = o.id
                AND o.datepaid BETWEEN ? AND ?
                AND o.status = 'Paid'
            GROUP BY p.id
            HAVING orders > 0
            ORDER BY revenue DESC
            LIMIT 20
        ", [$startDate, $endDate]);
    }

    private function getByCategory(string $startDate, string $endDate): array
    {
        return Capsule::select("
            SELECT
                pg.name as category,
                COUNT(DISTINCT p.id) as products,
                COUNT(DISTINCT oi.invoiceid) as orders,
                SUM(oi.amount) as revenue
            FROM tblproductgroups pg
            LEFT JOIN tblproducts p ON pg.id = p.gid
            LEFT JOIN tblinvoiceitems oi ON p.id = oi.relid
            LEFT JOIN tblinvoices o ON oi.invoiceid = o.id
                AND o.datepaid BETWEEN ? AND ?
                AND o.status = 'Paid'
            GROUP BY pg.id
            ORDER BY revenue DESC
        ", [$startDate, $endDate]);
    }

    private function getPricingAnalysis(string $startDate, string $endDate): array
    {
        return Capsule::select("
            SELECT
                p.name,
                p.monthly as list_price,
                AVG(oi.amount / NULLIF(oi.quantity, 0)) as avg_sale_price,
                COUNT(oi.id) as transactions,
                SUM(oi.amount) as total_revenue
            FROM tblproducts p
            LEFT JOIN tblinvoiceitems oi ON p.id = oi.relid
            LEFT JOIN tblinvoices o ON oi.invoiceid = o.id
                AND o.datepaid BETWEEN ? AND ?
                AND o.status = 'Paid'
            WHERE p.monthly > 0
            GROUP BY p.id
            HAVING transactions > 0
            ORDER BY (SUM(oi.amount) / COUNT(oi.id)) / p.monthly DESC
        ", [$startDate, $endDate]);
    }

    private function getProductLifecycle(): array
    {
        return Capsule::select("
            SELECT
                p.name,
                p.created_at,
                COUNT(CASE WHEN h.regdate >= DATE_SUB(NOW(), INTERVAL 30 DAY) THEN 1 END) as new_30d,
                COUNT(CASE WHEN h.regdate >= DATE_SUB(NOW(), INTERVAL 90 DAY) THEN 1 END) as new_90d,
                COUNT(CASE WHEN h.domainstatus = 'Active' THEN 1 END) as active,
                COUNT(CASE WHEN h.domainstatus IN ('Terminated', 'Cancelled') THEN 1 END) as churned
            FROM tblproducts p
            LEFT JOIN tblhosting h ON p.id = h.packageid
            GROUP BY p.id
            ORDER BY active DESC
        ");
    }
}
```

## Related Workflows
- [WHMCS Sales Report](./whmcs-sales-report.md)
- [WHMCS Revenue Report](./whmcs-revenue-report.md)