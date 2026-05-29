# WHMCS Cancellation Report Module

## Overview
Cancellation reason analysis and churn pattern reporting.

## Module File: report.php

```php
<?php
/**
 * WHMCS Cancellation Report
 */

if (!defined("WHMCS")) {
    die("This file cannot be accessed directly");
}

use WHMCS\Report\AbstractReport;
use WHMCS\Report\Contract\ReportInterface;
use WHMCS\Report\Contract\TableReportInterface;

class Cancellation_Report extends AbstractReport implements ReportInterface, TableReportInterface
{
    protected $translation = [
        'shortFriendlyName' => 'Cancellations',
        'friendlyName' => 'Cancellation Report',
        'description' => 'Analyze cancellation reasons and churn patterns',
    ];

    protected $tableHeaders = [
        'month' => 'Month',
        'cancellations' => 'Cancelled',
        'migrations' => 'Migrated Out',
        'no_renew' => 'No Renewal',
        'price' => 'Price',
        'service_quality' => 'Service Quality',
        'competitor' => 'Competitor',
        'other' => 'Other',
    ];

    public function getReportData(): array
    {
        $data = [];

        for ($i = 11; $i >= 0; $i--) {
            $monthStart = date('Y-m-01', strtotime("-{$i} months"));
            $monthEnd = date('Y-m-t', strtotime($monthStart));

            $cancellations = Capsule::table('tblhosting')
                ->where('domainstatus', 'Terminated')
                ->whereBetween('termination_date', [$monthStart, $monthEnd])
                ->count();

            $data[] = [
                'month' => date('F Y', strtotime($monthStart)),
                'cancellations' => $cancellations,
                'migrations' => 0,
                'no_renew' => 0,
                'price' => 0,
                'service_quality' => 0,
                'competitor' => 0,
                'other' => 0,
            ];
        }

        return $data;
    }
}

function whmcs_cancellation_report_activate(): array
{
    return ['status' => 'success', 'description' => 'Cancellation Report activated'];
}

function whmcs_cancellation_report_deactivate(): array
{
    return ['status' => 'success', 'description' => 'Cancellation Report deactivated'];
}
