# WHMCS Metrics Visualization Workflow

## Purpose

Implement comprehensive metrics dashboards and data visualization for WHMCS to track business performance, monitor key metrics, and enable data-driven decision making. This workflow covers dashboard creation, metric calculation, visualization implementation, and reporting automation.

## Prerequisites

- WHMCS v8.0+
- Admin access to WHMCS
- PHP 8.0+
- Optional: Charting library (Chart.js, D3.js, or similar)

## Workflow Steps

### Step 1: Metrics Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                  Metrics Visualization Framework                 │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  Data Sources:                                                   │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐          │
│  │   WHMCS DB  │  │  External    │  │   Custom     │          │
│  │   Tables    │  │  APIs        │  │   Data       │          │
│  └──────────────┘  └──────────────┘  └──────────────┘          │
│                                                                  │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐          │
│  │   Business  │  │   Customer   │  │   Operational│          │
│  │   Metrics   │  │   Metrics    │  │   Metrics    │          │
│  ├──────────────┤  ├──────────────┤  ├──────────────┤          │
│  │ • MRR/ARR   │  │ • Churn Rate │  │ • Server     │          │
│  │ • Revenue   │  │ • LTV        │  │ • API Latency│          │
│  │ • Growth    │  │ • NPS        │  │ • Uptime     │          │
│  └──────────────┘  └──────────────┘  └──────────────┘          │
│                                                                  │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │                    Visualization Layer                    │  │
│  │  Dashboards ──▶ Reports ──▶ Alerts ──▶ Exports           │  │
│  └──────────────────────────────────────────────────────────┘  │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### Step 2: Metrics Calculation Engine

```php
<?php
// /var/www/html/whmcs/includes/helpers/MetricsCalculator.php

namespace WHMCS\Helpers;

use Illuminate\Database\Capsule\Manager as Capsule;

class MetricsCalculator
{
    /**
     * Calculate all key business metrics
     */
    public static function calculateAllMetrics(string $period = '30 days'): array
    {
        return [
            'revenue' => self::calculateRevenueMetrics($period),
            'customers' => self::calculateCustomerMetrics($period),
            'services' => self::calculateServiceMetrics($period),
            'support' => self::calculateSupportMetrics($period),
            'growth' => self::calculateGrowthMetrics($period),
        ];
    }

    /**
     * Calculate revenue metrics
     */
    public static function calculateRevenueMetrics(string $period = '30 days'): array
    {
        $startDate = date('Y-m-d', strtotime("-{$period}"));
        $endDate = date('Y-m-d');

        // Total revenue
        $totalRevenue = Capsule::table('tblinvoices')
            ->where('status', 'Paid')
            ->whereBetween('datepaid', [$startDate, $endDate])
            ->sum('total');

        // Revenue by day
        $dailyRevenue = Capsule::select("
            SELECT 
                DATE(datepaid) as date,
                SUM(total) as revenue
            FROM tblinvoices
            WHERE status = 'Paid'
            AND datepaid BETWEEN ? AND ?
            GROUP BY DATE(datepaid)
            ORDER BY date
        ", [$startDate, $endDate]);

        // Revenue by payment method
        $revenueByMethod = Capsule::select("
            SELECT 
                p.gateway,
                SUM(i.total) as revenue,
                COUNT(*) as transactions
            FROM tblinvoices i
            LEFT JOIN tblpaymentgateways p ON i.paymentmethod = p.gateway
            WHERE i.status = 'Paid'
            AND i.datepaid BETWEEN ? AND ?
            GROUP BY p.gateway
        ", [$startDate, $endDate]);

        // Average invoice value
        $avgInvoiceValue = Capsule::table('tblinvoices')
            ->where('status', 'Paid')
            ->whereBetween('datepaid', [$startDate, $endDate])
            ->avg('total');

        return [
            'total_revenue' => round($totalRevenue, 2),
            'avg_invoice_value' => round($avgInvoiceValue, 2),
            'daily_revenue' => $dailyRevenue,
            'by_payment_method' => $revenueByMethod,
            'currency' => Capsule::table('tblcurrencies')->where('default', 1)->value('code') ?? 'USD'
        ];
    }

    /**
     * Calculate customer metrics
     */
    public static function calculateCustomerMetrics(string $period = '30 days'): array
    {
        $startDate = date('Y-m-d', strtotime("-{$period}"));

        // New customers
        $newCustomers = Capsule::table('tblclients')
            ->where('createdat', '>=', $startDate)
            ->count();

        // Active customers
        $activeCustomers = Capsule::table('tblclients')
            ->where('status', 'Active')
            ->count();

        // Customer status breakdown
        $statusBreakdown = Capsule::select("
            SELECT status, COUNT(*) as count
            FROM tblclients
            GROUP BY status
        ");

        // Average customer lifetime value
        $avgLTV = Capsule::select("
            SELECT 
                userid,
                SUM(total) as total_spent
            FROM tblinvoices
            WHERE status = 'Paid'
            GROUP BY userid
            HAVING COUNT(*) > 0
        ");

        $avgLTVValue = count($avgLTV) > 0 
            ? array_sum(array_column($avgLTV, 'total_spent')) / count($avgLTV)
            : 0;

        return [
            'new_customers' => $newCustomers,
            'total_active' => $activeCustomers,
            'status_breakdown' => $statusBreakdown,
            'avg_lifetime_value' => round($avgLTVValue, 2),
            'customer_growth_rate' => self::calculateCustomerGrowthRate($startDate)
        ];
    }

    /**
     * Calculate customer growth rate
     */
    private static function calculateCustomerGrowthRate(string $startDate): float
    {
        $previousPeriodStart = date('Y-m-d', strtotime($startDate . ' -30 days'));

        $currentNew = Capsule::table('tblclients')
            ->where('createdat', '>=', $startDate)
            ->count();

        $previousNew = Capsule::table('tblclients')
            ->whereBetween('createdat', [$previousPeriodStart, $startDate])
            ->count();

        if ($previousNew === 0) {
            return $currentNew > 0 ? 100 : 0;
        }

        return round((($currentNew - $previousNew) / $previousNew) * 100, 2);
    }

    /**
     * Calculate service metrics
     */
    public static function calculateServiceMetrics(string $period = '30 days'): array
    {
        $startDate = date('Y-m-d', strtotime("-{$period}"));

        // Service counts by status
        $statusCounts = Capsule::select("
            SELECT 
                domainstatus,
                COUNT(*) as count
            FROM tblhosting
            GROUP BY domainstatus
        ");

        // New services
        $newServices = Capsule::table('tblhosting')
            ->where('regdate', '>=', $startDate)
            ->count();

        // Terminated services
        $terminatedServices = Capsule::table('tblhosting')
            ->where('domainstatus', 'Terminated')
            ->where('termination_date', '>=', $startDate)
            ->count();

        // Upsells/Downgrades
        $changes = Capsule::table('tblhosting')
            ->where('last_update', '>=', $startDate)
            ->count();

        // Revenue per product
        $revenueByProduct = Capsule::select("
            SELECT 
                p.name as product_name,
                COUNT(h.id) as subscriptions,
                SUM(CASE WHEN h.billingcycle = 1 THEN h.amount
                         WHEN h.billingcycle = 2 THEN h.amount / 3
                         WHEN h.billingcycle = 3 THEN h.amount / 6
                         WHEN h.billingcycle = 4 THEN h.amount / 12
                         ELSE h.amount END) as mrr
            FROM tblhosting h
            INNER JOIN tblproducts p ON h.packageid = p.id
            WHERE h.domainstatus = 'Active'
            GROUP BY p.id, p.name
            ORDER BY mrr DESC
        ");

        return [
            'new_services' => $newServices,
            'terminated_services' => $terminatedServices,
            'service_changes' => $changes,
            'status_breakdown' => $statusCounts,
            'revenue_by_product' => $revenueByProduct,
            'net_service_growth' => $newServices - $terminatedServices
        ];
    }

    /**
     * Calculate support metrics
     */
    public static function calculateSupportMetrics(string $period = '30 days'): array
    {
        $startDate = date('Y-m-d', strtotime("-{$period}"));

        // Ticket counts by status
        $ticketStatus = Capsule::select("
            SELECT 
                status,
                COUNT(*) as count
            FROM tbltickets
            WHERE created >= ?
            GROUP BY status
        ", [$startDate]);

        // Average response time
        $avgResponseTime = Capsule::select("
            SELECT 
                AVG(TIMESTAMPDIFF(HOUR, created, first_reply)) as avg_hours
            FROM tbltickets
            WHERE created >= ?
            AND first_reply IS NOT NULL
        ", [$startDate])[0]->avg_hours ?? 0;

        // Tickets by department
        $ticketsByDept = Capsule::select("
            SELECT 
                d.name as department,
                COUNT(t.id) as tickets,
                AVG(TIMESTAMPDIFF(HOUR, t.created, t.first_reply)) as avg_response
            FROM tbltickets t
            INNER JOIN tblticketdepartments d ON t.deptid = d.id
            WHERE t.created >= ?
            GROUP BY d.id, d.name
        ", [$startDate]);

        // Satisfaction scores
        $satisfaction = Capsule::select("
            SELECT 
                AVG(rating) as avg_rating,
                COUNT(*) as responses
            FROM tblticketfeedback
            WHERE created >= ?
        ", [$startDate])[0] ?? ['avg_rating' => 0, 'responses' => 0];

        return [
            'tickets_by_status' => $ticketStatus,
            'total_tickets' => array_sum(array_column($ticketStatus, 'count')),
            'avg_response_hours' => round($avgResponseTime, 1),
            'tickets_by_department' => $ticketsByDept,
            'avg_satisfaction' => round($satisfaction->avg_rating ?? 0, 2),
            'feedback_responses' => $satisfaction->responses ?? 0
        ];
    }

    /**
     * Calculate growth metrics
     */
    public static function calculateGrowthMetrics(string $period = '30 days'): array
    {
        $startDate = date('Y-m-d', strtotime("-{$period}"));

        // MRR calculation
        $activeServices = Capsule::table('tblhosting')
            ->where('domainstatus', 'Active')
            ->get();

        $mrr = 0;
        foreach ($activeServices as $service) {
            $amount = (float) $service->amount;
            switch ($service->billingcycle) {
                case 1: $mrr += $amount; break;
                case 2: $mrr += $amount / 3; break;
                case 3: $mrr += $amount / 6; break;
                case 4: $mrr += $amount / 12; break;
                case 5: $mrr += $amount / 24; break;
                case 6: $mrr += $amount / 36; break;
                default: $mrr += $amount;
            }
        }

        // Month over month growth
        $previousMonth = date('Y-m-01', strtotime('-1 month'));
        $currentMonth = date('Y-m-01');

        $currentMRR = $mrr;
        $previousMRR = self::calculateHistoricalMRR($previousMonth);

        $mrrGrowth = $previousMRR > 0 
            ? (($currentMRR - $previousMRR) / $previousMRR) * 100 
            : 0;

        return [
            'mrr' => round($mrr, 2),
            'arr' => round($mrr * 12, 2),
            'mrr_growth_percent' => round($mrrGrowth, 2),
            'previous_mrr' => round($previousMRR, 2),
            'mrr_change' => round($currentMRR - $previousMRR, 2)
        ];
    }

    /**
     * Calculate historical MRR
     */
    private static function calculateHistoricalMRR(string $date): float
    {
        $services = Capsule::table('tblhosting')
            ->where('regdate', '<=', $date)
            ->where('domainstatus', 'Active')
            ->get();

        $mrr = 0;
        foreach ($services as $service) {
            $amount = (float) $service->amount;
            switch ($service->billingcycle) {
                case 1: $mrr += $amount; break;
                case 2: $mrr += $amount / 3; break;
                case 3: $mrr += $amount / 6; break;
                case 4: $mrr += $amount / 12; break;
                default: $mrr += $amount;
            }
        }

        return $mrr;
    }
}
```

### Step 3: Dashboard Widget Implementation

```php
<?php
// /var/www/html/whmcs/modules/widgets/MetricsDashboard.php

namespace WHMCS\Module\Widget;

class MetricsDashboard extends \WHMCS\Module\AbstractWidget
{
    protected $title = 'Business Metrics';
    protected $author = 'System';
    protected $type = 'dashboard';

    public function getData(): array
    {
        $calculator = new \WHMCS\Helpers\MetricsCalculator();

        return [
            'revenue' => $calculator->calculateRevenueMetrics('30 days'),
            'customers' => $calculator->calculateCustomerMetrics('30 days'),
            'services' => $calculator->calculateServiceMetrics('30 days'),
            'growth' => $calculator->calculateGrowthMetrics('30 days'),
            'trend_data' => $this->getTrendData(),
            'generated_at' => date('Y-m-d H:i:s')
        ];
    }

    private function getTrendData(int $days = 30): array
    {
        $data = [];

        for ($i = $days - 1; $i >= 0; $i--) {
            $date = date('Y-m-d', strtotime("-{$i} days"));

            $revenue = \Illuminate\Database\Capsule\Manager::table('tblinvoices')
                ->where('status', 'Paid')
                ->whereDate('datepaid', $date)
                ->sum('total');

            $newCustomers = \Illuminate\Database\Capsule\Manager::table('tblclients')
                ->whereDate('createdat', $date)
                ->count();

            $data[] = [
                'date' => $date,
                'revenue' => round($revenue, 2),
                'new_customers' => $newCustomers
            ];
        }

        return $data;
    }

    public function generateOutput($data): string
    {
        $revenue = $data['revenue'];
        $growth = $data['growth'];
        $customers = $data['customers'];

        $growthClass = $growth['mrr_growth_percent'] >= 0 ? 'positive' : 'negative';
        $growthSign = $growth['mrr_growth_percent'] >= 0 ? '+' : '';

        return <<<HTML
<div class="metrics-dashboard">
    <div class="metrics-grid">
        <div class="metric-card revenue">
            <div class="metric-value">\${$this->formatNumber($growth['mrr'])}</div>
            <div class="metric-label">Monthly Recurring Revenue</div>
            <div class="metric-change {$growthClass}">
                {$growthSign}{$growth['mrr_growth_percent']}% vs last month
            </div>
        </div>

        <div class="metric-card customers">
            <div class="metric-value">{$customers['total_active']}</div>
            <div class="metric-label">Active Customers</div>
            <div class="metric-change positive">
                +{$customers['new_customers']} this month
            </div>
        </div>

        <div class="metric-card services">
            <div class="metric-value">{$data['services']['new_services']}</div>
            <div class="metric-label">New Services</div>
            <div class="metric-change negative">
                -{$data['services']['terminated_services']} terminated
            </div>
        </div>

        <div class="metric-card invoices">
            <div class="metric-value">\${$this->formatNumber($revenue['avg_invoice_value'])}</div>
            <div class="metric-label">Avg Invoice Value</div>
            <div class="metric-change neutral">
                {$revenue['by_payment_method'][0]->gateway ?? 'N/A'}
            </div>
        </div>
    </div>

    <div class="trend-chart" id="metrics-trend-chart">
        <canvas id="trendChart"></canvas>
    </div>
</div>

<script>
document.addEventListener('DOMContentLoaded', function() {
    const trendData = {$this->jsonEncode($data['trend_data'])};
    const ctx = document.getElementById('trendChart').getContext('2d');

    new Chart(ctx, {
        type: 'line',
        data: {
            labels: trendData.map(d => d.date),
            datasets: [{
                label: 'Revenue',
                data: trendData.map(d => d.revenue),
                borderColor: '#007bff',
                fill: false
            }]
        },
        options: {
            responsive: true,
            maintainAspectRatio: false,
            scales: {
                y: { beginAtZero: true }
            }
        }
    });
});
</script>
HTML;
    }

    private function formatNumber(float $num): string
    {
        if ($num >= 1000000) {
            return number_format($num / 1000000, 1) . 'M';
        } elseif ($num >= 1000) {
            return number_format($num / 1000, 1) . 'K';
        }
        return number_format($num, 2);
    }

    private function jsonEncode(array $data): string
    {
        return json_encode($data, JSON_HEX_APOS);
    }
}
```

### Step 4: Custom Reports Implementation

```php
<?php
// /var/www/html/whmcs/includes/helpers/ReportGenerator.php

namespace WHMCS\Helpers;

class ReportGenerator
{
    /**
     * Generate executive summary report
     */
    public static function generateExecutiveSummary(): array
    {
        $calculator = new MetricsCalculator();
        $metrics = $calculator->calculateAllMetrics('30 days');

        return [
            'period' => 'Last 30 Days',
            'generated_at' => date('Y-m-d H:i:s'),
            'summary' => [
                'revenue' => [
                    'total' => $metrics['revenue']['total_revenue'],
                    'mrr' => $metrics['growth']['mrr'],
                    'growth' => $metrics['growth']['mrr_growth_percent']
                ],
                'customers' => [
                    'active' => $metrics['customers']['total_active'],
                    'new' => $metrics['customers']['new_customers'],
                    'avg_ltv' => $metrics['customers']['avg_lifetime_value']
                ],
                'services' => [
                    'active' => self::getActiveServiceCount(),
                    'new' => $metrics['services']['new_services'],
                    'growth' => $metrics['services']['net_service_growth']
                ]
            ],
            'recommendations' => self::generateRecommendations($metrics)
        ];
    }

    /**
     * Generate detailed revenue report
     */
    public static function generateRevenueReport(array $params): array
    {
        $startDate = $params['start_date'] ?? date('Y-m-01');
        $endDate = $params['end_date'] ?? date('Y-m-t');
        $groupBy = $params['group_by'] ?? 'day';

        $data = Capsule::select("
            SELECT 
                {$this->getDateGroup($groupBy)} as period,
                SUM(total) as revenue,
                COUNT(*) as invoices,
                AVG(total) as avg_value
            FROM tblinvoices
            WHERE status = 'Paid'
            AND datepaid BETWEEN ? AND ?
            GROUP BY {$this->getDateGroup($groupBy)}
            ORDER BY period
        ", [$startDate, $endDate]);

        return [
            'start_date' => $startDate,
            'end_date' => $endDate,
            'group_by' => $groupBy,
            'total_revenue' => array_sum(array_column($data, 'revenue')),
            'total_invoices' => array_sum(array_column($data, 'invoices')),
            'avg_invoice_value' => count($data) > 0 
                ? array_sum(array_column($data, 'avg_value')) / count($data)
                : 0,
            'data' => $data
        ];
    }

    private function getDateGroup(string $groupBy): string
    {
        switch ($groupBy) {
            case 'day':
                return 'DATE(datepaid)';
            case 'week':
                return 'YEARWEEK(datepaid, 1)';
            case 'month':
                return 'DATE_FORMAT(datepaid, "%Y-%m")';
            default:
                return 'DATE(datepaid)';
        }
    }

    /**
     * Generate recommendations based on metrics
     */
    private static function generateRecommendations(array $metrics): array
    {
        $recommendations = [];

        // Growth recommendations
        if ($metrics['growth']['mrr_growth_percent'] < 5) {
            $recommendations[] = [
                'priority' => 'high',
                'area' => 'growth',
                'message' => 'MRR growth is below target. Consider upsell campaigns.'
            ];
        }

        // Churn recommendations
        $churnRate = $metrics['services']['terminated_services'] / max(1, $metrics['services']['new_services']);
        if ($churnRate > 0.2) {
            $recommendations[] = [
                'priority' => 'high',
                'area' => 'retention',
                'message' => 'Churn rate is elevated. Implement retention programs.'
            ];
        }

        // Support recommendations
        if ($metrics['support']['avg_response_hours'] > 4) {
            $recommendations[] = [
                'priority' => 'medium',
                'area' => 'support',
                'message' => 'Support response time can be improved.'
            ];
        }

        return $recommendations;
    }

    private function getActiveServiceCount(): int
    {
        return Capsule::table('tblhosting')
            ->where('domainstatus', 'Active')
            ->count();
    }
}
```

### Step 5: Automated Report Scheduling

```php
<?php
// /var/www/html/whmcs/includes/hooks/report_scheduling.php

/**
 * Daily metrics report generation
 */
add_hook('DailyCronJob', 1, function() {
    logActivity('Starting automated report generation');

    $generator = new \WHMCS\Helpers\ReportGenerator();

    // Generate daily summary
    $summary = $generator->generateExecutiveSummary();

    // Store in reporting table
    Capsule::table('tblreport_snapshots')->insert([
        'report_type' => 'daily_summary',
        'period_start' => date('Y-m-d', strtotime('-1 day')),
        'period_end' => date('Y-m-d'),
        'metrics' => json_encode($summary),
        'created_at' => date('Y-m-d H:i:s')
    ]);

    // Send email if critical metrics
    foreach ($summary['recommendations'] as $rec) {
        if ($rec['priority'] === 'high') {
            sendEmail('MetricsAlert', null, [
                'alert_message' => $rec['message'],
                'metrics' => $summary['summary']
            ]);
        }
    }

    logActivity('Report generation completed');
});

/**
 * Weekly executive report
 */
add_hook('WeeklyCronJob', 1, function() {
    $generator = new \WHMCS\Helpers\ReportGenerator();
    $report = $generator->generateExecutiveSummary();

    // Send to management
    sendEmail('WeeklyExecutiveReport', null, [
        'report' => $report,
        'week_start' => date('Y-m-d', strtotime('-7 days')),
        'week_end' => date('Y-m-d')
    ]);
});
```

### Step 6: Report Storage Schema

```sql
-- Create report snapshots table
CREATE TABLE IF NOT EXISTS `tblreport_snapshots` (
    `id` INT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    `report_type` VARCHAR(50) NOT NULL,
    `period_start` DATE,
    `period_end` DATE,
    `metrics` JSON,
    `created_at` TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    INDEX `idx_report_type` (`report_type`),
    INDEX `idx_period` (`period_start`, `period_end`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;

-- Create scheduled reports table
CREATE TABLE IF NOT EXISTS `tblscheduled_reports` (
    `id` INT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    `report_name` VARCHAR(100) NOT NULL,
    `report_type` VARCHAR(50) NOT NULL,
    `schedule` ENUM('daily', 'weekly', 'monthly') DEFAULT 'weekly',
    `recipients` JSON,
    `parameters` JSON,
    `is_active` TINYINT(1) DEFAULT 1,
    `last_run` TIMESTAMP NULL,
    `next_run` TIMESTAMP,
    `created_at` TIMESTAMP DEFAULT CURRENT_TIMESTAMP
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;

-- Create custom dashboards table
CREATE TABLE IF NOT EXISTS `tblcustom_dashboards` (
    `id` INT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    `user_id` INT UNSIGNED,
    `dashboard_name` VARCHAR(100) NOT NULL,
    `widgets` JSON,
    `layout` JSON,
    `is_default` TINYINT(1) DEFAULT 0,
    `created_at` TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    INDEX `idx_user_id` (`user_id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
```

## Metrics Categories

| Category | Metrics | Update Frequency |
|----------|---------|------------------|
| Revenue | MRR, ARR, Revenue Growth, Avg Invoice Value | Daily |
| Customers | Active Count, New, Churned, LTV, NPS | Daily |
| Services | Active, New, Renewals, Upgrades | Daily |
| Support | Tickets, Response Time, CSAT | Hourly |
| Operations | API Latency, Server Health, Uptime | Real-time |

## Best Practices

1. **Consistent Metrics**: Use same definitions across all reports
2. **Trend Analysis**: Always show comparison to previous period
3. **Alert Thresholds**: Set up alerts for metrics outside normal ranges
4. **Data Accuracy**: Validate metrics against source data regularly
5. **User Segmentation**: Tailor dashboards for different user roles
6. **Performance**: Cache metric calculations for faster loading

## Common Pitfalls

- **Too Many Metrics**: Focus on key performance indicators
- **No Context**: Always show comparison/benchmark
- **Stale Data**: Ensure timely data refresh
- **Poor Visualization**: Use appropriate chart types for data
- **Ignoring Anomalies**: Investigate unexpected changes

## Verification Checklist

- [ ] Metrics calculator implemented
- [ ] Dashboard widgets created
- [ ] Report generator functional
- [ ] Scheduled reports configured
- [ ] Alert thresholds set
- [ ] Export functionality added
- [ ] User dashboard preferences working

## Related Documentation

- [WHMCS Revenue Optimization](whmcs-revenue-optimization.md)
- [WHMCS Customer Retention](whmcs-customer-retention.md)
- [WHMCS Churn Reduction](whmcs-churn-reduction.md)