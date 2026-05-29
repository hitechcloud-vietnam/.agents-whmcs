# WHMCS CLV Report Workflow

## Overview
This workflow generates Customer Lifetime Value reports and analysis.

## Prerequisites
- WHMCS with service billing
- Admin access for analytics
- Historical transaction data

## Step-by-Step Process

### Step 1: Create CLV Calculator
```php
<?php
// /includes/reports/CLVReportGenerator.php

class CLVReportGenerator
{
    public function generate(array $params = []): array
    {
        return [
            'summary' => $this->getCLVSummary(),
            'by_segment' => $this->getCLVBySegment(),
            'top_customers' => $this->getTopCustomersByCLV(),
            'cohort_analysis' => $this->getCohortCLVAnalysis(),
            'churn_risk' => $this->getChurnRiskAnalysis()
        ];
    }

    private function getCLVSummary(): array
    {
        $data = Capsule::select("
            SELECT
                COUNT(DISTINCT c.id) as total_customers,
                SUM(t.total) as total_revenue,
                AVG(customer_ltv) as avg_clv,
                MIN(customer_ltv) as min_clv,
                MAX(customer_ltv) as max_clv
            FROM tblclients c
            JOIN (
                SELECT
                    userid,
                    SUM(total) as customer_ltv
                FROM tblinvoices
                WHERE status = 'Paid'
                GROUP BY userid
            ) revenue ON c.id = revenue.userid
        ");

        return (array)$data[0];
    }

    private function calculateCustomerCLV(int $clientId): float
    {
        $result = Capsule::select("
            SELECT SUM(total) as ltv
            FROM tblinvoices
            WHERE userid = ?
            AND status = 'Paid'
        ", [$clientId]);

        return $result[0]->ltv ?? 0;
    }

    private function getCLVBySegment(): array
    {
        return Capsule::select("
            SELECT
                CASE
                    WHEN clv < 100 THEN 'Under $100'
                    WHEN clv < 500 THEN '$100-$500'
                    WHEN clv < 1000 THEN '$500-$1000'
                    WHEN clv < 5000 THEN '$1000-$5000'
                    ELSE 'Over $5000'
                END as segment,
                COUNT(*) as customers,
                SUM(clv) as total_value,
                AVG(clv) as avg_value
            FROM (
                SELECT
                    c.id,
                    c.groupid,
                    SUM(i.total) as clv
                FROM tblclients c
                JOIN tblinvoices i ON c.id = i.userid AND i.status = 'Paid'
                GROUP BY c.id
            ) customer_values
            GROUP BY segment
            ORDER BY MIN(segment)
        ");
    }

    private function getTopCustomersByCLV(): array
    {
        return Capsule::select("
            SELECT
                c.id,
                c.firstname,
                c.lastname,
                c.email,
                c.datecreated,
                COUNT(DISTINCT h.id) as services,
                SUM(i.total) as total_revenue
            FROM tblclients c
            LEFT JOIN tblinvoices i ON c.id = i.userid AND i.status = 'Paid'
            LEFT JOIN tblhosting h ON c.id = h.userid
            GROUP BY c.id
            HAVING total_revenue > 0
            ORDER BY total_revenue DESC
            LIMIT 50
        ");
    }

    private function getCohortCLVAnalysis(): array
    {
        return Capsule::select("
            SELECT
                DATE_FORMAT(c.datecreated, '%Y-%m') as cohort,
                COUNT(DISTINCT c.id) as customers,
                SUM(CASE WHEN months_active = 0 THEN revenue ELSE 0 END) as m0,
                SUM(CASE WHEN months_active = 1 THEN revenue ELSE 0 END) as m1,
                SUM(CASE WHEN months_active = 3 THEN revenue ELSE 0 END) as m3,
                SUM(CASE WHEN months_active = 6 THEN revenue ELSE 0 END) as m6,
                SUM(CASE WHEN months_active = 12 THEN revenue ELSE 0 END) as m12
            FROM tblclients c
            JOIN (
                SELECT
                    userid,
                    SUM(total) as revenue,
                    FLOOR(DATEDIFF(NOW(), (SELECT MAX(datepaid) FROM tblinvoices WHERE userid = tblinvoices.userid AND status = 'Paid')) / 30) as months_active
                FROM tblinvoices
                WHERE status = 'Paid'
                GROUP BY userid
            ) revenue ON c.id = revenue.userid
            GROUP BY cohort
            ORDER BY cohort DESC
            LIMIT 12
        ");
    }

    private function getChurnRiskAnalysis(): array
    {
        return Capsule::select("
            SELECT
                c.id,
                c.firstname,
                c.lastname,
                c.lastlogin,
                DATEDIFF(NOW(), c.lastlogin) as days_inactive,
                COUNT(DISTINCT i.id) as recent_invoices,
                SUM(CASE WHEN i.datepaid >= DATE_SUB(NOW(), INTERVAL 90 DAY) THEN i.total ELSE 0 END) as recent_spend
            FROM tblclients c
            LEFT JOIN tblinvoices i ON c.id = i.userid AND i.status = 'Paid'
            WHERE c.status = 'Active'
            GROUP BY c.id
            HAVING days_inactive > 60 AND recent_spend < 100
            ORDER BY days_inactive DESC
        ");
    }
}
```

### Step 2: Create CLV Report Hook
```php
<?php
// /includes/hooks/clv_report_hooks.php

add_hook('MonthlyCronJob', 1, function($vars) {
    $reportGenerator = new CLVReportGenerator();

    $report = $reportGenerator->generate();

    // Store report
    Capsule::table('mod_reports')->insert([
        'report_type' => 'clv_monthly',
        'data' => json_encode($report),
        'created_at' => date('Y-m-d H:i:s')
    ]);

    // Send to management
    sendEmail('admin', 'Monthly CLV Report', [
        'avg_clv' => formatCurrency($report['summary']['avg_clv']),
        'total_customers' => $report['summary']['total_customers'],
        'by_segment' => $report['by_segment']
    ]);

    return $report;
});
```

## CLV Metrics

| Metric | Description |
|--------|-------------|
| Average CLV | Mean revenue per customer |
| CLV by Segment | Breakdown by customer tier |
| Top Customers | Highest lifetime value customers |
| Churn Risk | Low-value at-risk customers |

## Related Workflows
- [WHMCS CAC Report](./whmcs-cac-report.md)
- [WHMCS Retention Report](./whmcs-retention-report.md)