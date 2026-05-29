# WHMCS ARR Report Workflow

## Overview
This workflow generates Annual Recurring Revenue reports.

## Prerequisites
- WHMCS with service billing
- Admin access for analytics
- MRR calculation configured

## Step-by-Step Process

### Step 1: Create ARR Calculator
```php
<?php
// /includes/reports/ARRReportGenerator.php

class ARRReportGenerator
{
    public function generate(array $params = []): array
    {
        $asOfDate = $params['as_of_date'] ?? date('Y-m-d');

        $mrrReport = $this->getMRRReport();
        $mrr = $mrrReport['current_mrr']['total_mrr'];

        return [
            'arr' => round($mrr * 12, 2),
            'mrr' => $mrr,
            'arr_breakdown' => [
                'by_plan' => $this->getARRByPlan($asOfDate),
                'by_cohort' => $this->getARRByCohort($asOfDate)
            ],
            'arr_growth' => $this->calculateARRGrowth($asOfDate),
            'arr_projections' => $this->getARRProjections()
        ];
    }

    private function getMRRReport()
    {
        $services = Capsule::select("
            SELECT
                SUM(
                    CASE billingcycle
                        WHEN 'Monthly' THEN amount
                        WHEN 'Quarterly' THEN amount / 3
                        WHEN 'Semi-Annual' THEN amount / 6
                        WHEN 'Annual' THEN amount / 12
                        ELSE amount
                    END
                ) as total_mrr
            FROM tblhosting
            WHERE domainstatus = 'Active'
        ");

        return [
            'current_mrr' => ['total_mrr' => $services[0]->total_mrr ?? 0]
        ];
    }

    private function getARRByPlan(string $asOfDate): array
    {
        return Capsule::select("
            SELECT
                p.name as plan_name,
                COUNT(h.id) as customers,
                SUM(
                    CASE h.billingcycle
                        WHEN 'Monthly' THEN amount
                        WHEN 'Quarterly' THEN amount / 3
                        WHEN 'Semi-Annual' THEN amount / 6
                        WHEN 'Annual' THEN amount / 12
                        ELSE amount
                    END
                ) * 12 as arr
            FROM tblproducts p
            JOIN tblhosting h ON p.id = h.packageid
            WHERE h.domainstatus = 'Active'
            GROUP BY p.id
            ORDER BY arr DESC
        ");
    }

    private function getARRByCohort(string $asOfDate): array
    {
        return Capsule::select("
            SELECT
                DATE_FORMAT(regdate, '%Y') as year,
                COUNT(DISTINCT userid) as customers,
                SUM(
                    CASE billingcycle
                        WHEN 'Monthly' THEN amount
                        WHEN 'Quarterly' THEN amount / 3
                        WHEN 'Semi-Annual' THEN amount / 6
                        WHEN 'Annual' THEN amount / 12
                        ELSE amount
                    END
                ) * 12 as arr
            FROM tblhosting
            WHERE domainstatus = 'Active'
            GROUP BY year
            ORDER BY year
        ");
    }

    private function calculateARRGrowth(string $asOfDate): array
    {
        $currentARR = $this->generate()['arr'];

        // Get ARR from 30 days ago
        $lastMonth = Capsule::table('mod_mrr_snapshots')
            ->where('as_of_date', '<=', date('Y-m-d', strtotime('-30 days')))
            ->orderBy('as_of_date', 'DESC')
            ->first();

        $previousARR = $lastMonth ? $lastMonth->mrr * 12 : 0;

        $growth = $previousARR > 0
            ? (($currentARR - $previousARR) / $previousARR) * 100
            : 0;

        return [
            'current_arr' => $currentARR,
            'previous_arr' => $previousARR,
            'growth_percentage' => round($growth, 2),
            'growth_amount' => $currentARR - $previousARR
        ];
    }

    private function getARRProjections(): array
    {
        // Get current MRR
        $currentMRR = Capsule::select("
            SELECT SUM(
                CASE billingcycle
                    WHEN 'Monthly' THEN amount
                    WHEN 'Quarterly' THEN amount / 3
                    WHEN 'Semi-Annual' THEN amount / 6
                    WHEN 'Annual' THEN amount / 12
                    ELSE amount
                END
            ) as mrr
            FROM tblhosting
            WHERE domainstatus = 'Active'
        ")[0]->mrr ?? 0;

        // Simple projection based on current MRR and growth rate
        $monthlyGrowthRate = 0.05; // 5% default

        $projections = [];
        for ($month = 1; $month <= 12; $month++) {
            $futureMRR = $currentMRR * pow(1 + $monthlyGrowthRate, $month);
            $projections[] = [
                'month' => date('Y-m', strtotime("+{$month} months")),
                'projected_mrr' => round($futureMRR, 2),
                'projected_arr' => round($futureMRR * 12, 2)
            ];
        }

        return $projections;
    }
}
```

### Step 2: Create ARR Report Hook
```php
<?php
// /includes/hooks/arr_report_hooks.php

add_hook('MonthlyCronJob', 1, function($vars) {
    $reportGenerator = new ARRReportGenerator();

    $report = $reportGenerator->generate([
        'as_of_date' => date('Y-m-d')
    ]);

    // Store report
    Capsule::table('mod_reports')->insert([
        'report_type' => 'arr_monthly',
        'data' => json_encode($report),
        'created_at' => date('Y-m-d H:i:s')
    ]);

    // Send to stakeholders
    sendEmail('admin', 'Monthly ARR Report', [
        'current_arr' => formatCurrency($report['arr']),
        'mrr' => formatCurrency($report['mrr']),
        'growth' => $report['arr_growth']['growth_percentage'] . '%'
    ]);

    return $report;
});
```

## ARR vs MRR

| Metric | Calculation | Use Case |
|--------|-------------|----------|
| MRR | Monthly Recurring Revenue | Short-term tracking |
| ARR | MRR x 12 | Annual performance |

## Related Workflows
- [WHMCS MRR Report](./whmcs-mrr-report.md)
- [WHMCS Revenue Report](./whmcs-revenue-report.md)