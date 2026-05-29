# WHMCS CAC Report Workflow

## Overview
This workflow generates Customer Acquisition Cost reports and analysis.

## Prerequisites
- WHMCS with marketing tracking
- Admin access for analytics
- Marketing spend data

## Step-by-Step Process

### Step 1: Create CAC Calculator
```php
<?php
// /includes/reports/CACReportGenerator.php

class CACReportGenerator
{
    public function generate(array $params = []): array
    {
        $startDate = $params['start_date'] ?? date('Y-m-01');
        $endDate = $params['end_date'] ?? date('Y-m-t');

        return [
            'summary' => $this->getCACSummary($startDate, $endDate),
            'by_channel' => $this->getCACByChannel($startDate, $endDate),
            'by_campaign' => $this->getCACByCampaign($startDate, $endDate),
            'cac_trend' => $this->getCACTrend($startDate, $endDate),
            'lifetime_value_ratio' => $this->getCLTVCACRatio()
        ];
    }

    private function getCACSummary(string $startDate, string $endDate): array
    {
        // Total marketing spend
        $marketingSpend = $this->getMarketingSpend($startDate, $endDate);

        // New customers acquired
        $newCustomers = Capsule::table('tblclients')
            ->whereBetween('datecreated', [$startDate, $endDate])
            ->count();

        $cac = $newCustomers > 0 ? $marketingSpend / $newCustomers : 0;

        return [
            'marketing_spend' => $marketingSpend,
            'new_customers' => $newCustomers,
            'cac' => round($cac, 2),
            'period' => ['start' => $startDate, 'end' => $endDate]
        ];
    }

    private function getMarketingSpend(string $startDate, string $endDate): float
    {
        // Get from marketing spend table
        $result = Capsule::table('mod_marketing_spend')
            ->whereBetween('date', [$startDate, $endDate])
            ->sum('amount');

        return $result ?? 0;
    }

    private function getCACByChannel(string $startDate, string $endDate): array
    {
        return Capsule::select("
            SELECT
                channel,
                SUM(spend) as total_spend,
                COUNT(DISTINCT client_id) as conversions,
                SUM(spend) / NULLIF(COUNT(DISTINCT client_id), 0) as cac
            FROM mod_marketing_attribution
            WHERE date BETWEEN ? AND ?
            GROUP BY channel
            ORDER BY cac
        ", [$startDate, $endDate]);
    }

    private function getCACByCampaign(string $startDate, string $endDate): array
    {
        return Capsule::select("
            SELECT
                campaign,
                channel,
                SUM(spend) as spend,
                COUNT(DISTINCT client_id) as conversions,
                SUM(spend) / NULLIF(COUNT(DISTINCT client_id), 0) as cac,
                SUM(revenue) / NULLIF(COUNT(DISTINCT client_id), 0) as revenue_per_acquisition
            FROM mod_marketing_attribution
            WHERE date BETWEEN ? AND ?
            GROUP BY campaign, channel
            ORDER BY conversions DESC
        ", [$startDate, $endDate]);
    }

    private function getCACTrend(string $startDate, string $endDate): array
    {
        return Capsule::select("
            SELECT
                DATE_FORMAT(date, '%Y-%m') as month,
                SUM(spend) as spend,
                COUNT(DISTINCT client_id) as customers,
                SUM(spend) / NULLIF(COUNT(DISTINCT client_id), 0) as cac
            FROM mod_marketing_attribution
            WHERE date >= DATE_SUB(?, INTERVAL 12 MONTH)
            GROUP BY month
            ORDER BY month
        ", [$startDate]);
    }

    private function getCLTVCACRatio(): float
    {
        // Get average CLV
        $avgClv = Capsule::select("
            SELECT AVG(customer_ltv) as avg_clv
            FROM (
                SELECT userid, SUM(total) as customer_ltv
                FROM tblinvoices WHERE status = 'Paid' GROUP BY userid
            ) revenue
        ")[0]->avg_clv ?? 0;

        // Get average CAC
        $avgCac = Capsule::select("
            SELECT AVG(cac) as avg_cac
            FROM mod_marketing_attribution
        ")[0]->avg_cac ?? 0;

        return $avgCac > 0 ? round($avgClv / $avgCac, 2) : 0;
    }
}
```

## Related Workflows
- [WHMCS CLV Report](./whmcs-clv-report.md)
- [WHMCS Revenue Report](./whmcs-revenue-report.md)