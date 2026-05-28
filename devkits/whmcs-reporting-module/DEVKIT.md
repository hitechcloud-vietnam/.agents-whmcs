# WHMCS Reporting Module DevKit
# Version: 1.0 | Updated: 2026-05-28

## DevKit Structure

```
devkits/whmcs-reporting-module/
├── reports/
│   └── {module}Report.php    # Report definition
├── lib/
│   ├── ReportBuilder.php     # Report builder class
│   └── ChartGenerator.php    # Chart generation class
└── templates/
    ├── report.tpl             # Report template
    └── export.tpl            # Export options template
```

## Report Definition Template

```php
<?php
/**
 * WHMCS Reporting Module: {Module}
 * DevKit Template
 * 
 * Installation: Copy to /includes/report/{module}_report.php
 */

if (!defined("WHMCS")) {
    die("Direct access denied");
}

use WHMCS\Database\Capsule;

/**
 * Report Definition
 */
function {module}_Report(): array
{
    return [
        'name' => '{Report Name}',
        'description' => '{Report description}',
        'version' => '1.0',
        'author' => '{Author}',
        'tables' => [
            'tblclients',
            'tblhosting',
            'tblinvoices',
            'tblorders',
        ],
    ];
}

/**
 * Report Output
 */
function {module}_Report_output(array $vars): string
{
    $report = new \{Module}\ReportBuilder();
    
    // Get filter parameters
    $dateFrom = $vars['date_from'] ?? date('Y-m-01');
    $dateTo = $vars['date_to'] ?? date('Y-m-d');
    $groupBy = $vars['group_by'] ?? 'day';
    $export = $vars['export'] ?? false;
    
    // Build report data
    $data = $report->getReportData($dateFrom, $dateTo, $groupBy);
    $summary = $report->getSummaryStats($dateFrom, $dateTo);
    $chartData = $report->getChartData($dateFrom, $dateTo, $groupBy);
    
    // Export mode
    if ($export) {
        return $report->exportToCSV($data);
    }
    
    // Render template
    $chartId = 'report_chart_' . time();
    
    return [
        'chartData' => $chartData,
        'chartId' => $chartId,
        'dateFrom' => $dateFrom,
        'dateTo' => $dateTo,
        'groupBy' => $groupBy,
        'data' => $data,
        'summary' => $summary,
    ];
}
```

## Report Builder Class

```php
<?php
namespace {Module};

use WHMCS\Database\Capsule;

class ReportBuilder {
    
    public function getReportData(string $dateFrom, string $dateTo, string $groupBy): array
    {
        $dateFormat = $this->getDateFormat($groupBy);
        
        $query = Capsule::table('tblorders')
            ->selectRaw("
                DATE_FORMAT(date, '{$dateFormat}') as period,
                COUNT(*) as total_orders,
                SUM(tblorders.amount) as total_amount,
                COUNT(DISTINCT userid) as unique_customers
            ")
            ->join('tblinvoices', 'tblorders.invoiceid', '=', 'tblinvoices.id')
            ->where('tblorders.status', '!=', 'Cancelled')
            ->whereBetween('tblorders.date', [$dateFrom, $dateTo])
            ->groupBy('period')
            ->orderBy('period', 'asc');
        
        return $query->get()->toArray();
    }
    
    public function getSummaryStats(string $dateFrom, string $dateTo): array
    {
        $totalOrders = Capsule::table('tblorders')
            ->where('status', '!=', 'Cancelled')
            ->whereBetween('date', [$dateFrom, $dateTo])
            ->count();
        
        $totalAmount = Capsule::table('tblinvoices')
            ->where('status', 'Paid')
            ->whereBetween('date', [$dateFrom, $dateTo])
            ->sum('total');
        
        $newClients = Capsule::table('tblclients')
            ->whereBetween('created_at', [$dateFrom . ' 00:00:00', $dateTo . ' 23:59:59'])
            ->count();
        
        $activeServices = Capsule::table('tblhosting')
            ->where('domainstatus', 'Active')
            ->whereBetween('regdate', [$dateFrom, $dateTo])
            ->count();
        
        return [
            'total_orders' => $totalOrders,
            'total_amount' => $totalAmount,
            'new_clients' => $newClients,
            'active_services' => $activeServices,
        ];
    }
    
    public function getChartData(string $dateFrom, string $dateTo, string $groupBy): array
    {
        $dateFormat = $this->getDateFormat($groupBy);
        
        $orders = Capsule::table('tblorders')
            ->selectRaw("
                DATE_FORMAT(date, '{$dateFormat}') as period,
                COUNT(*) as count
            ")
            ->where('status', '!=', 'Cancelled')
            ->whereBetween('date', [$dateFrom, $dateTo])
            ->groupBy('period')
            ->orderBy('period', 'asc')
            ->get();
        
        $revenue = Capsule::table('tblinvoices')
            ->selectRaw("
                DATE_FORMAT(date, '{$dateFormat}') as period,
                SUM(total) as amount
            ")
            ->where('status', 'Paid')
            ->whereBetween('date', [$dateFrom, $dateTo])
            ->groupBy('period')
            ->orderBy('period', 'asc')
            ->get();
        
        $labels = [];
        $orderCounts = [];
        $revenueAmounts = [];
        
        foreach ($orders as $order) {
            $labels[] = $order->period;
            $orderCounts[] = (int) $order->count;
        }
        
        foreach ($revenue as $rev) {
            $revenueAmounts[] = (float) $rev->amount;
        }
        
        return [
            'labels' => $labels,
            'datasets' => [
                [
                    'label' => 'Orders',
                    'data' => $orderCounts,
                    'backgroundColor' => 'rgba(54, 162, 235, 0.5)',
                    'borderColor' => 'rgba(54, 162, 235, 1)',
                    'yAxisID' => 'y-axis-1',
                ],
                [
                    'label' => 'Revenue',
                    'data' => $revenueAmounts,
                    'backgroundColor' => 'rgba(75, 192, 192, 0.5)',
                    'borderColor' => 'rgba(75, 192, 192, 1)',
                    'yAxisID' => 'y-axis-2',
                ],
            ],
        ];
    }
    
    public function exportToCSV(array $data): string
    {
        $output = "Period,Total Orders,Total Amount,Unique Customers\n";
        
        foreach ($data as $row) {
            $output .= sprintf(
                "%s,%d,%.2f,%d\n",
                $row->period,
                $row->total_orders,
                $row->total_amount,
                $row->unique_customers
            );
        }
        
        header('Content-Type: text/csv');
        header('Content-Disposition: attachment; filename="report_' . date('Y-m-d') . '.csv"');
        echo $output;
        exit;
    }
    
    private function getDateFormat(string $groupBy): string
    {
        return match($groupBy) {
            'hour' => '%Y-%m-%d %H:00',
            'day' => '%Y-%m-%d',
            'week' => '%Y-%u',
            'month' => '%Y-%m',
            'year' => '%Y',
            default => '%Y-%m-%d',
        };
    }
}
```

## Chart Generator Class

```php
<?php
namespace {Module};

class ChartGenerator {
    
    public function generateLineChart(array $data, string $canvasId): string
    {
        return "
            <canvas id=\"{$canvasId}\"></canvas>
            <script>
                new Chart(
                    document.getElementById('{$canvasId}').getContext('2d'),
                    {
                        type: 'line',
                        data: {
                            labels: " . json_encode($data['labels']) . ",
                            datasets: " . json_encode($data['datasets']) . "
                        },
                        options: {
                            responsive: true,
                            interaction: { mode: 'index', intersect: false },
                            stacked: false,
                            plugins: {
                                title: { display: true, text: 'Report Chart' }
                            },
                            scales: {
                                y: { type: 'linear', display: true, position: 'left' },
                                y1: { type: 'linear', display: true, position: 'right', grid: { drawOnChartArea: false } }
                            }
                        }
                    }
                );
            </script>
        ";
    }
    
    public function generateBarChart(array $data, string $canvasId): string
    {
        return "
            <canvas id=\"{$canvasId}\"></canvas>
            <script>
                new Chart(
                    document.getElementById('{$canvasId}').getContext('2d'),
                    {
                        type: 'bar',
                        data: {
                            labels: " . json_encode($data['labels']) . ",
                            datasets: " . json_encode($data['datasets']) . "
                        },
                        options: {
                            responsive: true,
                            plugins: {
                                legend: { position: 'top' },
                                title: { display: true, text: 'Report Chart' }
                            }
                        }
                    }
                );
            </script>
        ";
    }
    
    public function generatePieChart(array $data, string $canvasId): string
    {
        return "
            <canvas id=\"{$canvasId}\"></canvas>
            <script>
                new Chart(
                    document.getElementById('{$canvasId}').getContext('2d'),
                    {
                        type: 'pie',
                        data: {
                            labels: " . json_encode($data['labels']) . ",
                            datasets: [{
                                data: " . json_encode($data['values']) . ",
                                backgroundColor: [
                                    'rgba(255, 99, 132, 0.7)',
                                    'rgba(54, 162, 235, 0.7)',
                                    'rgba(255, 206, 86, 0.7)',
                                    'rgba(75, 192, 192, 0.7)',
                                    'rgba(153, 102, 255, 0.7)',
                                ]
                            }]
                        },
                        options: {
                            responsive: true,
                            plugins: { legend: { position: 'right' } }
                        }
                    }
                );
            </script>
        ";
    }
    
    public function generateDoughnutChart(array $data, string $canvasId): string
    {
        return "
            <canvas id=\"{$canvasId}\"></canvas>
            <script>
                new Chart(
                    document.getElementById('{$canvasId}').getContext('2d'),
                    {
                        type: 'doughnut',
                        data: {
                            labels: " . json_encode($data['labels']) . ",
                            datasets: [{
                                data: " . json_encode($data['values']) . ",
                                backgroundColor: [
                                    'rgba(255, 99, 132, 0.7)',
                                    'rgba(54, 162, 235, 0.7)',
                                    'rgba(255, 206, 86, 0.7)',
                                    'rgba(75, 192, 192, 0.7)',
                                    'rgba(153, 102, 255, 0.7)',
                                ]
                            }]
                        },
                        options: {
                            responsive: true,
                            plugins: { legend: { position: 'right' } }
                        }
                    }
                );
            </script>
        ";
    }
}
```

## Report Template

```smarty
<div class="report-container">
    <div class="report-header">
        <h2>{$report.name}</h2>
        <p class="text-muted">{$report.description}</p>
    </div>
    
    <div class="report-filters panel panel-default">
        <div class="panel-body">
            <form method="get" class="form-inline">
                <input type="hidden" name="module" value="reports">
                <input type="hidden" name="action" value="index">
                <input type="hidden" name="report" value="{$report.file}">
                
                <div class="form-group">
                    <label>From:</label>
                    <input type="date" name="date_from" class="form-control" 
                           value="{$dateFrom}">
                </div>
                
                <div class="form-group">
                    <label>To:</label>
                    <input type="date" name="date_to" class="form-control" 
                           value="{$dateTo}">
                </div>
                
                <div class="form-group">
                    <label>Group By:</label>
                    <select name="group_by" class="form-control">
                        <option value="hour" {if $groupBy == 'hour'}selected{/if}>Hour</option>
                        <option value="day" {if $groupBy == 'day'}selected{/if}>Day</option>
                        <option value="week" {if $groupBy == 'week'}selected{/if}>Week</option>
                        <option value="month" {if $groupBy == 'month'}selected{/if}>Month</option>
                        <option value="year" {if $groupBy == 'year'}selected{/if}>Year</option>
                    </select>
                </div>
                
                <button type="submit" class="btn btn-primary">
                    <i class="fa fa-filter"></i> Filter
                </button>
                <a href="?module=reports&action=index&report={$report.file}&date_from={$dateFrom}&date_to={$dateTo}&group_by={$groupBy}&export=1" 
                   class="btn btn-success">
                    <i class="fa fa-download"></i> Export CSV
                </a>
            </form>
        </div>
    </div>
    
    <div class="report-summary">
        <div class="row">
            <div class="col-md-3">
                <div class="summary-card">
                    <div class="summary-icon bg-primary">
                        <i class="fa fa-shopping-cart"></i>
                    </div>
                    <div class="summary-content">
                        <div class="summary-value">{$summary.total_orders|number_format}</div>
                        <div class="summary-label">Total Orders</div>
                    </div>
                </div>
            </div>
            <div class="col-md-3">
                <div class="summary-card">
                    <div class="summary-icon bg-success">
                        <i class="fa fa-dollar"></i>
                    </div>
                    <div class="summary-content">
                        <div class="summary-value">{$summary.total_amount|formatCurrency}</div>
                        <div class="summary-label">Total Revenue</div>
                    </div>
                </div>
            </div>
            <div class="col-md-3">
                <div class="summary-card">
                    <div class="summary-icon bg-info">
                        <i class="fa fa-users"></i>
                    </div>
                    <div class="summary-content">
                        <div class="summary-value">{$summary.new_clients|number_format}</div>
                        <div class="summary-label">New Clients</div>
                    </div>
                </div>
            </div>
            <div class="col-md-3">
                <div class="summary-card">
                    <div class="summary-icon bg-warning">
                        <i class="fa fa-server"></i>
                    </div>
                    <div class="summary-content">
                        <div class="summary-value">{$summary.active_services|number_format}</div>
                        <div class="summary-label">Active Services</div>
                    </div>
                </div>
            </div>
        </div>
    </div>
    
    <div class="report-chart panel panel-default">
        <div class="panel-heading">
            <h3 class="panel-title">Trend Analysis</h3>
        </div>
        <div class="panel-body">
            <div class="chart-wrapper">
                <canvas id="{$chartId}" height="300"></canvas>
            </div>
        </div>
    </div>
    
    <div class="report-table panel panel-default">
        <div class="panel-heading">
            <h3 class="panel-title">Detailed Data</h3>
        </div>
        <div class="panel-body">
            <table class="table table-striped table-bordered">
                <thead>
                    <tr>
                        <th>Period</th>
                        <th>Total Orders</th>
                        <th>Total Amount</th>
                        <th>Unique Customers</th>
                    </tr>
                </thead>
                <tbody>
                    {foreach $data as $row}
                    <tr>
                        <td>{$row.period}</td>
                        <td>{$row.total_orders|number_format}</td>
                        <td>{$row.total_amount|formatCurrency}</td>
                        <td>{$row.unique_customers|number_format}</td>
                    </tr>
                    {/foreach}
                </tbody>
            </table>
        </div>
    </div>
</div>

<script src="https://cdn.jsdelivr.net/npm/chart.js"></script>
<script>
    var chartData = {$chartData|json_encode};
    var ctx = document.getElementById('{$chartId}').getContext('2d');
    new Chart(ctx, chartData);
</script>
```

## Checklist

```
Pre-Dev:
□ Define report purpose and metrics
□ Identify data sources and tables
□ Plan filtering options
□ Determine visualization types
□ Design export functionality

Development:
□ Create report definition function
□ Implement ReportBuilder class
□ Add getReportData() method
□ Add getSummaryStats() method
□ Implement ChartGenerator class
□ Add chart generation methods
□ Create report Smarty template
□ Add date filtering
□ Add group by options
□ Implement CSV export

Testing:
□ Test report renders correctly
□ Verify date filtering works
□ Test group by options
□ Verify chart displays
□ Test CSV export
□ Test with different data ranges
□ Verify performance with large datasets
```