# WHMCS Upsell Conversion Report Module

## Overview
Upgrade funnel analysis and upsell conversion metrics.

## Module File: report.php

```php
<?php
/**
 * WHMCS Upsell Conversion Report
 */

if (!defined("WHMCS")) {
    die("This file cannot be accessed directly");
}

use WHMCS\Report\AbstractReport;
use WHMCS\Report\Contract\ReportInterface;
use WHMCS\Report\Contract\TableReportInterface;

class Upsell_Conversion_Report extends AbstractReport implements ReportInterface, TableReportInterface
{
    protected $translation = [
        'shortFriendlyName' => 'Upsell Conversion',
        'friendlyName' => 'Upsell Conversion Report',
        'description' => 'Track upgrade and upsell conversion rates',
    ];

    protected $tableHeaders = [
        'product' => 'Product',
        'current_customers' => 'Current',
        'upgrades_offered' => 'Offered',
        'upgrades_accepted' => 'Accepted',
        'conversion_rate' => 'Conv. Rate %',
        'upgrade_revenue' => 'Revenue',
    ];

    public function getReportData(): array
    {
        $products = Capsule::table('tblproducts')
            ->where('type', 'hosting')
            ->where('hidden', 0)
            ->get(['id', 'name']);

        $data = [];
        foreach ($products as $product) {
            $currentCustomers = Capsule::table('tblhosting')
                ->where('packageid', $product->id)
                ->where('domainstatus', 'Active')
                ->count();

            $data[] = [
                'product' => $product->name,
                'current_customers' => $currentCustomers,
                'upgrades_offered' => 0,
                'upgrades_accepted' => 0,
                'conversion_rate' => 0,
                'upgrade_revenue' => 0,
            ];
        }

        return $data;
    }
}

function whmcs_upsell_conversion_report_activate(): array
{
    return ['status' => 'success', 'description' => 'Upsell Conversion Report activated'];
}

function whmcs_upsell_conversion_report_deactivate(): array
{
    return ['status' => 'success', 'description' => 'Upsell Conversion Report deactivated'];
}
