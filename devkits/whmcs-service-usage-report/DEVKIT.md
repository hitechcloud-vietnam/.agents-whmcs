# WHMCS Service Usage Report Module

## Overview
Bandwidth and resource usage reporting for metered services.

## Module File: report.php

```php
<?php
/**
 * WHMCS Service Usage Report
 */

if (!defined("WHMCS")) {
    die("This file cannot be accessed directly");
}

use WHMCS\Report\AbstractReport;
use WHMCS\Report\Contract\ReportInterface;
use WHMCS\Report\Contract\TableReportInterface;

class Service_Usage_Report extends AbstractReport implements ReportInterface, TableReportInterface
{
    protected $translation = [
        'shortFriendlyName' => 'Service Usage',
        'friendlyName' => 'Service Usage Report',
        'description' => 'Bandwidth and resource usage for metered services',
    ];

    protected $tableHeaders = [
        'client' => 'Client',
        'service' => 'Service',
        'bandwidth_used' => 'Bandwidth (GB)',
        'storage_used' => 'Storage (GB)',
        'api_calls' => 'API Calls',
        'estimated_cost' => 'Est. Cost',
    ];

    public function getReportData(): array
    {
        $startDate = date('Y-m-01');
        $endDate = date('Y-m-t');

        return Capsule::table('mod_usage_records')
            ->join('tblhosting', 'mod_usage_records.service_id', '=', 'tblhosting.id')
            ->join('tblclients', 'tblhosting.userid', '=', 'tblclients.id')
            ->where('mod_usage_records.billing_date', '>=', $startDate)
            ->where('mod_usage_records.billing_date', '<=', $endDate)
            ->selectRaw("
                CONCAT(tblclients.firstname, ' ', tblclients.lastname) as client,
                tblhosting.domain as service,
                SUM(JSON_EXTRACT(usage_data, '$.bandwidth_gb')) as bandwidth_used,
                SUM(JSON_EXTRACT(usage_data, '$.storage_gb')) as storage_used,
                SUM(JSON_EXTRACT(usage_data, '$.api_calls')) as api_calls,
                SUM(mod_usage_records.calculated_amount) as estimated_cost
            ")
            ->groupBy('tblhosting.id')
            ->orderBy('estimated_cost', 'desc')
            ->get();
    }

    public function getSummaryData(): array
    {
        $startDate = date('Y-m-01');

        return [
            'total_bandwidth' => Capsule::table('mod_usage_records')
                ->where('billing_date', '>=', $startDate)
                ->sum(Capsule::raw("JSON_EXTRACT(usage_data, '$.bandwidth_gb')")),
            'total_cost' => Capsule::table('mod_usage_records')
                ->where('billing_date', '>=', $startDate)
                ->sum('calculated_amount'),
        ];
    }
}

function whmcs_service_usage_report_activate(): array
{
    return ['status' => 'success', 'description' => 'Service Usage Report activated'];
}

function whmcs_service_usage_report_deactivate(): array
{
    return ['status' => 'success', 'description' => 'Service Usage Report deactivated'];
}
