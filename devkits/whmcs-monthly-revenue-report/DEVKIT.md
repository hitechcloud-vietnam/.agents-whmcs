# WHMCS Monthly Revenue Report Module

## Overview
Comprehensive monthly revenue reporting with trends, comparisons, and forecasting.

## Module File: report.php

```php
<?php
/**
 * WHMCS Monthly Revenue Report
 * 
 * @package    WHMCS\Module\Reports
 * @copyright  Copyright (c) 2024 HiTech Cloud Ltd
 */

if (!defined("WHMCS")) {
    die("This file cannot be accessed directly");
}

use WHMCS\Report\AbstractReport;
use WHMCS\Report\Contract\ReportInterface;
use WHMCS\Report\Contract\TableReportInterface;

class Monthly_Revenue_Report extends AbstractReport implements ReportInterface, TableReportInterface
{
    protected $translation = [
        'shortFriendlyName' => 'Monthly Revenue',
        'friendlyName' => 'Monthly Revenue Report',
        'description' => 'Comprehensive monthly revenue analysis with trends and comparisons',
    ];

    protected $tableDefinition = [
        'tables' => [
            'main' => [
                'id' => [
                    'type' => 'int',
                    'length' => 11,
                    'hidden' => true,
                ],
                'month' => [
                    'type' => 'string',
                    'label' => 'Month',
                ],
                'revenue' => [
                    'type' => 'currency',
                    'label' => 'Revenue',
                    'format' => '2',
                ],
                'transactions' => [
                    'type' => 'int',
                    'label' => 'Transactions',
                ],
                'avg_order_value' => [
                    'type' => 'currency',
                    'label' => 'Avg Order Value',
                    'format' => '2',
                ],
                'new_clients' => [
                    'type' => 'int',
                    'label' => 'New Clients',
                ],
                'growth' => [
                    'type' => 'percentage',
                    'label' => 'Growth',
                    'format' => '1',
                ],
            ],
        ],
    ];

    protected $tableHeaders = [
        'month' => 'Month',
        'revenue' => 'Revenue',
        'transactions' => 'Transactions',
        'avg_order_value' => 'Avg Order Value',
        'new_clients' => 'New Clients',
        'growth' => 'Growth',
    ];

    /**
     * Get report data
     */
    public function getReportData(): array
    {
        $data = [];
        $previousRevenue = 0;

        for ($i = 11; $i >= 0; $i--) {
            $month = date('m', strtotime("-{$i} months"));
            $year = date('Y', strtotime("-{$i} months"));
            
            $startDate = "{$year}-{$month}-01";
            $endDate = date('Y-m-t', strtotime($startDate));

            // Revenue
            $revenue = Capsule::table('tblaccounts')
                ->whereBetween('date', [$startDate, $endDate])
                ->where('amountin', '>', 0)
                ->sum('amountin');

            // Transactions
            $transactions = Capsule::table('tblaccounts')
                ->whereBetween('date', [$startDate, $endDate])
                ->where('amountin', '>', 0)
                ->count();

            // New clients
            $newClients = Capsule::table('tblclients')
                ->whereBetween('datecreated', [$startDate, $endDate])
                ->count();

            // Calculate growth
            $growth = 0;
            if ($previousRevenue > 0) {
                $growth = (($revenue - $previousRevenue) / $previousRevenue) * 100;
            }

            // Average order value
            $avgOrderValue = $transactions > 0 ? $revenue / $transactions : 0;

            $data[] = [
                'month' => date('F Y', strtotime($startDate)),
                'revenue' => $revenue,
                'transactions' => $transactions,
                'avg_order_value' => $avgOrderValue,
                'new_clients' => $newClients,
                'growth' => $growth,
            ];

            $previousRevenue = $revenue;
        }

        return $data;
    }

    /**
     * Get summary statistics
     */
    public function getSummaryData(): array
    {
        $currentMonth = date('Y-m-01');
        $lastMonth = date('Y-m-01', strtotime('-1 month'));
        $lastMonthEnd = date('Y-m-t', strtotime($lastMonth));

        $currentRevenue = Capsule::table('tblaccounts')
            ->where('date', '>=', $currentMonth)
            ->where('amountin', '>', 0)
            ->sum('amountin');

        $lastRevenue = Capsule::table('tblaccounts')
            ->whereBetween('date', [$lastMonth, $lastMonthEnd])
            ->where('amountin', '>', 0)
            ->sum('amountin');

        $monthGrowth = $lastRevenue > 0 
            ? round((($currentRevenue - $lastRevenue) / $lastRevenue) * 100, 1) 
            : 0;

        return [
            'current_month_revenue' => $currentRevenue,
            'last_month_revenue' => $lastRevenue,
            'month_growth' => $monthGrowth,
            'ytd_revenue' => Capsule::table('tblaccounts')
                ->where('date', '>=', date('Y-01-01'))
                ->where('amountin', '>', 0)
                ->sum('amountin'),
        ];
    }

    /**
     * Additional chart data
     */
    public function getChartData(): array
    {
        $labels = [];
        $revenues = [];

        $data = $this->getReportData();
        foreach ($data as $row) {
            $labels[] = date('M', strtotime($row['month'] . ' 01'));
            $revenues[] = $row['revenue'];
        }

        return [
            'labels' => $labels,
            'datasets' => [
                [
                    'label' => 'Revenue',
                    'data' => $revenues,
                    'fill' => true,
                    'backgroundColor' => 'rgba(54, 162, 235, 0.2)',
                    'borderColor' => 'rgba(54, 162, 235, 1)',
                ],
            ],
        ];
    }
}
```

## Activation & Deactivation

```php
<?php
function whmcs_monthly_revenue_report_activate(): array
{
    try {
        return [
            'status' => 'success',
            'description' => 'Monthly Revenue Report activated',
        ];
    } catch (\Exception $e) {
        return [
            'status' => 'error',
            'description' => 'Failed to activate: ' . $e->getMessage(),
        ];
    }
}

function whmcs_monthly_revenue_report_deactivate(): array
{
    return ['status' => 'success', 'description' => 'Monthly Revenue Report deactivated'];
}
```

## Report Interface Implementation

The report implements `ReportInterface` and `TableReportInterface` requiring:

| Method | Description |
|--------|-------------|
| getReportData() | Return array of data rows |
| getSummaryData() | Return summary statistics |
| getChartData() | Return chart.js compatible data |

## Requirements

- WHMCS 8.0.0+
- PHP 7.4+
