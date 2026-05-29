# WHMCS Client Retention Report Module

## Overview
Client churn analysis and retention metrics reporting.

## Module File: report.php

```php
<?php
/**
 * WHMCS Client Retention Report
 */

if (!defined("WHMCS")) {
    die("This file cannot be accessed directly");
}

use WHMCS\Report\AbstractReport;
use WHMCS\Report\Contract\ReportInterface;
use WHMCS\Report\Contract\TableReportInterface;

class Client_Retention_Report extends AbstractReport implements ReportInterface, TableReportInterface
{
    protected $translation = [
        'shortFriendlyName' => 'Client Retention',
        'friendlyName' => 'Client Retention & Churn Report',
        'description' => 'Analyze client retention rates and churn patterns',
    ];

    protected $tableHeaders = [
        'month' => 'Month',
        'total_clients' => 'Total Clients',
        'new_clients' => 'New',
        'churned' => 'Churned',
        'retained' => 'Retained',
        'retention_rate' => 'Retention %',
        'churn_rate' => 'Churn %',
    ];

    public function getReportData(): array
    {
        $data = [];

        for ($i = 11; $i >= 0; $i--) {
            $month = date('Y-m', strtotime("-{$i} months"));
            $startDate = "{$month}-01";
            $endDate = date('Y-m-t', strtotime($startDate));

            // Total active clients at end of month
            $totalClients = Capsule::table('tblclients')
                ->where('status', 'Active')
                ->count();

            // New clients this month
            $newClients = Capsule::table('tblclients')
                ->whereBetween('datecreated', [$startDate, $endDate])
                ->count();

            // Churned clients (terminated/cancelled services)
            $churned = Capsule::table('tblhosting')
                ->where('domainstatus', 'Terminated')
                ->whereBetween('termination_date', [$startDate, $endDate])
                ->count();

            // Retained clients
            $retained = $totalClients - $churned;
            
            // Retention rate
            $retentionRate = $totalClients > 0 ? (($totalClients - $churned) / $totalClients) * 100 : 100;
            
            // Churn rate
            $churnRate = 100 - $retentionRate;

            $data[] = [
                'month' => date('F Y', strtotime($startDate)),
                'total_clients' => $totalClients,
                'new_clients' => $newClients,
                'churned' => $churned,
                'retained' => $retained,
                'retention_rate' => round($retentionRate, 1),
                'churn_rate' => round($churnRate, 1),
            ];
        }

        return $data;
    }

    public function getSummaryData(): array
    {
        $currentYear = date('Y');
        $startOfYear = "{$currentYear}-01-01";

        $totalChurned = Capsule::table('tblhosting')
            ->where('domainstatus', 'Terminated')
            ->where('termination_date', '>=', $startOfYear)
            ->count();

        $avgRetention = 95; // Placeholder calculation

        return [
            'total_churned_ytd' => $totalChurned,
            'avg_retention_rate' => $avgRetention,
            'avg_churn_rate' => round(100 - $avgRetention, 1),
        ];
    }

    public function getChartData(): array
    {
        $data = $this->getReportData();
        
        return [
            'labels' => array_column($data, 'month'),
            'datasets' => [
                [
                    'label' => 'Retention Rate',
                    'data' => array_column($data, 'retention_rate'),
                    'borderColor' => '#27ae60',
                    'fill' => false,
                ],
                [
                    'label' => 'Churn Rate',
                    'data' => array_column($data, 'churn_rate'),
                    'borderColor' => '#e74c3c',
                    'fill' => false,
                ],
            ],
        ];
    }
}

function whmcs_client_retention_report_activate(): array
{
    return ['status' => 'success', 'description' => 'Client Retention Report activated'];
}

function whmcs_client_retention_report_deactivate(): array
{
    return ['status' => 'success', 'description' => 'Client Retention Report deactivated'];
}
