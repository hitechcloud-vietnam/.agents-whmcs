# WHMCS Revenue Dashboard Widget Module

## Overview
Revenue analytics widget displaying financial metrics, trends, and forecasting data in the WHMCS admin dashboard.

## Module Structure

### Main Module File: widget.php

```php
<?php
/**
 * WHMCS Revenue Dashboard Widget
 * 
 * @package    WHMCS\Module\Widgets
 * @copyright  Copyright (c) 2024 HiTech Cloud Ltd
 * @license    Commercial License
 */

if (!defined("WHMCS")) {
    die("This file cannot be accessed directly");
}

/**
 * Revenue Dashboard Widget
 */
class RevenueDashboardWidget extends WHMCS\Module\Contracts\WidgetModuleInterface
{
    /**
     * @var string Widget title
     */
    protected $title = 'Revenue Analytics';

    /**
     * @var string Widget description
     */
    protected $description = 'Display revenue analytics, trends, and financial metrics';

    /**
     * @var int Widget priority
     */
    protected $priority = 100;

    /**
     * @var string Widget icon
     */
    protected $icon = 'fa-chart-line';

    /**
     * @var array Required permissions
     */
    protected $requiredPermission = [
        'reports',
        'invoices',
    ];

    /**
     * Get widget data
     *
     * @return array
     */
    public function getData(): array
    {
        $data = [
            'today' => $this->getTodayRevenue(),
            'yesterday' => $this->getYesterdayRevenue(),
            'this_month' => $this->getMonthRevenue(date('m'), date('Y')),
            'last_month' => $this->getMonthRevenue(date('m', strtotime('-1 month')), date('Y', strtotime('-1 month'))),
            'this_year' => $this->getYearRevenue(date('Y')),
            'last_year' => $this->getYearRevenue(date('Y') - 1),
            'pending_invoices' => $this->getPendingInvoicesTotal(),
            'outstanding_invoices' => $this->getOutstandingAmount(),
            'recent_transactions' => $this->getRecentTransactions(),
            'revenue_by_gateway' => $this->getRevenueByGateway(),
            'monthly_comparison' => $this->getMonthlyComparison(),
            'projected_monthly' => $this->getProjectedMonthlyRevenue(),
        ];

        return $data;
    }

    /**
     * Generate widget output
     *
     * @param array $data Widget data
     * @return string
     */
    public function generateOutput(array $data): string
    {
        $growthClass = $this->calculateGrowthClass($data);
        $trendIcon = $this->getTrendIcon($data);

        return <<<HTML
<div class="widget-content-padded">
    <div class="row">
        <div class="col-sm-6">
            <div class="revenue-metric">
                <span class="revenue-label">Today's Revenue</span>
                <span class="revenue-value">{$data['today']['amount']}</span>
                <span class="revenue-count">{$data['today']['count']} transactions</span>
            </div>
        </div>
        <div class="col-sm-6">
            <div class="revenue-metric">
                <span class="revenue-label">This Month</span>
                <span class="revenue-value">{$data['this_month']['amount']}</span>
                <span class="revenue-growth {$growthClass}">
                    {$trendIcon} {$data['this_month']['growth']}% vs last month
                </span>
            </div>
        </div>
    </div>

    <div class="row">
        <div class="col-sm-6">
            <div class="revenue-metric">
                <span class="revenue-label">Outstanding</span>
                <span class="revenue-value warning">{$data['outstanding_invoices']}</span>
                <span class="revenue-count">Pending invoices: {$data['pending_invoices']['count']}</span>
            </div>
        </div>
        <div class="col-sm-6">
            <div class="revenue-metric">
                <span class="revenue-label">Projected (Month)</span>
                <span class="revenue-value">{$data['projected_monthly']}</span>
                <span class="revenue-count">Based on current trends</span>
            </div>
        </div>
    </div>

    <hr>

    <div class="revenue-breakdown">
        <h5>Revenue by Payment Gateway</h5>
        <div class="gateway-list">
            {$this->renderGatewayList($data['revenue_by_gateway'])}
        </div>
    </div>

    <hr>

    <div class="recent-transactions">
        <h5>Recent Transactions</h5>
        <table class="table table-striped">
            <thead>
                <tr>
                    <th>Date</th>
                    <th>Description</th>
                    <th class="text-right">Amount</th>
                </tr>
            </thead>
            <tbody>
                {$this->renderRecentTransactions($data['recent_transactions'])}
            </tbody>
        </table>
    </div>
</div>

<style>
.revenue-metric {
    text-align: center;
    padding: 10px;
}
.revenue-label {
    display: block;
    font-size: 11px;
    color: #777;
    text-transform: uppercase;
}
.revenue-value {
    display: block;
    font-size: 20px;
    font-weight: bold;
    color: #2a2a2a;
}
.revenue-value.warning {
    color: #e67e22;
}
.revenue-count {
    display: block;
    font-size: 11px;
    color: #999;
}
.revenue-growth {
    font-size: 11px;
}
.revenue-growth.positive { color: #27ae60; }
.revenue-growth.negative { color: #e74c3c; }
.gateway-list {
    display: flex;
    flex-wrap: wrap;
    gap: 10px;
}
.gateway-badge {
    padding: 5px 10px;
    background: #f5f5f5;
    border-radius: 4px;
    font-size: 12px;
}
</style>
HTML;
    }

    /**
     * Get today's revenue
     */
    protected function getTodayRevenue(): array
    {
        $today = date('Y-m-d');
        
        $result = Capsule::table('tblaccounts')
            ->where('date', $today)
            ->selectRaw('SUM(amountin) as total, COUNT(*) as count')
            ->first();

        return [
            'amount' => formatCurrency($result->total ?? 0),
            'count' => $result->count ?? 0,
        ];
    }

    /**
     * Get yesterday's revenue
     */
    protected function getYesterdayRevenue(): array
    {
        $yesterday = date('Y-m-d', strtotime('-1 day'));
        
        $result = Capsule::table('tblaccounts')
            ->where('date', $yesterday)
            ->selectRaw('SUM(amountin) as total, COUNT(*) as count')
            ->first();

        return [
            'amount' => formatCurrency($result->total ?? 0),
            'count' => $result->count ?? 0,
        ];
    }

    /**
     * Get monthly revenue
     */
    protected function getMonthRevenue(int $month, int $year): array
    {
        $startDate = date('Y-m-d', strtotime("{$year}-{$month}-01"));
        $endDate = date('Y-m-t', strtotime($startDate));

        $result = Capsule::table('tblaccounts')
            ->whereBetween('date', [$startDate, $endDate])
            ->selectRaw('SUM(amountin) as total, COUNT(*) as count')
            ->first();

        $total = $result->total ?? 0;
        
        // Calculate growth vs previous month
        $prevMonth = date('m', strtotime('-1 month', strtotime($startDate)));
        $prevYear = date('Y', strtotime('-1 month', strtotime($startDate)));
        $prevResult = $this->getMonthRevenue($prevMonth, $prevYear);
        
        $growth = 0;
        if ($prevResult['total'] > 0) {
            $growth = round((($total - $prevResult['total']) / $prevResult['total']) * 100, 1);
        }

        return [
            'amount' => formatCurrency($total),
            'count' => $result->count ?? 0,
            'total' => $total,
            'growth' => $growth,
        ];
    }

    /**
     * Get yearly revenue
     */
    protected function getYearRevenue(int $year): array
    {
        $startDate = "{$year}-01-01";
        $endDate = "{$year}-12-31";

        $result = Capsule::table('tblaccounts')
            ->whereBetween('date', [$startDate, $endDate])
            ->selectRaw('SUM(amountin) as total, COUNT(*) as count')
            ->first();

        return [
            'amount' => formatCurrency($result->total ?? 0),
            'count' => $result->count ?? 0,
            'total' => $result->total ?? 0,
        ];
    }

    /**
     * Get pending invoices total
     */
    protected function getPendingInvoicesTotal(): array
    {
        $result = Capsule::table('tblinvoices')
            ->whereIn('status', ['Unpaid', 'Overdue'])
            ->selectRaw('SUM(total) as total, COUNT(*) as count')
            ->first();

        return [
            'amount' => formatCurrency($result->total ?? 0),
            'count' => $result->count ?? 0,
            'total' => $result->total ?? 0,
        ];
    }

    /**
     * Get outstanding amount
     */
    protected function getOutstandingAmount(): string
    {
        $result = Capsule::table('tblinvoices')
            ->whereIn('status', ['Unpaid', 'Overdue'])
            ->sum('total');

        return formatCurrency($result ?? 0);
    }

    /**
     * Get recent transactions
     */
    protected function getRecentTransactions(): array
    {
        return Capsule::table('tblaccounts')
            ->join('tblclients', 'tblaccounts.userid', '=', 'tblclients.id')
            ->orderBy('date', 'desc')
            ->limit(5)
            ->get([
                'tblaccounts.id',
                'tblaccounts.date',
                'tblaccounts.description',
                'tblaccounts.amountin',
                'tblaccounts.gateway',
                Capsule::raw("CONCAT(tblclients.firstname, ' ', tblclients.lastname) as client_name"),
            ]);
    }

    /**
     * Get revenue by payment gateway
     */
    protected function getRevenueByGateway(): array
    {
        $startOfMonth = date('Y-m-01');
        
        return Capsule::table('tblaccounts')
            ->where('date', '>=', $startOfMonth)
            ->where('amountin', '>', 0)
            ->groupBy('gateway')
            ->selectRaw('gateway, SUM(amountin) as total')
            ->get();
    }

    /**
     * Get monthly comparison data
     */
    protected function getMonthlyComparison(): array
    {
        $data = [];
        
        for ($i = 5; $i >= 0; $i--) {
            $month = date('m', strtotime("-{$i} months"));
            $year = date('Y', strtotime("-{$i} months"));
            $revenue = $this->getMonthRevenue($month, $year);
            
            $data[] = [
                'month' => date('M Y', strtotime("{$year}-{$month}-01")),
                'total' => $revenue['total'],
            ];
        }
        
        return $data;
    }

    /**
     * Get projected monthly revenue
     */
    protected function getProjectedMonthlyRevenue(): string
    {
        $dayOfMonth = (int)date('j');
        $monthTotal = Capsule::table('tblaccounts')
            ->where('date', '>=', date('Y-m-01'))
            ->where('amountin', '>', 0)
            ->sum('amountin');

        if ($dayOfMonth > 0) {
            $projected = ($monthTotal / $dayOfMonth) * date('t');
            return formatCurrency($projected);
        }

        return formatCurrency(0);
    }

    /**
     * Calculate growth CSS class
     */
    protected function calculateGrowthClass(array $data): string
    {
        $growth = $data['this_month']['growth'] ?? 0;
        
        if ($growth > 0) {
            return 'positive';
        } elseif ($growth < 0) {
            return 'negative';
        }
        
        return '';
    }

    /**
     * Get trend icon
     */
    protected function getTrendIcon(array $data): string
    {
        $growth = $data['this_month']['growth'] ?? 0;
        
        if ($growth > 0) {
            return '<i class="fa fa-arrow-up"></i>';
        } elseif ($growth < 0) {
            return '<i class="fa fa-arrow-down"></i>';
        }
        
        return '<i class="fa fa-minus"></i>';
    }

    /**
     * Render gateway list
     */
    protected function renderGatewayList(array $gateways): string
    {
        $html = '';
        
        foreach ($gateways as $gateway) {
            $name = $gateway->gateway ?: 'Unknown';
            $total = formatCurrency($gateway->total);
            $html .= "<span class=\"gateway-badge\">{$name}: {$total}</span>";
        }
        
        return $html ?: '<span class="text-muted">No data available</span>';
    }

    /**
     * Render recent transactions
     */
    protected function renderRecentTransactions(array $transactions): string
    {
        $html = '';
        
        foreach ($transactions as $tx) {
            $date = date('M j', strtotime($tx->date));
            $amount = formatCurrency($tx->amountin);
            $description = htmlspecialchars($tx->description ?: $tx->client_name);
            
            $html .= "<tr>
                <td>{$date}</td>
                <td>{$description}</td>
                <td class=\"text-right text-success\">{$amount}</td>
            </tr>";
        }
        
        return $html ?: '<tr><td colspan="3" class="text-center text-muted">No recent transactions</td></tr>';
    }

    /**
     * Get widget identity
     */
    public function getId(): string
    {
        return 'revenue_dashboard_widget';
    }

    /**
     * Get widget name
     */
    public function getName(): string
    {
        return $this->title;
    }
}
```

## Activation & Deactivation

```php
<?php
function whmcs_revenue_dashboard_widget_activate(): array
{
    try {
        return [
            'status' => 'success',
            'description' => 'Revenue Dashboard Widget activated successfully',
        ];
    } catch (\Exception $e) {
        return [
            'status' => 'error',
            'description' => 'Failed to activate widget: ' . $e->getMessage(),
        ];
    }
}

function whmcs_revenue_dashboard_widget_deactivate(): array
{
    try {
        return [
            'status' => 'success',
            'description' => 'Revenue Dashboard Widget deactivated',
        ];
    } catch (\Exception $e) {
        return [
            'status' => 'error',
            'description' => 'Failed to deactivate widget: ' . $e->getMessage(),
        ];
    }
}
```

## Widget Interface Implementation

The widget implements `WHMCS\Module\Contracts\WidgetModuleInterface` which requires:

| Method | Description |
|--------|-------------|
| getData() | Fetch and return widget data |
| generateOutput(array $data) | Generate HTML output |
| getId() | Return unique widget identifier |
| getName() | Return display name |

## Configuration Options

```php
// In config.php
return [
    'refresh_interval' => 300, // seconds
    'date_range' => 'this_month',
    'show_projections' => true,
    'show_gateway_breakdown' => true,
    'transaction_limit' => 5,
];
```

## Database Schema

```php
<?php
use WHMCS\Database\Capsule;

if (!Capsule::schema()->hasTable('mod_revenue_widget_cache')) {
    Capsule::schema()->create('mod_revenue_widget_cache', function ($table) {
        $table->string('cache_key', 100)->primary();
        $table->longText('cache_data')->nullable();
        $table->timestamp('cached_at')->useCurrent();
        $table->integer('ttl_seconds')->default(300);
    });
}
```

## Version History

| Version | Date | Changes |
|---------|------|---------|
| 1.0.0 | 2024-01-15 | Initial release |

## Requirements

- WHMCS 8.0.0+
- PHP 7.4+
