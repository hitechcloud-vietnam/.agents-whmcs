# WHMCS Domain Analytics Report Module

## Overview
Domain registration trends and analytics reporting.

## Module File: report.php

```php
<?php
/**
 * WHMCS Domain Analytics Report
 */

if (!defined("WHMCS")) {
    die("This file cannot be accessed directly");
}

use WHMCS\Report\AbstractReport;
use WHMCS\Report\Contract\ReportInterface;
use WHMCS\Report\Contract\TableReportInterface;

class Domain_Analytics_Report extends AbstractReport implements ReportInterface, TableReportInterface
{
    protected $translation = [
        'shortFriendlyName' => 'Domain Analytics',
        'friendlyName' => 'Domain Analytics Report',
        'description' => 'Domain registration trends and analytics',
    ];

    protected $tableHeaders = [
        'tld' => 'TLD',
        'total_domains' => 'Total',
        'new_registrations' => 'New',
        'renewals' => 'Renewals',
        'expiries' => 'Expiries',
        'revenue' => 'Revenue',
    ];

    public function getReportData(): array
    {
        $startDate = date('Y-01-01');

        $tlds = Capsule::table('tbldomains')
            ->selectRaw("
                SUBSTRING_INDEX(domain, '.', -1) as tld,
                COUNT(*) as total_domains
            ")
            ->groupBy('tld')
            ->get();

        $data = [];
        foreach ($tlds as $tld) {
            $data[] = [
                'tld' => '.' . $tld->tld,
                'total_domains' => $tld->total_domains,
                'new_registrations' => 0,
                'renewals' => 0,
                'expiries' => 0,
                'revenue' => 0,
            ];
        }

        return $data;
    }
}

function whmcs_domain_analytics_report_activate(): array
{
    return ['status' => 'success', 'description' => 'Domain Analytics Report activated'];
}

function whmcs_domain_analytics_report_deactivate(): array
{
    return ['status' => 'success', 'description' => 'Domain Analytics Report deactivated'];
}
