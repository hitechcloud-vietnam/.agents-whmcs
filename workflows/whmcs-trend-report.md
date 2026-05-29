# WHMCS Trend Report Workflow

## Overview
This workflow generates trend analysis reports for business metrics.

## Prerequisites
- WHMCS with historical data
- Admin access for analytics
- Trend reporting module

## Step-by-Step Process

### Step 1: Create Trend Report Generator
```php
<?php
// /includes/reports/TrendReportGenerator.php

class TrendReportGenerator
{
    public function generate(array $params = []): array
    {
        $months = $params['months'] ?? 12;

        return [
            'revenue_trends' => $this->getRevenueTrends($months),
            'client_trends' => $this->getClientTrends($months),
            'service_trends' => $this->getServiceTrends($months),
            'churn_trends' => $this->getChurnTrends($months),
            'comparative_analysis' => $this->getComparativeAnalysis($months)
        ];
    }

    private function getRevenueTrends(int $months): array
    {
        return Capsule::select("
            SELECT
                DATE_FORMAT(datepaid, '%Y-%m') as month,
                SUM(total) as revenue,
                COUNT(DISTINCT userid) as paying_customers,
                AVG(total) as avg_order_value,
                SUM(total) - LAG(SUM(total)) OVER (ORDER BY DATE_FORMAT(datepaid, '%Y-%m')) as mom_change
            FROM tblinvoices
            WHERE status = 'Paid'
            AND datepaid >= DATE_SUB(NOW(), INTERVAL ? MONTH)
            GROUP BY DATE_FORMAT(datepaid, '%Y-%m')
            ORDER BY month
        ", [$months]);
    }

    private function getClientTrends(int $months): array
    {
        return Capsule::select("
            SELECT
                DATE_FORMAT(datecreated, '%Y-%m') as month,
                COUNT(*) as new_clients,
                COUNT(CASE WHEN lastlogin >= DATE_SUB(NOW(), INTERVAL 30 DAY) THEN 1 END) as active_clients
            FROM tblclients
            WHERE datecreated >= DATE_SUB(NOW(), INTERVAL ? MONTH)
            GROUP BY DATE_FORMAT(datecreated, '%Y-%m')
            ORDER BY month
        ", [$months]);
    }

    private function getServiceTrends(int $months): array
    {
        return Capsule::select("
            SELECT
                DATE_FORMAT(regdate, '%Y-%m') as month,
                COUNT(*) as new_services,
                SUM(CASE WHEN domainstatus = 'Active' THEN 1 ELSE 0 END) as active,
                SUM(CASE WHEN domainstatus IN ('Suspended', 'Terminated') THEN 1 ELSE 0 END) as inactive
            FROM tblhosting
            WHERE regdate >= DATE_SUB(NOW(), INTERVAL ? MONTH)
            GROUP BY DATE_FORMAT(regdate, '%Y-%m')
            ORDER BY month
        ", [$months]);
    }

    private function getChurnTrends(int $months): array
    {
        return Capsule::select("
            SELECT
                DATE_FORMAT(termination_date, '%Y-%m') as month,
                COUNT(*) as churned_services,
                SUM(amount) as churned_mrr
            FROM tblhosting
            WHERE domainstatus IN ('Terminated', 'Cancelled')
            AND termination_date >= DATE_SUB(NOW(), INTERVAL ? MONTH)
            GROUP BY DATE_FORMAT(termination_date, '%Y-%m')
            ORDER BY month
        ", [$months]);
    }

    private function getComparativeAnalysis(int $months): array
    {
        $currentPeriod = Capsule::select("
            SELECT
                SUM(total) as revenue,
                COUNT(DISTINCT userid) as customers
            FROM tblinvoices
            WHERE status = 'Paid'
            AND datepaid >= DATE_SUB(NOW(), INTERVAL ? MONTH)
        ", [$months])[0];

        $previousPeriod = Capsule::select("
            SELECT
                SUM(total) as revenue,
                COUNT(DISTINCT userid) as customers
            FROM tblinvoices
            WHERE status = 'Paid'
            AND datepaid >= DATE_SUB(NOW(), INTERVAL ? MONTH)
            AND datepaid < DATE_SUB(NOW(), INTERVAL ? MONTH)
        ", [$months, $months])[0];

        return [
            'current' => (array)$currentPeriod,
            'previous' => (array)$previousPeriod,
            'revenue_growth' => $previousPeriod->revenue > 0
                ? round((($currentPeriod->revenue - $previousPeriod->revenue) / $previousPeriod->revenue) * 100, 1)
                : 0,
            'customer_growth' => $previousPeriod->customers > 0
                ? round((($currentPeriod->customers - $previousPeriod->customers) / $previousPeriod->customers) * 100, 1)
                : 0
        ];
    }
}
```

## Related Workflows
- [WHMCS Revenue Report](./whmcs-revenue-report.md)
- [WHMCS Forecast Report](./whmcs-forecast-report.md)