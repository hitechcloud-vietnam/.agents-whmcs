# WHMCS Payment Analytics Workflow

## Description
Set up payment analytics and reporting for WHMCS.

## Prerequisites
- WHMCS 7.0+
- Multiple payment gateways (optional)
- Analytics platform

## Steps

### Step 1: Create Payment Analytics Service
```php
<?php
/**
 * Payment Analytics Service
 */

class PaymentAnalytics
{
    /**
     * Get payment summary for period
     */
    public function getSummary($startDate, $endDate)
    {
        $payments = Capsule::table('tblaccounts')
            ->where('date', '>=', $startDate)
            ->where('date', '<=', $endDate)
            ->where('amount', '>', 0)
            ->get();
        
        $totalRevenue = $payments->sum('amount');
        $transactionCount = $payments->count();
        $averageTransaction = $transactionCount > 0 ? $totalRevenue / $transactionCount : 0;
        
        return [
            'total_revenue' => $totalRevenue,
            'transaction_count' => $transactionCount,
            'average_transaction' => $averageTransaction,
        ];
    }
    
    /**
     * Revenue by gateway
     */
    public function getRevenueByGateway($startDate, $endDate)
    {
        return Capsule::table('tblaccounts')
            ->where('date', '>=', $startDate)
            ->where('date', '<=', $endDate)
            ->where('amount', '>', 0)
            ->selectRaw('gateway, SUM(amount) as total, COUNT(*) as count')
            ->groupBy('gateway')
            ->get();
    }
    
    /**
     * Revenue trends
     */
    public function getTrends($startDate, $endDate, $groupBy = 'day')
    {
        $dateFormat = match($groupBy) {
            'day' => '%Y-%m-%d',
            'week' => '%Y-%u',
            'month' => '%Y-%m',
            default => '%Y-%m-%d',
        };
        
        return Capsule::table('tblaccounts')
            ->where('date', '>=', $startDate)
            ->where('date', '<=', $endDate)
            ->where('amount', '>', 0)
            ->selectRaw("DATE_FORMAT(date, '$dateFormat') as period, SUM(amount) as revenue, COUNT(*) as transactions")
            ->groupByRaw("DATE_FORMAT(date, '$dateFormat')")
            ->orderBy('period')
            ->get();
    }
    
    /**
     * Payment success rate
     */
    public function getSuccessRate($startDate, $endDate)
    {
        $successful = Capsule::table('tblaccounts')
            ->where('date', '>=', $startDate)
            ->where('date', '<=', $endDate)
            ->where('amount', '>', 0)
            ->count();
        
        $total = $successful + Capsule::table('mod_payment_attempts')
            ->where('created_at', '>=', $startDate)
            ->where('created_at', '<=', $endDate)
            ->where('status', 'failed')
            ->count();
        
        return $total > 0 ? ($successful / $total) * 100 : 0;
    }
    
    /**
     * Revenue by client segment
     */
    public function getRevenueBySegment($startDate, $endDate)
    {
        return Capsule::table('tblaccounts')
            ->join('tblclients', 'tblaccounts.userid', '=', 'tblclients.id')
            ->where('tblaccounts.date', '>=', $startDate)
            ->where('tblaccounts.date', '<=', $endDate)
            ->where('tblaccounts.amount', '>', 0)
            ->selectRaw("
                CASE 
                    WHEN tblclients.date >= DATE_SUB(NOW(), INTERVAL 30 DAY) THEN 'New'
                    WHEN tblclients.date >= DATE_SUB(NOW(), INTERVAL 180 DAY) THEN 'Recent'
                    ELSE 'Established'
                END as segment,
                SUM(tblaccounts.amount) as revenue,
                COUNT(*) as transactions
            ")
            ->groupBy('segment')
            ->get();
    }
}
```

### Step 2: Create Analytics Dashboard
```php
<?php
// modules/addons/analytics/analytics.php

function analytics_dashboard($vars)
{
    $analytics = new PaymentAnalytics();
    
    $period = $_GET['period'] ?? 'month';
    $startDate = match($period) {
        'day' => date('Y-m-d'),
        'week' => date('Y-m-d', strtotime('-7 days')),
        'month' => date('Y-m-d', strtotime('-30 days')),
        'quarter' => date('Y-m-d', strtotime('-90 days')),
        'year' => date('Y-m-d', strtotime('-365 days')),
        default => date('Y-m-d', strtotime('-30 days')),
    };
    
    $endDate = date('Y-m-d');
    
    return [
        'summary' => $analytics->getSummary($startDate, $endDate),
        'by_gateway' => $analytics->getRevenueByGateway($startDate, $endDate),
        'trends' => $analytics->getTrends($startDate, $endDate),
        'success_rate' => $analytics->getSuccessRate($startDate, $endDate),
        'by_segment' => $analytics->getRevenueBySegment($startDate, $endDate),
        'period' => $period,
    ];
}
```

### Step 3: Create Charts
```php
<?php
function renderRevenueChart($data)
{
    $labels = [];
    $values = [];
    
    foreach ($data as $row) {
        $labels[] = $row->period;
        $values[] = $row->revenue;
    }
    
    return [
        'type' => 'line',
        'data' => [
            'labels' => $labels,
            'datasets' => [[
                'label' => 'Revenue',
                'data' => $values,
                'borderColor' => '#36a64f',
                'fill' => true,
            ]],
        ],
        'options' => [
            'responsive' => true,
            'scales' => [
                'y' => ['beginAtZero' => true],
            ],
        ],
    ];
}

function renderGatewayChart($data)
{
    $labels = [];
    $values = [];
    $colors = ['#0074D9', '#FFDC00', '#FF4136', '#2ECC40', '#B10DC9'];
    
    foreach ($data as $i => $row) {
        $labels[] = $row->gateway ?: 'Unknown';
        $values[] = $row->total;
    }
    
    return [
        'type' => 'doughnut',
        'data' => [
            'labels' => $labels,
            'datasets' => [[
                'data' => $values,
                'backgroundColor' => array_slice($colors, 0, count($labels)),
            ]],
        ],
    ];
}
```

## Key Metrics
| Metric | Description |
|--------|-------------|
| Revenue | Total payment amount |
| Transaction Count | Number of successful payments |
| Average Transaction Value | Revenue / Transactions |
| Success Rate | Successful / Total attempts |
| Refund Rate | Refunds / Revenue |
| Gateway Mix | Revenue by payment method |
| Customer LTV | Lifetime value by segment |

## Tags
- analytics
- reporting
- payment
- metrics
- dashboard