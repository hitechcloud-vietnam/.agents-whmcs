# WHMCS Tax Reporting Widget Module

## Overview
Tax summary and reporting widget for compliance tracking.

## Module File: widget.php

```php
<?php
/**
 * WHMCS Tax Reporting Widget
 */

if (!defined("WHMCS")) {
    die("This file cannot be accessed directly");
}

class TaxReportingWidget extends WHMCS\Module\Contracts\WidgetModuleInterface
{
    protected $title = 'Tax Summary';
    protected $description = 'Tax collection and reporting summary';
    protected $priority = 55;
    protected $icon = 'fa-calculator';

    public function getData(): array
    {
        return [
            'tax_collected' => $this->getTaxCollectedThisMonth(),
            'tax_collected_year' => $this->getTaxCollectedThisYear(),
            'transactions' => $this->getTaxableTransactions(),
            'by_rate' => $this->getTaxByRate(),
        ];
    }

    public function generateOutput(array $data): string
    {
        return <<<HTML
<div class="widget-content-padded">
    <div class="row text-center">
        <div class="col-sm-6">
            <div class="metric-box">
                <span class="metric-value">{$data['tax_collected']}</span>
                <span class="metric-label">This Month</span>
            </div>
        </div>
        <div class="col-sm-6">
            <div class="metric-box">
                <span class="metric-value">{$data['tax_collected_year']}</span>
                <span class="metric-label">Year to Date</span>
            </div>
        </div>
    </div>

    <hr>

    <div class="tax-by-rate">
        <h5>Tax by Rate</h5>
        {$this->renderTaxByRate($data['by_rate'])}
    </div>

    <hr>

    <div class="text-center">
        <a href="reports.php?report=tax-liability" class="btn btn-default btn-sm">Full Tax Report</a>
    </div>
</div>
HTML;
    }

    protected function getTaxCollectedThisMonth(): string
    {
        $start = date('Y-m-01');
        $end = date('Y-m-t');

        $tax = Capsule::table('tblaccounts')
            ->whereBetween('date', [$start, $end])
            ->where('amountin', '>', 0)
            ->sum('tax');

        return formatCurrency($tax ?? 0);
    }

    protected function getTaxCollectedThisYear(): string
    {
        $start = date('Y-01-01');

        $tax = Capsule::table('tblaccounts')
            ->where('date', '>=', $start)
            ->where('amountin', '>', 0)
            ->sum('tax');

        return formatCurrency($tax ?? 0);
    }

    protected function getTaxableTransactions(): int
    {
        $start = date('Y-m-01');

        return Capsule::table('tblaccounts')
            ->where('date', '>=', $start)
            ->where('amountin', '>', 0)
            ->count();
    }

    protected function getTaxByRate(): array
    {
        return Capsule::table('tblaccounts')
            ->join('tblinvoices', 'tblaccounts.invoiceid', '=', 'tblinvoices.id')
            ->where('tblaccounts.date', '>=', date('Y-01-01'))
            ->where('tblaccounts.amountin', '>', 0)
            ->groupBy('tblinvoices.taxrate')
            ->selectRaw('tblinvoices.taxrate, SUM(tblaccounts.tax) as total')
            ->get();
    }

    protected function renderTaxByRate(array $rates): string
    {
        if (empty($rates)) {
            return '<p class="text-muted">No tax data</p>';
        }

        $html = '<ul class="list-unstyled">';
        foreach ($rates as $rate) {
            $ratePercent = number_format($rate->taxrate, 1);
            $amount = formatCurrency($rate->total);
            $html .= '<li style="padding: 5px 0; border-bottom: 1px solid #eee;">
                <span>' . $ratePercent . '% Tax</span>
                <span class="pull-right">' . $amount . '</span>
            </li>';
        }
        $html .= '</ul>';
        return $html;
    }

    public function getId(): string { return 'tax_reporting_widget'; }
    public function getName(): string { return $this->title; }
}

function whmcs_tax_reporting_widget_activate(): array
{
    return ['status' => 'success', 'description' => 'Tax Reporting Widget activated'];
}

function whmcs_tax_reporting_widget_deactivate(): array
{
    return ['status' => 'success', 'description' => 'Tax Reporting Widget deactivated'];
}
