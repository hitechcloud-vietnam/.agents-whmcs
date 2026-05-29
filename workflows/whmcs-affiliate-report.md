# WHMCS Affiliate Report Workflow

## Overview
This workflow generates affiliate program performance and commission reports.

## Prerequisites
- WHMCS with Affiliate Addon enabled
- Admin access for reports
- Affiliate tracking configured

## Step-by-Step Process

### Step 1: Create Affiliate Report Generator
```php
<?php
// /includes/reports/AffiliateReportGenerator.php

class AffiliateReportGenerator
{
    public function generate(array $params = []): array
    {
        $startDate = $params['start_date'] ?? date('Y-m-01');
        $endDate = $params['end_date'] ?? date('Y-m-t');

        return [
            'summary' => $this->getAffiliateSummary($startDate, $endDate),
            'top_affiliates' => $this->getTopAffiliates($startDate, $endDate),
            'commissions' => $this->getCommissionBreakdown($startDate, $endDate),
            'conversion_metrics' => $this->getConversionMetrics($startDate, $endDate),
            'payout_summary' => $this->getPayoutSummary($startDate, $endDate)
        ];
    }

    private function getAffiliateSummary(string $startDate, string $endDate): array
    {
        return [
            'total_affiliates' => Capsule::table('tblaffiliates')->count(),
            'active_affiliates' => Capsule::table('tblaffiliates')
                ->where('lastpaid', '>=', date('Y-m-d', strtotime('-30 days')))
                ->count(),
            'total_referrals' => Capsule::table('tblaffiliatesaccounts')
                ->whereBetween('timestamp', [$startDate, $endDate])
                ->count(),
            'total_commissions' => Capsule::table('tblaffiliates')
                ->sum('balance'),
            'pending_payouts' => Capsule::table('tblaffiliates')
                ->where('balance', '>=', Capsule::table('tblaffiliates')
                    ->selectRaw('MIN(payoutthreshold)'))
                ->sum('balance')
        ];
    }

    private function getTopAffiliates(string $startDate, string $endDate): array
    {
        return Capsule::select("
            SELECT
                a.id,
                c.firstname,
                c.lastname,
                c.email,
                COUNT(aa.id) as referrals,
                SUM(aa.commission) as total_earnings,
                a.balance
            FROM tblaffiliates a
            JOIN tblclients c ON a.clientid = c.id
            LEFT JOIN tblaffiliatesaccounts aa ON a.id = aa.affiliateid
            WHERE aa.timestamp BETWEEN ? AND ?
            GROUP BY a.id
            ORDER BY total_earnings DESC
            LIMIT 20
        ", [$startDate, $endDate]);
    }

    private function getCommissionBreakdown(string $startDate, string $endDate): array
    {
        return Capsule::select("
            SELECT
                DATE(aa.timestamp) as date,
                COUNT(*) as referrals,
                SUM(aa.commission) as commissions
            FROM tblaffiliatesaccounts aa
            WHERE aa.timestamp BETWEEN ? AND ?
            GROUP BY DATE(aa.timestamp)
            ORDER BY date DESC
        ", [$startDate, $endDate]);
    }

    private function getConversionMetrics(string $startDate, string $endDate): array
    {
        $clicks = Capsule::table('tblaffiliates')
            ->whereBetween('date', [$startDate, $endDate])
            ->sum('refferedby');

        $conversions = Capsule::table('tblaffiliatesaccounts')
            ->whereBetween('timestamp', [$startDate, $endDate])
            ->count();

        $conversionRate = $clicks > 0 ? ($conversions / $clicks) * 100 : 0;

        return [
            'total_clicks' => $clicks,
            'total_conversions' => $conversions,
            'conversion_rate' => round($conversionRate, 2),
            'avg_commission_per_conversion' => $conversions > 0
                ? Capsule::table('tblaffiliatesaccounts')
                    ->whereBetween('timestamp', [$startDate, $endDate])
                    ->avg('commission')
                : 0
        ];
    }

    private function getPayoutSummary(string $startDate, string $endDate): array
    {
        return Capsule::select("
            SELECT
                DATE(ap.created_at) as date,
                a.clientid,
                c.firstname,
                c.lastname,
                ap.amount,
                ap.method
            FROM mod_affiliate_payouts ap
            JOIN tblaffiliates a ON ap.affiliate_id = a.id
            JOIN tblclients c ON a.clientid = c.id
            WHERE ap.created_at BETWEEN ? AND ?
            ORDER BY date DESC
        ", [$startDate, $endDate]);
    }
}
```

### Step 2: Create Report Hooks
```php
<?php
// /includes/hooks/affiliate_report_hooks.php

add_hook('MonthlyCronJob', 1, function($vars) {
    $reportGenerator = new AffiliateReportGenerator();

    $report = $reportGenerator->generate([
        'start_date' => date('Y-m-01'),
        'end_date' => date('Y-m-t')
    ]);

    // Store for historical reference
    Capsule::table('mod_reports')->insert([
        'report_type' => 'affiliate_monthly',
        'data' => json_encode($report),
        'created_at' => date('Y-m-d H:i:s')
    ]);

    // Send to affiliate managers
    sendEmail('admin', 'Monthly Affiliate Report', [
        'summary' => $report['summary'],
        'top_affiliates' => $report['top_affiliates']
    ]);

    return $report;
});
```

## Related Workflows
- [WHMCS Revenue Report](./whmcs-revenue-report.md)
- [WHMCS Report Automation](./whmcs-report-automation.md)