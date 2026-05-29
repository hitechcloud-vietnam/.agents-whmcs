# WHMCS Conversion Report Workflow

## Overview
This workflow generates conversion funnel and analytics reports.

## Prerequisites
- WHMCS with analytics tracking
- Admin access for reports
- Cart/order data

## Step-by-Step Process

### Step 1: Create Conversion Report Generator
```php
<?php
// /includes/reports/ConversionReportGenerator.php

class ConversionReportGenerator
{
    public function generate(array $params = []): array
    {
        $startDate = $params['start_date'] ?? date('Y-m-01');
        $endDate = $params['end_date'] ?? date('Y-m-t');

        return [
            'funnel' => $this->getConversionFunnel($startDate, $endDate),
            'abandonment' => $this->getAbandonmentAnalysis($startDate, $endDate),
            'by_product' => $this->getConversionByProduct($startDate, $endDate),
            'by_source' => $this->getConversionBySource($startDate, $endDate),
            'trends' => $this->getConversionTrends($startDate, $endDate)
        ];
    }

    private function getConversionFunnel(string $startDate, string $endDate): array
    {
        // Visitors (from analytics)
        $visitors = Capsule::table('mod_analytics')
            ->whereBetween('date', [$startDate, $endDate])
            ->sum('visitors');

        // Add to cart
        $addToCart = Capsule::table('tblcart')
            ->whereBetween('created_at', [$startDate, $endDate])
            ->count();

        // Checkout started
        $checkoutStarted = Capsule::table('tblorders')
            ->whereBetween('date', [$startDate, $endDate])
            ->count();

        // Completed orders
        $completed = Capsule::table('tblorders')
            ->whereBetween('date', [$startDate, $endDate])
            ->whereIn('status', ['Active', 'Pending'])
            ->count();

        return [
            'visitors' => $visitors,
            'add_to_cart' => $addToCart,
            'checkout_started' => $checkoutStarted,
            'completed' => $completed,
            'cart_to_checkout' => $addToCart > 0 ? round(($checkoutStarted / $addToCart) * 100, 1) : 0,
            'checkout_to_order' => $checkoutStarted > 0 ? round(($completed / $checkoutStarted) * 100, 1) : 0,
            'overall_conversion' => $visitors > 0 ? round(($completed / $visitors) * 100, 1) : 0
        ];
    }

    private function getAbandonmentAnalysis(string $startDate, string $endDate): array
    {
        return Capsule::select("
            SELECT
                DATE(created_at) as date,
                COUNT(DISTINCT id) as abandoned_carts,
                SUM(subtotal) as abandoned_value
            FROM tblcart
            WHERE created_at BETWEEN ? AND ?
            AND status = 'abandoned'
            GROUP BY date
            ORDER BY date DESC
        ", [$startDate, $endDate]);
    }

    private function getConversionByProduct(string $startDate, string $endDate): array
    {
        return Capsule::select("
            SELECT
                p.name as product_name,
                COUNT(DISTINCT o.id) as views,
                COUNT(DISTINCT CASE WHEN o.status IN ('Active', 'Pending') THEN o.id END) as purchases,
                COUNT(DISTINCT CASE WHEN o.status IN ('Active', 'Pending') THEN o.id END) * 100.0 / NULLIF(COUNT(DISTINCT o.id), 0) as conversion_rate
            FROM tblproducts p
            LEFT JOIN tblorders o ON p.id = o.products
                AND o.date BETWEEN ? AND ?
            GROUP BY p.id
            HAVING views > 0
            ORDER BY conversion_rate DESC
        ", [$startDate, $endDate]);
    }

    private function getConversionBySource(string $startDate, string $endDate): array
    {
        return Capsule::select("
            SELECT
                referrer,
                COUNT(*) as visits,
                COUNT(DISTINCT o.id) as conversions,
                COUNT(DISTINCT o.id) * 100.0 / NULLIF(COUNT(*), 0) as conversion_rate
            FROM mod_page_views pv
            LEFT JOIN tblorders o ON pv.session_id = o.session_id
                AND o.date BETWEEN ? AND ?
            WHERE pv.date BETWEEN ? AND ?
            GROUP BY referrer
            HAVING visits > 10
            ORDER BY conversion_rate DESC
        ", [$startDate, $endDate, $startDate, $endDate]);
    }

    private function getConversionTrends(string $startDate, string $endDate): array
    {
        return Capsule::select("
            SELECT
                DATE(date) as date,
                SUM(visitors) as visitors,
                COUNT(DISTINCT CASE WHEN status IN ('Active', 'Pending') THEN id END) as orders,
                COUNT(DISTINCT CASE WHEN status IN ('Active', 'Pending') THEN id END) * 100.0 / NULLIF(SUM(visitors), 0) as conversion_rate
            FROM (
                SELECT o.id, o.date, o.status, a.visitors
                FROM tblorders o
                LEFT JOIN mod_analytics a ON DATE(a.date) = DATE(o.date)
                WHERE o.date BETWEEN ? AND ?
            ) combined
            GROUP BY date
            ORDER BY date DESC
        ", [$startDate, $endDate]);
    }
}
```

## Related Workflows
- [WHMCS Sales Report](./whmcs-sales-report.md)
- [WHMCS Report Automation](./whmcs-report-automation.md)