# WHMCS Widget Module DevKit
# Version: 1.0 | Updated: 2026-05-28

## DevKit Structure

```
devkits/whmcs-widget-module/
├── widget.php           # Widget definition
├── lib/
│   └── WidgetData.php    # Data provider class
├── templates/
│   └── widget.tpl        # Widget template
└── widget-admin.php      # Admin configuration
```

## Widget Definition Template

```php
<?php
/**
 * WHMCS Admin Widget: {Module}
 * DevKit Template
 * 
 * Installation: Copy to /includes/widget/{module}_widget.php
 */

if (!defined("WHMCS")) {
    die("Direct access denied");
}

/**
 * Widget Class
 */
class {Module}Widget extends \WHMCS\Module\AbstractWidget
{
    protected $title = '{Widget Title}';
    protected $description = '{Widget description}';
    protected $width = 2; // 1, 2, or 4 columns
    
    public function getData(): array
    {
        return [
            'total_clients' => $this->getTotalClients(),
            'total_services' => $this->getTotalServices(),
            'total_income' => $this->getTotalIncome(),
            'recent_orders' => $this->getRecentOrders(),
            'chart_data' => $this->getChartData(),
        ];
    }
    
    private function getTotalClients(): int
    {
        return \WHMCS\Database\Capsule::table('tblclients')->count();
    }
    
    private function getTotalServices(): int
    {
        return \WHMCS\Database\Capsule::table('tblhosting')->count();
    }
    
    private function getTotalIncome(): float
    {
        return \WHMCS\Database\Capsule::table('tblinvoices')
            ->where('status', 'Paid')
            ->sum('total');
    }
    
    private function getRecentOrders(): array
    {
        return \WHMCS\Database\Capsule::table('tblorders')
            ->orderBy('date', 'desc')
            ->limit(5)
            ->get()
            ->toArray();
    }
    
    private function getChartData(): array
    {
        $data = [];
        for ($i = 6; $i >= 0; $i--) {
            $date = date('Y-m-d', strtotime("-{$i} days"));
            $count = \WHMCS\Database\Capsule::table('tblorders')
                ->whereDate('date', $date)
                ->count();
            $data[] = [
                'date' => $date,
                'count' => $count,
            ];
        }
        return $data;
    }
}

/**
 * Register Widget
 */
add_hook('AdminHomepage', 1, function($vars) {
    return new {Module}Widget();
});
```

## Alternative Widget Definition (Function-based)

```php
<?php
/**
 * WHMCS Admin Widget: {Module}
 * Function-based Widget Template
 */

if (!defined("WHMCS")) {
    die("Direct access denied");
}

/**
 * Widget Output Function
 * Called by WHMCS to render the widget
 */
function {module}_widget(array $vars): string
{
    $data = {module}_getWidgetData();
    
    $chartId = 'chart_' . md5($vars['widget']['guid']);
    
    $output = '<div class="portlet">
        <div class="portlet-title">
            <div class="caption">
                <i class="fa fa-{icon}"></i>
                ' . $vars['widget']['name'] . '
            </div>
            <div class="actions">
                <a href="{module}/settings" class="btn btn-default btn-sm">
                    <i class="fa fa-cog"></i>
                </a>
            </div>
        </div>
        <div class="portlet-body">
            <div class="row">
                <div class="col-xs-6">
                    <div class="dashboard-stat blue-madison">
                        <div class="visual">
                            <i class="fa fa-users"></i>
                        </div>
                        <div class="details">
                            <div class="number">' . number_format($data['total_clients']) . '</div>
                            <div class="desc">Total Clients</div>
                        </div>
                    </div>
                </div>
                <div class="col-xs-6">
                    <div class="dashboard-stat green-haze">
                        <div class="visual">
                            <i class="fa fa-shopping-cart"></i>
                        </div>
                        <div class="details">
                            <div class="number">' . number_format($data['total_services']) . '</div>
                            <div class="desc">Active Services</div>
                        </div>
                    </div>
                </div>
            </div>
            
            <canvas id="' . $chartId . '" height="150"></canvas>
            
            <script>
                (function() {
                    var ctx = document.getElementById("' . $chartId . '");
                    new Chart(ctx, {
                        type: "line",
                        data: {
                            labels: ' . json_encode(array_column($data['chart_data'], 'date')) . ',
                            datasets: [{
                                label: "Orders",
                                data: ' . json_encode(array_column($data['chart_data'], 'count')) . ',
                                borderColor: "#3598dc",
                                fill: false
                            }]
                        },
                        options: {
                            responsive: true,
                            maintainAspectRatio: false
                        }
                    });
                })();
            </script>
        </div>
    </div>';
    
    return $output;
}

/**
 * Get Widget Data
 */
function {module}_getWidgetData(): array
{
    use WHMCS\Database\Capsule;
    
    $totalClients = Capsule::table('tblclients')->count();
    $totalServices = Capsule::table('tblhosting')
        ->where('domainstatus', 'Active')
        ->count();
    
    $chartData = [];
    for ($i = 6; $i >= 0; $i--) {
        $date = date('Y-m-d', strtotime("-{$i} days"));
        $count = Capsule::table('tblorders')
            ->whereDate('date', $date)
            ->count();
        $chartData[] = ['date' => $date, 'count' => $count];
    }
    
    return [
        'total_clients' => $totalClients,
        'total_services' => $totalServices,
        'chart_data' => $chartData,
    ];
}

/**
 * Register Widget Hook
 */
add_hook('AdminHomepage', 1, function($vars) {
    return [
        'name' => '{Widget Title}',
        'description' => '{Widget description}',
        'filename' => basename(__FILE__, '.php'),
        'function' => '{module}_widget',
    ];
});
```

## Widget Data Provider Class

```php
<?php
namespace {Module};

use WHMCS\Database\Capsule;

class WidgetData {
    
    public function getOverviewStats(): array
    {
        return [
            'total_clients' => $this->getTotalClients(),
            'new_clients_today' => $this->getNewClientsToday(),
            'new_clients_this_week' => $this->getNewClientsThisWeek(),
            'new_clients_this_month' => $this->getNewClientsThisMonth(),
            'active_services' => $this->getActiveServices(),
            'pending_orders' => $this->getPendingOrders(),
            'open_tickets' => $this->getOpenTickets(),
            'monthly_income' => $this->getMonthlyIncome(),
        ];
    }
    
    public function getTotalClients(): int
    {
        return Capsule::table('tblclients')->count();
    }
    
    public function getNewClientsToday(): int
    {
        return Capsule::table('tblclients')
            ->whereDate('created_at', date('Y-m-d'))
            ->count();
    }
    
    public function getNewClientsThisWeek(): int
    {
        return Capsule::table('tblclients')
            ->whereBetween('created_at', [
                date('Y-m-d', strtotime('monday this week')),
                date('Y-m-d', strtotime('sunday this week'))
            ])
            ->count();
    }
    
    public function getNewClientsThisMonth(): int
    {
        return Capsule::table('tblclients')
            ->whereYear('created_at', date('Y'))
            ->whereMonth('created_at', date('m'))
            ->count();
    }
    
    public function getActiveServices(): int
    {
        return Capsule::table('tblhosting')
            ->where('domainstatus', 'Active')
            ->count();
    }
    
    public function getPendingOrders(): int
    {
        return Capsule::table('tblorders')
            ->whereIn('status', ['Pending', 'Active'])
            ->count();
    }
    
    public function getOpenTickets(): int
    {
        return Capsule::table('tbltickets')
            ->whereIn('status', ['Open', 'Awaiting Reply'])
            ->count();
    }
    
    public function getMonthlyIncome(): float
    {
        return Capsule::table('tblinvoices')
            ->where('status', 'Paid')
            ->whereYear('datepaid', date('Y'))
            ->whereMonth('datepaid', date('m'))
            ->sum('total');
    }
    
    public function getChartData(string $period = 'week'): array
    {
        $labels = [];
        $values = [];
        
        switch ($period) {
            case 'week':
                for ($i = 6; $i >= 0; $i--) {
                    $date = date('Y-m-d', strtotime("-{$i} days"));
                    $count = Capsule::table('tblorders')
                        ->whereDate('date', $date)
                        ->count();
                    $labels[] = date('D', strtotime($date));
                    $values[] = $count;
                }
                break;
                
            case 'month':
                for ($i = 29; $i >= 0; $i--) {
                    $date = date('Y-m-d', strtotime("-{$i} days"));
                    $count = Capsule::table('tblorders')
                        ->whereDate('date', $date)
                        ->count();
                    $labels[] = date('d', strtotime($date));
                    $values[] = $count;
                }
                break;
                
            case 'year':
                for ($i = 11; $i >= 0; $i--) {
                    $month = date('Y-m', strtotime("-{$i} months"));
                    $count = Capsule::table('tblorders')
                        ->whereRaw("DATE_FORMAT(date, '%Y-%m') = ?", [$month])
                        ->count();
                    $labels[] = date('M', strtotime($month . '-01'));
                    $values[] = $count;
                }
                break;
        }
        
        return ['labels' => $labels, 'values' => $values];
    }
    
    public function getRecentActivity(int $limit = 10): array
    {
        return Capsule::table('tblactivitylog')
            ->orderBy('id', 'desc')
            ->limit($limit)
            ->get()
            ->toArray();
    }
    
    public function getTopProducts(int $limit = 5): array
    {
        return Capsule::table('tblhosting')
            ->selectRaw('tblproducts.name, COUNT(*) as count')
            ->join('tblproducts', 'tblhosting.packageid', '=', 'tblproducts.id')
            ->where('tblhosting.domainstatus', 'Active')
            ->groupBy('tblproducts.name')
            ->orderBy('count', 'desc')
            ->limit($limit)
            ->get()
            ->toArray();
    }
}
```

## Widget Template

```smarty
<div class="widget" id="widget-{module}">
    <div class="widget-header">
        <div class="widget-title">
            <i class="fa fa-{icon}"></i>
            <span>{$title}</span>
        </div>
        <div class="widget-actions">
            <a href="{$base_url}settings" class="btn btn-xs btn-default">
                <i class="fa fa-cog"></i>
            </a>
        </div>
    </div>
    
    <div class="widget-body">
        <div class="stats-grid">
            <div class="stat-item">
                <div class="stat-value">{$data.total_clients|number_format}</div>
                <div class="stat-label">Total Clients</div>
            </div>
            <div class="stat-item">
                <div class="stat-value">{$data.total_services|number_format}</div>
                <div class="stat-label">Active Services</div>
            </div>
            <div class="stat-item">
                <div class="stat-value">{$data.total_income|formatCurrency}</div>
                <div class="stat-label">Total Income</div>
            </div>
        </div>
        
        <div class="chart-container">
            <canvas id="chart-{module}" height="120"></canvas>
        </div>
        
        <div class="recent-list">
            <h4>Recent Orders</h4>
            <ul class="list-unstyled">
                {foreach $data.recent_orders as $order}
                <li class="order-item">
                    <span class="order-id">#{$order.id}</span>
                    <span class="order-client">{$order.name}</span>
                    <span class="order-date">{$order.date}</span>
                </li>
                {/foreach}
            </ul>
        </div>
    </div>
    
    <div class="widget-footer">
        <a href="{$base_url}">View All</a>
    </div>
</div>

<script>
    // Initialize Chart
    var ctx = document.getElementById('chart-{module}').getContext('2d');
    new Chart(ctx, {
        type: 'line',
        data: {
            labels: {json_encode($data.chart_labels)},
            datasets: [{
                label: 'Orders',
                data: {json_encode($data.chart_values)},
                borderColor: '#3598dc',
                backgroundColor: 'rgba(53, 152, 220, 0.1)',
                fill: true,
                tension: 0.4
            }]
        },
        options: {
            responsive: true,
            maintainAspectRatio: false,
            plugins: {
                legend: { display: false }
            },
            scales: {
                y: { beginAtZero: true }
            }
        }
    });
</script>
```

## Admin Configuration Page

```php
<?php
/**
 * Widget Admin Configuration
 */

if (!defined("WHMCS")) {
    die("Direct access denied");
}

function {module}_widget_config(): array
{
    return [
        'name' => ['Type' => 'text', 'Default' => '{Widget Title}'],
        'icon' => ['Type' => 'text', 'Default' => 'fa-chart-bar'],
        'refresh_interval' => ['Type' => 'text', 'Default' => '300'],
        'display_stats' => ['Type' => 'textarea', 'Default' => 'total_clients,total_services'],
    ];
}

function {module}_widget_config_save(array $params): void
{
    $settings = [
        'name' => $params['name'],
        'icon' => $params['icon'],
        'refresh_interval' => (int) $params['refresh_interval'],
        'display_stats' => $params['display_stats'],
    ];
    
    Capsule::table('tbladdon_modules')
        ->where('module', '{module}_widget')
        ->update(['value' => json_encode($settings)]);
}
```

## Checklist

```
Pre-Dev:
□ Define widget purpose and data to display
□ Determine widget size (1, 2, or 4 columns)
□ Plan chart/visualization requirements
□ Identify data sources
□ Design widget layout

Development:
□ Create widget class or function
□ Implement getData() method
□ Add chart visualization (Chart.js)
□ Create WidgetData provider class
□ Add statistics calculations
□ Create widget Smarty template
□ Add refresh functionality
□ Implement admin configuration

Testing:
□ Test widget displays on admin homepage
□ Verify data is accurate
□ Test chart rendering
□ Test refresh functionality
□ Test widget resizing
□ Verify permissions
```