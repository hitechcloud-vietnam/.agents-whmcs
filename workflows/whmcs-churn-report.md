# WHMCS Churn Report Workflow

## Overview
This workflow generates customer churn analysis reports.

## Prerequisites
- WHMCS with service tracking
- Admin access for analytics
- Historical data

## Step-by-Step Process

### Step 1: Create Churn Report Generator
```php
<?php
// /includes/reports/ChurnReportGenerator.php

class ChurnReportGenerator {
    public function generate(array $params = []): array
    {
        $startDate = $params['start_date'] ?? date('Y-m-01');
        $endDate = $params['end_date'] ?? date('Y-m-t');

        return [
            'summary' => $this->getChurnSummary($startDate, $endDate),
            'by_product' => $this->getChurnByProduct($startDate, $endDate),
            'by_reason' => $this->getChurnByReason($startDate, $endDate),
            'revenue_impact' => $this->getRevenueImpact($startDate, $endDate)
        ];
    }

    private function getChurnSummary(string $startDate, string $endDate): array
    {
        $totalAtStart = Capsule::table('tblhosting')
            ->where('regdate', '<', $startDate)
            ->count();

        $churned = Capsule::table('tblhosting')
            ->whereIn('domainstatus', ['Terminated', 'Cancelled'])
            ->whereBetween('termination_date', [$startDate, $endDate])
            ->count();

        $churnRate = $totalAtStart > 0 ? ($churned / $totalAtStart) * 100 : 0;

        return [
            'total_services_start' => $totalAtStart,
            'churned' => $churned,
            'churn_rate' => round($churnRate, 2)
        ];
    }

    private function getChurnByProduct(string $startDate, string $endDate): array
    {
        return Capsule::select("
            SELECT
                p.name as product,
                COUNT(CASE WHEN h.domainstatus IN ('Terminated', 'Cancelled')
                    AND h.termination_date BETWEEN ? AND ? THEN 1 END) as churned,
                COUNT(h.id) as total,
                COUNT(CASE WHEN h.domainstatus IN ('Terminated', 'Cancelled')
                    AND h.termination_date BETWEEN ? AND ? THEN 1 END) * 100.0 / COUNT(h.id) as churn_rate
            FROM tblproducts p
            LEFT JOIN tblhosting h ON p.id = h.packageid
            GROUP BY p.id
            ORDER BY churn_rate DESC
        ", [$startDate, $endDate, $startDate, $endDate]);
    }
}
```

## Related Workflows
- [WHMCS Retention Report](./whmcs-retention-report.md)
- [WHMCS Revenue Report](./whmcs-revenue-report.md)