# WHMCS Cohort Report Workflow

## Overview
This workflow generates customer cohort analysis reports.

## Prerequisites
- WHMCS with client history
- Admin access for analytics
- Cohort analysis capabilities

## Step-by-Step Process

### Step 1: Create Cohort Report Generator
```php
<?php
// /includes/reports/CohortReportGenerator.php

class CohortReportGenerator
{
    public function generate(array $params = []): array
    {
        $cohortMonths = $params['cohort_months'] ?? 12;

        return [
            'monthly_cohorts' => $this->getMonthlyCohorts($cohortMonths),
            'revenue_retention' => $this->getRevenueRetention(),
            'cohort_comparison' => $this->getCohortComparison(),
            'engagement_by_cohort' => $this->getEngagementByCohort()
        ];
    }

    private function getMonthlyCohorts(int $months): array
    {
        return Capsule::select("
            SELECT
                DATE_FORMAT(c.datecreated, '%Y-%m') as cohort,
                COUNT(DISTINCT c.id) as cohort_size,
                -- Month 0 retention
                COUNT(DISTINCT CASE WHEN MONTHS_DIFF(h.regdate, c.datecreated) = 0 THEN c.id END) as m0,
                -- Month 1 retention
                COUNT(DISTINCT CASE WHEN MONTHS_DIFF(h.regdate, c.datecreated) <= 1 AND h.domainstatus = 'Active' THEN c.id END) as m1,
                -- Month 3 retention
                COUNT(DISTINCT CASE WHEN MONTHS_DIFF(h.regdate, c.datecreated) <= 3 AND h.domainstatus = 'Active' THEN c.id END) as m3,
                -- Month 6 retention
                COUNT(DISTINCT CASE WHEN MONTHS_DIFF(h.regdate, c.datecreated) <= 6 AND h.domainstatus = 'Active' THEN c.id END) as m6,
                -- Month 12 retention
                COUNT(DISTINCT CASE WHEN MONTHS_DIFF(h.regdate, c.datecreated) <= 12 AND h.domainstatus = 'Active' THEN c.id END) as m12
            FROM tblclients c
            LEFT JOIN tblhosting h ON c.id = h.userid
            WHERE c.datecreated >= DATE_SUB(NOW(), INTERVAL ? MONTH)
            GROUP BY DATE_FORMAT(c.datecreated, '%Y-%m')
            ORDER BY cohort
        ", [$months]);
    }

    private function getRevenueRetention(): array
    {
        return Capsule::select("
            SELECT
                DATE_FORMAT(c.datecreated, '%Y-%m') as cohort,
                DATE_FORMAT(i.datepaid, '%Y-%m') as revenue_month,
                SUM(i.total) as revenue,
                SUM(i.total) / COUNT(DISTINCT c.id) as revenue_per_customer
            FROM tblclients c
            JOIN tblinvoices i ON c.id = i.userid AND i.status = 'Paid'
            WHERE c.datecreated >= DATE_SUB(NOW(), INTERVAL 12 MONTH)
            GROUP BY cohort, revenue_month
            ORDER BY cohort, revenue_month
        ");
    }

    private function getCohortComparison(): array
    {
        return Capsule::select("
            SELECT
                CASE
                    WHEN c.datecreated >= DATE_SUB(NOW(), INTERVAL 3 MONTH) THEN 'Recent (3mo)'
                    WHEN c.datecreated >= DATE_SUB(NOW(), INTERVAL 6 MONTH) THEN 'Medium (3-6mo)'
                    ELSE 'Mature (6mo+)'
                END as cohort_group,
                COUNT(DISTINCT c.id) as customers,
                AVG(revenue.total_revenue) as avg_ltv,
                SUM(revenue.total_revenue) as total_revenue
            FROM tblclients c
            JOIN (
                SELECT userid, SUM(total) as total_revenue
                FROM tblinvoices WHERE status = 'Paid' GROUP BY userid
            ) revenue ON c.id = revenue.userid
            GROUP BY cohort_group
        ");
    }

    private function getEngagementByCohort(): array
    {
        return Capsule::select("
            SELECT
                DATE_FORMAT(datecreated, '%Y-%m') as cohort,
                COUNT(*) as clients,
                AVG(DATEDIFF(COALESCE(lastlogin, NOW()), datecreated)) as avg_days_active,
                MAX(COALESCE(lastlogin, datecreated)) as last_activity
            FROM tblclients
            WHERE datecreated >= DATE_SUB(NOW(), INTERVAL 12 MONTH)
            GROUP BY cohort
            ORDER BY cohort DESC
        ");
    }
}
```

## Cohort Analysis Metrics

| Metric | Description |
|--------|-------------|
| Cohort Size | Number of customers in cohort |
| M0, M1, M3, M6, M12 | Retention at each month |
| Revenue Retention | Revenue from cohort over time |
| LTV | Lifetime value by cohort |

## Related Workflows
- [WHMCS Retention Report](./whmcs-retention-report.md)
- [WHMCS CLV Report](./whmcs-clv-report.md)