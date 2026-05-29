# WHMCS Invoice Summary Widget Module

## Overview
Outstanding invoices summary widget with aging analysis and collection status.

## Module File: widget.php

```php
<?php
/**
 * WHMCS Invoice Summary Widget
 * 
 * @package    WHMCS\Module\Widgets
 */

if (!defined("WHMCS")) {
    die("This file cannot be accessed directly");
}

class InvoiceSummaryWidget extends WHMCS\Module\Contracts\WidgetModuleInterface
{
    protected $title = 'Invoice Summary';
    protected $description = 'Outstanding invoices and collection summary';
    protected $priority = 70;
    protected $icon = 'fa-file-invoice-dollar';

    public function getData(): array
    {
        return [
            'total_outstanding' => $this->getTotalOutstanding(),
            'overdue_amount' => $this->getOverdueAmount(),
            'unpaid_count' => $this->getUnpaidCount(),
            'overdue_count' => $this->getOverdueCount(),
            'aging_summary' => $this->getAgingSummary(),
            'top_debtors' => $this->getTopDebtors(),
        ];
    }

    public function generateOutput(array $data): string
    {
        return <<<HTML
<div class="widget-content-padded">
    <div class="row text-center">
        <div class="col-sm-6">
            <div class="metric-box">
                <span class="metric-value">{$data['total_outstanding']}</span>
                <span class="metric-label">Total Outstanding</span>
            </div>
        </div>
        <div class="col-sm-6">
            <div class="metric-box">
                <span class="metric-value text-danger">{$data['overdue_amount']}</span>
                <span class="metric-label">Overdue</span>
            </div>
        </div>
    </div>

    <div class="row text-center">
        <div class="col-sm-6">
            <small>{$data['unpaid_count']} Unpaid</small>
        </div>
        <div class="col-sm-6">
            <small>{$data['overdue_count']} Overdue</small>
        </div>
    </div>

    <hr>

    <div class="aging-summary">
        <h5>Invoice Aging</h5>
        {$this->renderAgingChart($data['aging_summary'])}
    </div>

    <hr>

    <div class="top-debtors">
        <h5>Top Debtors</h5>
        {$this->renderTopDebtors($data['top_debtors'])}
    </div>
</div>

<style>
.metric-box { padding: 10px; }
.metric-value { display: block; font-size: 20px; font-weight: bold; }
.metric-label { display: block; font-size: 11px; color: #777; text-transform: uppercase; }
.aging-bar { height: 20px; margin: 5px 0; }
</style>
HTML;
    }

    protected function getTotalOutstanding(): string
    {
        $total = Capsule::table('tblinvoices')
            ->whereIn('status', ['Unpaid', 'Overdue'])
            ->sum('total');
        return formatCurrency($total ?? 0);
    }

    protected function getOverdueAmount(): string
    {
        $total = Capsule::table('tblinvoices')
            ->where('status', 'Overdue')
            ->sum('total');
        return formatCurrency($total ?? 0);
    }

    protected function getUnpaidCount(): int
    {
        return Capsule::table('tblinvoices')
            ->where('status', 'Unpaid')
            ->count();
    }

    protected function getOverdueCount(): int
    {
        return Capsule::table('tblinvoices')
            ->where('status', 'Overdue')
            ->count();
    }

    protected function getAgingSummary(): array
    {
        $buckets = [
            'current' => ['label' => 'Current', 'max_days' => 0],
            '1_30' => ['label' => '1-30 days', 'max_days' => 30],
            '31_60' => ['label' => '31-60 days', 'max_days' => 60],
            '61_90' => ['label' => '61-90 days', 'max_days' => 90],
            'over_90' => ['label' => 'Over 90 days', 'max_days' => 999],
        ];

        $invoices = Capsule::table('tblinvoices')
            ->whereIn('status', ['Unpaid', 'Overdue'])
            ->get(['total', 'duedate']);

        $summary = ['current' => 0, '1_30' => 0, '31_60' => 0, '61_90' => 0, 'over_90' => 0];

        foreach ($invoices as $invoice) {
            $daysOverdue = (strtotime($invoice->duedate) - time()) / 86400;
            $daysOverdue = -$daysOverdue;

            if ($daysOverdue <= 0) {
                $summary['current'] += $invoice->total;
            } elseif ($daysOverdue <= 30) {
                $summary['1_30'] += $invoice->total;
            } elseif ($daysOverdue <= 60) {
                $summary['31_60'] += $invoice->total;
            } elseif ($daysOverdue <= 90) {
                $summary['61_90'] += $invoice->total;
            } else {
                $summary['over_90'] += $invoice->total;
            }
        }

        return $summary;
    }

    protected function getTopDebtors(): array
    {
        return Capsule::table('tblinvoices')
            ->join('tblclients', 'tblinvoices.userid', '=', 'tblclients.id')
            ->whereIn('tblinvoices.status', ['Unpaid', 'Overdue'])
            ->groupBy('tblinvoices.userid')
            ->selectRaw('tblclients.id, CONCAT(tblclients.firstname, " ", tblclients.lastname) as name, SUM(tblinvoices.total) as total')
            ->orderBy('total', 'desc')
            ->limit(5)
            ->get();
    }

    protected function renderAgingChart(array $aging): string
    {
        $total = array_sum($aging);
        if ($total == 0) {
            return '<p class="text-muted">No outstanding invoices</p>';
        }

        $colors = [
            'current' => '#27ae60',
            '1_30' => '#3498db',
            '31_60' => '#f39c12',
            '61_90' => '#e67e22',
            'over_90' => '#e74c3c',
        ];

        $html = '<div class="progress" style="height: 25px;">';
        foreach ($aging as $key => $amount) {
            if ($amount > 0) {
                $width = ($amount / $total) * 100;
                $html .= "<div class=\"progress-bar\" style=\"width: {$width}%; background: {$colors[$key]}\" 
                    title=\"" . ucfirst(str_replace('_', ' ', $key)) . ": " . formatCurrency($amount) . "\"></div>";
            }
        }
        $html .= '</div>';

        return $html;
    }

    protected function renderTopDebtors(array $debtors): string
    {
        if (empty($debtors)) {
            return '<p class="text-muted">No debtors</p>';
        }

        $html = '<ul class="list-unstyled">';
        foreach ($debtors as $debtor) {
            $html .= '<li style="padding: 5px 0; border-bottom: 1px solid #eee;">
                <span>' . htmlspecialchars($debtor->name) . '</span>
                <span class="pull-right text-danger">' . formatCurrency($debtor->total) . '</span>
            </li>';
        }
        $html .= '</ul>';

        return $html;
    }

    public function getId(): string
    {
        return 'invoice_summary_widget';
    }

    public function getName(): string
    {
        return $this->title;
    }
}

function whmcs_invoice_summary_widget_activate(): array
{
    return ['status' => 'success', 'description' => 'Invoice Summary Widget activated'];
}

function whmcs_invoice_summary_widget_deactivate(): array
{
    return ['status' => 'success', 'description' => 'Invoice Summary Widget deactivated'];
}
