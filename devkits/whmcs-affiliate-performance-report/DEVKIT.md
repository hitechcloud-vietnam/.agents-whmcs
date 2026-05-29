# WHMCS Affiliate Performance Report Module

## Overview
Affiliate commission and performance reporting.

## Module File: report.php

```php
<?php
/**
 * WHMCS Affiliate Performance Report
 */

if (!defined("WHMCS")) {
    die("This file cannot be访问 directly");
}

use WHMCS\Report\AbstractReport;
use WHMCS\Report\Contract\ReportInterface;
use WHMCS\Report\Contract\TableReportInterface;

class Affiliate_Performance_Report extends AbstractReport implements ReportInterface, TableReportInterface
{
    protected $translation = [
        'shortFriendlyName' => 'Affiliate Performance',
        'friendlyName' => 'Affiliate Performance Report',
        'description' => 'Track affiliate commissions and referral performance',
    ];

    protected $tableHeaders = [
        'affiliate' => 'Affiliate',
        'referrals' => 'Referrals',
        'total_sales' => 'Total Sales',
        'commissions' => 'Commissions',
        'paid_out' => 'Paid Out',
        'pending' => 'Pending',
        'conversion_rate' => 'Conv. Rate %',
    ];

    public function getReportData(): array
    {
        $startDate = date('Y-01-01');

        $affiliates = Capsule::table('tblaffiliates')
            ->join('tblclients', 'tblaffiliates.clientid', '=', 'tblclients.id')
            ->selectRaw("
                tblaffiliates.id,
                CONCAT(tblclients.firstname, ' ', tblclients.lastname) as affiliate,
                tblaffiliates.referrals,
                tblaffiliates.pendingcommissions as pending,
                tblaffiliates.totalcommissions as total_commissions
            ")
            ->get();

        $data = [];
        foreach ($affiliates as $aff) {
            $data[] = [
                'affiliate' => $aff->affiliate,
                'referrals' => $aff->referrals,
                'total_sales' => 0,
                'commissions' => $aff->total_commissions,
                'paid_out' => $aff->total_commissions - $aff->pending,
                'pending' => $aff->pending,
                'conversion_rate' => 0,
            ];
        }

        return $data;
    }

    public function getSummaryData(): array
    {
        return [
            'total_affiliates' => Capsule::table('tblaffiliates')->count(),
            'total_commissions' => Capsule::table('tblaffiliates')->sum('totalcommissions'),
        ];
    }
}

function whmcs_affiliate_performance_report_activate(): array
{
    return ['status' => 'success', 'description' => 'Affiliate Performance Report activated'];
}

function whmcs_affiliate_performance_report_deactivate(): array
{
    return ['status' => 'success', 'description' => 'Affiliate Performance Report deactivated'];
}
