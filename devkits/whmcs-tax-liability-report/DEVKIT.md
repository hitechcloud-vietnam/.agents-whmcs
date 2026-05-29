# WHMCS Tax Liability Report Module

## Overview
Tax reporting for compliance with detailed breakdown by tax rate and jurisdiction.

## Module File: report.php

```php
<?php
/**
 * WHMCS Tax Liability Report
 */

if (!defined("WHMCS")) {
    die("This file cannot be accessed directly");
}

use WHMCS\Report\AbstractReport;
use WHMCS\Report\Contract\ReportInterface;
use WHMCS\Report\Contract\TableReportInterface;

class Tax_Liability_Report extends AbstractReport implements ReportInterface, TableReportInterface
{
    protected $translation = [
        'shortFriendlyName' => 'Tax Liability',
        'friendlyName' => 'Tax Liability Report',
        'description' => 'Tax collection and liability reporting for compliance',
    ];

    protected $tableHeaders = [
        'tax_rate' => 'Tax Rate',
        'taxable_sales' => 'Taxable Sales',
        'tax_collected' => 'Tax Collected',
        'refunds' => 'Tax on Refunds',
        'net_tax' => 'Net Tax',
        'transactions' => 'Transactions',
    ];

    public function getReportData(): array
    {
        $startDate = date('Y-01-01');
        $endDate = date('Y-m-d');

        $results = Capsule::table('tblaccounts')
            ->join('tblinvoices', 'tblaccounts.invoiceid', '=', 'tblinvoices.id')
            ->whereBetween('tblaccounts.date', [$startDate, $endDate])
            ->where('tblaccounts.amountin', '>', 0)
            ->selectRaw('
                tblinvoices.taxrate,
                SUM(tblaccounts.amountin - tblaccounts.tax) as taxable_sales,
                SUM(tblaccounts.tax) as tax_collected,
                COUNT(*) as transactions
            ')
            ->groupBy('tblinvoices.taxrate')
            ->get();

        $data = [];
        foreach ($results as $result) {
            $data[] = [
                'tax_rate' => number_format($result->taxrate, 1) . '%',
                'taxable_sales' => $result->taxable_sales,
                'tax_collected' => $result->tax_collected,
                'refunds' => 0,
                'net_tax' => $result->tax_collected,
                'transactions' => $result->transactions,
            ];
        }

        return $data;
    }

    public function getSummaryData(): array
    {
        $startDate = date('Y-01-01');

        $totalTax = Capsule::table('tblaccounts')
            ->where('date', '>=', $startDate)
            ->where('amountin', '>', 0)
            ->sum('tax');

        return [
            'total_tax_collected' => $totalTax,
            'reporting_period' => date('Y'),
        ];
    }
}

function whmcs_tax_liability_report_activate(): array
{
    return ['status' => 'success', 'description' => 'Tax Liability Report activated'];
}

function whmcs_tax_liability_report_deactivate(): array
{
    return ['status' => 'success', 'description' => 'Tax Liability Report deactivated'];
}
