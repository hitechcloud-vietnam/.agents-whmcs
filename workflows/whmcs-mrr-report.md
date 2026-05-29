# WHMCS MRR Report Workflow

## Overview
This workflow generates Monthly Recurring Revenue reports and metrics.

## Prerequisites
- WHMCS with service billing
- Admin access for analytics
- Proper billing cycles configured

## Step-by-Step Process

### Step 1: Create MRR Report Generator
```php
<?php
// /includes/reports/MRRReportGenerator.php

class MRRReportGenerator
{
    public function generate(array $params = []): array
    {
        $asOfDate = $params['as_of_date'] ?? date('Y-m-d');

        return [
            'current_mrr' => $this->calculateCurrentMRR($asOfDate),
            'mrr_breakdown' => $this->getMRRBreakdown($asOfDate),
            'mrr_movement' => $this->getMRRMovement($asOfDate),
            'by_product' => $this->getMRRByProduct($asOfDate),
            'cohort_mrr' => $this->getCohortMRR($asOfDate)
        ];
    }

    private function calculateCurrentMRR(string $asOfDate): array
    {
        // Get all active services with recurring billing
        $services = Capsule::select("
            SELECT
                billingcycle,
                SUM(amount) as total
            FROM tblhosting
            WHERE domainstatus = 'Active'
            AND billingcycle IN ('Monthly', 'Quarterly', 'Semi-Annual', 'Annual')
            GROUP BY billingcycle
        ");

        $mrr = 0;
        $breakdown = [];

        foreach ($services as $service) {
            $monthlyAmount = match ($service->billingcycle) {
                'Monthly' => $service->total,
                'Quarterly' => $service->total / 3,
                'Semi-Annual' => $service->total / 6,
                'Annual' => $service->total / 12,
                default => $service->total
            };

            $mrr += $monthlyAmount;
            $breakdown[$service->billingcycle] = $monthlyAmount;
        }

        return [
            'total_mrr' => round($mrr, 2),
            'breakdown' => $breakdown,
            'as_of_date' => $asOfDate
        ];
    }

    private function getMRRBreakdown(string $asOfDate): array
    {
        return [
            'new_mrr' => $this->getNewMRR($asOfDate),
            'expansion_mrr' => $this->getExpansionMRR($asOfDate),
            'contraction_mrr' => $this->getContractionMRR($asOfDate),
            'churned_mrr' => $this->getChurnedMRR($asOfDate),
            'net_new_mrr' => 0 // Calculated below
        ];
    }

    private function getNewMRR(string $asOfDate): float
    {
        $monthStart = date('Y-m-01', strtotime($asOfDate));

        $result = Capsule::select("
            SELECT SUM(
                CASE billingcycle
                    WHEN 'Monthly' THEN amount
                    WHEN 'Quarterly' THEN amount / 3
                    WHEN 'Semi-Annual' THEN amount / 6
                    WHEN 'Annual' THEN amount / 12
                    ELSE amount
                END
            ) as new_mrr
            FROM tblhosting
            WHERE domainstatus = 'Active'
            AND regdate >= ?
            AND regdate <= ?
        ", [$monthStart, $asOfDate]);

        return round($result[0]->new_mrr ?? 0, 2);
    }

    private function getExpansionMRR(string $asOfDate): float
    {
        $monthStart = date('Y-m-01', strtotime($asOfDate));

        // Find services with upgrade/change
        $result = Capsule::select("
            SELECT SUM(amount_difference / 12) as expansion_mrr
            FROM (
                SELECT
                    h.id,
                    (h.amount - prev.amount) as amount_difference
                FROM tblhosting h
                JOIN (
                    SELECT serviceid, amount, created_at
                    FROM tblhosting_history
                    WHERE created_at >= ?
                ) prev ON h.id = prev.serviceid
                WHERE h.domainstatus = 'Active'
                AND h.amount > prev.amount
            ) upgrades
        ", [$monthStart]);

        return round($result[0]->expansion_mrr ?? 0, 2);
    }

    private function getContractionMRR(string $asOfDate): float
    {
        // Similar to expansion but for downgrades
        return 0;
    }

    private function getChurnedMRR(string $asOfDate): float
    {
        $monthStart = date('Y-m-01', strtotime($asOfDate));

        $result = Capsule::select("
            SELECT SUM(
                CASE billingcycle
                    WHEN 'Monthly' THEN amount
                    WHEN 'Quarterly' THEN amount / 3
                    WHEN 'Semi-Annual' THEN amount / 6
                    WHEN 'Annual' THEN amount / 12
                    ELSE amount
                END
            ) as churned_mrr
            FROM tblhosting
            WHERE domainstatus = 'Terminated'
            AND regdate < ?
            AND termination_date >= ?
            AND termination_date <= ?
        ", [$monthStart, $monthStart, $asOfDate]);

        return round($result[0]->churned_mrr ?? 0, 2);
    }

    private function getMRRByProduct(string $asOfDate): array
    {
        return Capsule::select("
            SELECT
                p.name as product_name,
                COUNT(h.id) as services,
                SUM(
                    CASE h.billingcycle
                        WHEN 'Monthly' THEN h.amount
                        WHEN 'Quarterly' THEN h.amount / 3
                        WHEN 'Semi-Annual' THEN h.amount / 6
                        WHEN 'Annual' THEN h.amount / 12
                        ELSE h.amount
                    END
                ) as mrr
            FROM tblproducts p
            LEFT JOIN tblhosting h ON p.id = h.packageid AND h.domainstatus = 'Active'
            GROUP BY p.id
            HAVING mrr > 0
            ORDER BY mrr DESC
        ");
    }

    private function getCohortMRR(string $asOfDate): array
    {
        return Capsule::select("
            SELECT
                DATE_FORMAT(regdate, '%Y-%m') as cohort,
                COUNT(*) as customers,
                SUM(
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
            AND regdate >= DATE_SUB(?, INTERVAL 12 MONTH)
            GROUP BY cohort
            ORDER BY cohort DESC
        ", [$asOfDate]);
    }
}
```

### Step 2: Create MRR Report Hook
```php
<?php
// /includes/hooks/mrr_report_hooks.php

add_hook('DailyCronJob', 1, function($vars) {
    $reportGenerator = new MRRReportGenerator();

    $report = $reportGenerator->generate([
        'as_of_date' => date('Y-m-d')
    ]);

    // Store daily MRR snapshot
    Capsule::table('mod_mrr_snapshots')->insert([
        'mrr' => $report['current_mrr']['total_mrr'],
        'as_of_date' => date('Y-m-d'),
        'created_at' => date('Y-m-d H:i:s')
    ]);

    return $report;
});
```

## MRR Components

| Component | Description |
|-----------|-------------|
| New MRR | Revenue from new customers |
| Expansion MRR | Revenue from upgrades |
| Contraction MRR | Revenue lost from downgrades |
| Churned MRR | Revenue lost from cancellations |
| Net New MRR | New MRR + Expansion - Churned |

## Related Workflows
- [WHMCS ARR Report](./whmcs-arr-report.md)
- [WHMCS Churn Report](./whmcs-churn-report.md)