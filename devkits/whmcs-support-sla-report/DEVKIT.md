# WHMCS Support SLA Report Module

## Overview
Ticket SLA compliance and support performance metrics.

## Module File: report.php

```php
<?php
/**
 * WHMCS Support SLA Report
 */

if (!defined("WHMCS")) {
    die("This file cannot be accessed directly");
}

use WHMCS\Report\AbstractReport;
use WHMCS\Report\Contract\ReportInterface;
use WHMCS\Report\Contract\TableReportInterface;

class Support_SLA_Report extends AbstractReport implements ReportInterface, TableReportInterface
{
    protected $translation = [
        'shortFriendlyName' => 'Support SLA',
        'friendlyName' => 'Support SLA Compliance Report',
        'description' => 'Track ticket SLA compliance and response times',
    ];

    protected $tableHeaders = [
        'month' => 'Month',
        'tickets_closed' => 'Closed',
        'first_response_met' => 'First Response Met',
        'resolution_met' => 'Resolution Met',
        'sla_compliance' => 'SLA Compliance %',
        'avg_response_time' => 'Avg Response (h)',
        'avg_resolution_time' => 'Avg Resolution (h)',
    ];

    public function getReportData(): array
    {
        $data = [];

        for ($i = 11; $i >= 0; $i--) {
            $monthStart = date('Y-m-01', strtotime("-{$i} months"));
            $monthEnd = date('Y-m-t', strtotime($monthStart));

            $ticketsClosed = Capsule::table('mod_ticket_sla')
                ->where('status', 'closed')
                ->whereBetween('created_at', [$monthStart, $monthEnd])
                ->count();

            $firstResponseMet = Capsule::table('mod_ticket_sla')
                ->where('status', 'closed')
                ->where('first_response_status', 'met')
                ->whereBetween('created_at', [$monthStart, $monthEnd])
                ->count();

            $resolutionMet = Capsule::table('mod_ticket_sla')
                ->where('status', 'closed')
                ->where('resolution_status', 'met')
                ->whereBetween('created_at', [$monthStart, $monthEnd])
                ->count();

            $slaCompliance = $ticketsClosed > 0 ? ($firstResponseMet / $ticketsClosed) * 100 : 100;

            $avgResponseTime = Capsule::table('mod_ticket_sla')
                ->where('status', 'closed')
                ->whereNotNull('first_response_at')
                ->whereBetween('created_at', [$monthStart, $monthEnd])
                ->avg('resolution_time_minutes');

            $data[] = [
                'month' => date('F Y', strtotime($monthStart)),
                'tickets_closed' => $ticketsClosed,
                'first_response_met' => $firstResponseMet,
                'resolution_met' => $resolutionMet,
                'sla_compliance' => round($slaCompliance, 1),
                'avg_response_time' => round(($avgResponseTime ?? 0) / 60, 1),
                'avg_resolution_time' => 0,
            ];
        }

        return $data;
    }
}

function whmcs_support_sla_report_activate(): array
{
    return ['status' => 'success', 'description' => 'Support SLA Report activated'];
}

function whmcs_support_sla_report_deactivate(): array
{
    return ['status' => 'success', 'description' => 'Support SLA Report deactivated'];
}
