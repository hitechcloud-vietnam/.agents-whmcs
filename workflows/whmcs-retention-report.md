# WHMCS Retention Report Workflow

## Overview
This workflow generates customer retention and churn metrics reports.

## Prerequisites
- WHMCS with service tracking
- Admin access for analytics
- Historical data

## Step-by-Step Process

### Step 1: Create Retention Report Generator
```php
<?php
// /includes/reports/RetentionReportGenerator.php

class RetentionReportGenerator
{
    public function generate(array $params = []): array
    {
        $startDate = $params['start_date'] ?? date('Y-m-01');
        $endDate = $params['end_date'] ?? date('Y-m-t');

        return [
            'summary' => $this->getRetentionSummary($startDate, $endDate),
            'cohort_retention' => $this->getCohortRetention(),
            'churn_analysis' => $this->getChurnAnalysis($startDate, $endDate),
            'renewal_rates' => $this->getRenewalRates(),
            'at_risk_customers' => $this->getAtRiskCustomers()
        ];
    }

    private function getRetentionSummary(string $startDate, string $endDate): array
    {
        $periodStart = Capsule::table('tblhosting')
            ->where('regdate', '<', $startDate)
            ->where('domainstatus', 'Active')
            ->count();

        $periodEnd = Capsule::table('tblhosting')
            ->where('regdate', '<=', $endDate)
            ->where('domainstatus', 'Active')
            ->count();

        $churned = Capsule::table('tblhosting')
            ->whereIn('domainstatus', ['Terminated', 'Cancelled'])
            ->whereBetween('termination_date', [$startDate, $endDate])
            ->count();

        $retained = $periodStart - $churned;
        $retentionRate = $periodStart > 0 ? ($retained / $periodStart) * 100 : 0;
        $churnRate = $periodStart > 0 ? ($churned / $periodStart) * 100 : 0;

        return [
            'period_start_services' => $periodStart,
            'period_end_services' => $periodEnd,
            'retained' => $retained,
            'churned' => $churned,
            'retention_rate' => round($retentionRate, 1),
            'churn_rate' => round($churnRate, 1)
        ];
    }

    private function getCohortRetention(): array
    {
        return Capsule::select("
            SELECT
                DATE_FORMAT(regdate, '%Y-%m') as cohort,
                COUNT(*) as initial,
                SUM(CASE WHEN months_active >= 1 THEN 1 ELSE 0 END) as m1,
                SUM(CASE WHEN months_active >= 3 THEN 1 ELSE 0 END) as m3,
                SUM(CASE WHEN months_active >= 6 THEN 1 ELSE 0 END) as m6,
                SUM(CASE WHEN months_active >= 12 THEN 1 ELSE 0 END) as m12
            FROM (
                SELECT
                    h.regdate,
                    FLOOR(DATEDIFF(NOW(), h.regdate) / 30) as months_active
                FROM tblhosting h
                WHERE h.domainstatus = 'Active'
            ) services
            GROUP BY cohort
            ORDER BY cohort DESC
            LIMIT 12
        ");
    }

    private function getChurnAnalysis(string $startDate, string $endDate): array
    {
        return Capsule::select("
            SELECT
                p.name as product_name,
                COUNT(CASE WHEN h.domainstatus = 'Terminated' THEN 1 END) as churned,
                COUNT(CASE WHEN h.domainstatus = 'Active' THEN 1 END) as active,
                COUNT(CASE WHEN h.domainstatus = 'Terminated' THEN 1 END) * 100.0 / NULLIF(COUNT(*), 0) as churn_rate
            FROM tblproducts p
            LEFT JOIN tblhosting h ON p.id = h.packageid
            GROUP BY p.id
            HAVING churned > 0 OR active > 0
            ORDER BY churn_rate DESC
        ");
    }

    private function getRenewalRates(): array
    {
        $lastMonth = date('Y-m-d', strtotime('-30 days'));
        $twoMonthsAgo = date('Y-m-d', strtotime('-60 days'));

        return Capsule::select("
            SELECT
                DATE_FORMAT(nextduedate, '%Y-%m') as month,
                COUNT(*) as due,
                COUNT(CASE WHEN domainstatus = 'Active' THEN 1 END) as renewed,
                COUNT(CASE WHEN domainstatus IN ('Terminated', 'Cancelled') THEN 1 END) as churned
            FROM tblhosting
            WHERE nextduedate BETWEEN ? AND ?
            GROUP BY month
            ORDER BY month
        ", [$twoMonthsAgo, $lastMonth]);
    }

    private function getAtRiskCustomers(): array
    {
        return Capsule::select("
            SELECT
                c.id,
                c.firstname,
                c.lastname,
                c.email,
                c.lastlogin,
                COUNT(h.id) as services,
                SUM(h.amount) as mrr,
                DATEDIFF(NOW(), c.lastlogin) as days_inactive,
                CASE
                    WHEN DATEDIFF(NOW(), c.lastlogin) > 180 THEN 'high_risk'
                    WHEN DATEDIFF(NOW(), c.lastlogin) > 90 THEN 'medium_risk'
                    ELSE 'low_risk'
                END as risk_level
            FROM tblclients c
            JOIN tblhosting h ON c.id = h.userid
            WHERE h.domainstatus = 'Active'
            GROUP BY c.id
            HAVING days_inactive > 60
            ORDER BY days_inactive DESC
            LIMIT 50
        ");
    }
}
```

## Key Metrics

| Metric | Formula | Target |
|--------|---------|--------|
| Retention Rate | (Retained / Start) x 100 | > 90% |
| Churn Rate | (Churned / Start) x 100 | < 5% |
| Renewal Rate | (Renewed / Due) x 100 | > 85% |

## Related Workflows
- [WHMCS Churn Report](./whmcs-churn-report.md)
- [WHMCS CLV Report](./whmcs-clv-report.md)