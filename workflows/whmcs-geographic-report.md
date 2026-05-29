# WHMCS Geographic Report Workflow

## Overview
This workflow generates geographic distribution and location-based analytics reports.

## Prerequisites
- WHMCS with client data
- Admin access for analytics
- Country/state data populated

## Step-by-Step Process

### Step 1: Create Geographic Report Generator
```php
<?php
// /includes/reports/GeographicReportGenerator.php

class GeographicReportGenerator
{
    public function generate(array $params = []): array
    {
        $startDate = $params['start_date'] ?? date('Y-m-01');
        $endDate = $params['end_date'] ?? date('Y-m-t');

        return [
            'by_country' => $this->getByCountry($startDate, $endDate),
            'by_state' => $this->getByState($startDate, $endDate),
            'revenue_by_region' => $this->getRevenueByRegion($startDate, $endDate),
            'service_distribution' => $this->getServiceDistributionByRegion(),
            'growth_by_region' => $this->getGrowthByRegion($startDate, $endDate)
        ];
    }

    private function getByCountry(string $startDate, string $endDate): array
    {
        return Capsule::select("
            SELECT
                c.country,
                COUNT(DISTINCT c.id) as clients,
                COUNT(DISTINCT h.id) as services,
                SUM(h.amount) as mrr,
                SUM(i.total) as revenue
            FROM tblclients c
            LEFT JOIN tblhosting h ON c.id = h.userid AND h.domainstatus = 'Active'
            LEFT JOIN tblinvoices i ON c.id = i.userid AND i.status = 'Paid'
                AND i.datepaid BETWEEN ? AND ?
            GROUP BY c.country
            ORDER BY clients DESC
        ", [$startDate, $endDate]);
    }

    private function getByState(string $startDate, string $endDate): array
    {
        return Capsule::select("
            SELECT
                c.country,
                c.state,
                COUNT(DISTINCT c.id) as clients,
                SUM(i.total) as revenue
            FROM tblclients c
            LEFT JOIN tblinvoices i ON c.id = i.userid AND i.status = 'Paid'
                AND i.datepaid BETWEEN ? AND ?
            WHERE c.country = 'US'
            GROUP BY c.state
            ORDER BY clients DESC
            LIMIT 50
        ", [$startDate, $endDate]);
    }

    private function getRevenueByRegion(string $startDate, string $endDate): array
    {
        return Capsule::select("
            SELECT
                CASE
                    WHEN c.country IN ('US', 'CA') THEN 'North America'
                    WHEN c.country IN ('GB', 'DE', 'FR', 'NL', 'IT', 'ES') THEN 'Europe'
                    WHEN c.country IN ('AU', 'NZ') THEN 'Oceania'
                    WHEN c.country IN ('BR', 'MX', 'AR') THEN 'Latin America'
                    ELSE 'Other'
                END as region,
                COUNT(DISTINCT c.id) as clients,
                SUM(i.total) as revenue,
                AVG(i.total) as avg_order_value
            FROM tblclients c
            JOIN tblinvoices i ON c.id = i.userid AND i.status = 'Paid'
                AND i.datepaid BETWEEN ? AND ?
            GROUP BY region
            ORDER BY revenue DESC
        ", [$startDate, $endDate]);
    }

    private function getServiceDistributionByRegion(): array
    {
        return Capsule::select("
            SELECT
                p.name as product,
                c.country,
                COUNT(h.id) as services
            FROM tblhosting h
            JOIN tblclients c ON h.userid = c.id
            JOIN tblproducts p ON h.packageid = p.id
            WHERE h.domainstatus = 'Active'
            GROUP BY p.name, c.country
            ORDER BY services DESC
            LIMIT 100
        ");
    }

    private function getGrowthByRegion(string $startDate, string $endDate): array
    {
        return Capsule::select("
            SELECT
                c.country,
                COUNT(DISTINCT CASE WHEN c.datecreated BETWEEN ? AND ? THEN c.id END) as new_clients,
                SUM(CASE WHEN i.datepaid BETWEEN ? AND ? THEN i.total ELSE 0 END) as revenue
            FROM tblclients c
            LEFT JOIN tblinvoices i ON c.id = i.userid AND i.status = 'Paid'
            GROUP BY c.country
            ORDER BY new_clients DESC
            LIMIT 20
        ", [$startDate, $endDate, $startDate, $endDate]);
    }
}
```

## Related Workflows
- [WHMCS Revenue Report](./whmcs-revenue-report.md)
- [WHMCS Client Report](./whmcs-client-report.md)