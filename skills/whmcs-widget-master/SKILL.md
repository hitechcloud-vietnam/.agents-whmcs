# WHMCS Widget Master

## Overview
Master skill for WHMCS admin dashboard widget development. Covers widget creation, data fetching, visualization, and user interaction.

## Widget Structure

```php
<?php
// /modules/widgets/YourWidget.php

if (!defined("WHMCS")) {
    die("This file cannot be accessed directly");
}

use WHMCS\Users\Models\Client;

class YourWidget extends \WHMCS\Module\Widget
{
    protected $title = 'Your Widget Title';
    protected $description = 'Description of what this widget displays';
    protected $cache = true;
    protected $cacheExpiry = 300; // 5 minutes
    protected $requiredPermission = 'View Dashboard';

    public function getData()
    {
        // Fetch and return widget data
        return [
            'stats' => $this->getStatistics(),
            'recentItems' => $this->getRecentItems(),
            'chartData' => $this->getChartData(),
        ];
    }

    protected function getStatistics(): array
    {
        $results = [];

        // Total clients
        $results['totalClients'] = \Illuminate\Database\Capsule\Manager::table('tblclients')
            ->count();

        // Active services
        $results['activeServices'] = \Illuminate\Database\Capsule\Manager::table('tblhosting')
            ->where('domainstatus', 'Active')
            ->count();

        // Pending orders
        $results['pendingOrders'] = \Illuminate\Database\Capsule\Manager::table('tblorders')
            ->where('status', 'Pending')
            ->count();

        // Open tickets
        $results['openTickets'] = \Illuminate\Database\Capsule\Manager::table('tbltickets')
            ->whereIn('status', ['Open', 'Answered'])
            ->count();

        // Monthly revenue
        $results['monthlyRevenue'] = \Illuminate\Database\Capsule\Manager::table('tblinvoices')
            ->where('status', 'Paid')
            ->whereBetween('date', [
                date('Y-m-01'),
                date('Y-m-t')
            ])
            ->sum('total');

        return $results;
    }

    protected function getRecentItems(): array
    {
        // Recent orders
        $recentOrders = \Illuminate\Database\Capsule\Manager::table('tblorders')
            ->join('tblclients', 'tblorders.userid', '=', 'tblclients.id')
            ->orderBy('tblorders.date', 'desc')
            ->limit(5)
            ->get([
                'tblorders.id',
                'tblorders.date',
                'tblorders.status',
                'tblorders.total',
                'tblclients.firstname',
                'tblclients.lastname',
            ]);

        return [
            'orders' => $recentOrders,
        ];
    }

    protected function getChartData(): array
    {
        // Orders by day (last 7 days)
        $chartData = [];
        for ($i = 6; $i >= 0; $i--) {
            $date = date('Y-m-d', strtotime("-$i days"));
            $count = \Illuminate\Database\Capsule\Manager::table('tblorders')
                ->whereDate('date', $date)
                ->count();

            $chartData['labels'][] = date('M j', strtotime($date));
            $chartData['values'][] = $count;
        }

        return $chartData;
    }

    public function generateOutput($data)
    {
        $stats = $data['stats'];
        $recentOrders = $data['recentItems']['orders'];
        $chartData = $data['chartData'];

        $html = '<div class="row">';

        // Statistics boxes
        $html .= '<div class="col-sm-6 col-md-3">
            <div class="widget-small-icon" style="background:#3498db;">
                <i class="fa fa-users"></i>
            </div>
            <div class="widget-small-info">
                <span>' . $stats['totalClients'] . '</span>
                <small>Total Clients</small>
            </div>
        </div>';

        $html .= '<div class="col-sm-6 col-md-3">
            <div class="widget-small-icon" style="background:#2ecc71;">
                <i class="fa fa-server"></i>
            </div>
            <div class="widget-small-info">
                <span>' . $stats['activeServices'] . '</span>
                <small>Active Services</small>
            </div>
        </div>';

        $html .= '<div class="col-sm-6 col-md-3">
            <div class="widget-small-icon" style="background:#f39c12;">
                <i class="fa fa-shopping-cart"></i>
            </div>
            <div class="widget-small-info">
                <span>' . $stats['pendingOrders'] . '</span>
                <small>Pending Orders</small>
            </div>
        </div>';

        $html .= '<div class="col-sm-6 col-md-3">
            <div class="widget-small-icon" style="background:#e74c3c;">
                <i class="fa fa-life-ring"></i>
            </div>
            <div class="widget-small-info">
                <span>' . $stats['openTickets'] . '</span>
                <small>Open Tickets</small>
            </div>
        </div>';

        $html .= '</div>';

        // Recent orders
        $html .= '<div class="row mt-3">
            <div class="col-md-6">
                <h5 class="mt-3 mb-2">Recent Orders</h5>
                <table class="table table-striped table-sm">
                    <thead>
                        <tr>
                            <th>ID</th>
                            <th>Client</th>
                            <th>Amount</th>
                            <th>Status</th>
                        </tr>
                    </thead>
                    <tbody>';

        foreach ($recentOrders as $order) {
            $statusClass = $this->getStatusClass($order->status);
            $html .= '<tr>
                <td>#' . $order->id . '</td>
                <td>' . htmlspecialchars($order->firstname . ' ' . $order->lastname) . '</td>
                <td>$' . number_format($order->total, 2) . '</td>
                <td><span class="label label-' . $statusClass . '">' . $order->status . '</span></td>
            </tr>';
        }

        $html .= '</tbody>
                </table>
            </div>';

        // Chart
        $html .= '<div class="col-md-6">
                <h5 class="mt-3 mb-2">Orders (Last 7 Days)</h5>
                <canvas id="yourWidgetChart" height="150"></canvas>
            </div>
        </div>';

        // JavaScript for chart
        $html .= '<script>
            if (typeof Chart !== "undefined") {
                new Chart(document.getElementById("yourWidgetChart"), {
                    type: "line",
                    data: {
                        labels: ' . json_encode($chartData['labels']) . ',
                        datasets: [{
                            label: "Orders",
                            data: ' . json_encode($chartData['values']) . ',
                            borderColor: "#3498db",
                            backgroundColor: "rgba(52, 152, 219, 0.1)",
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
                            y: {
                                beginAtZero: true,
                                ticks: { stepSize: 1 }
                            }
                        }
                    }
                });
            }
        </script>';

        return $html;
    }

    protected function getStatusClass(string $status): string
    {
        $classes = [
            'Pending' => 'warning',
            'Active' => 'success',
            'Cancelled' => 'danger',
            'Suspended' => 'danger',
            'Terminated' => 'default',
            'Fraud' => 'danger',
            'Paid' => 'success',
        ];

        return $classes[$status] ?? 'default';
    }

    public function getPermissions(): array
    {
        return ['View Dashboard'];
    }
}
```

## Advanced Widget with AJAX

```php
<?php
// /modules/widgets/AdvancedWidget.php

if (!defined("WHMCS")) {
    die("This file cannot be accessed directly");
}

class AdvancedWidget extends \WHMCS\Module\Widget
{
    protected $title = 'Advanced Widget';
    protected $description = 'Widget with AJAX functionality';
    protected $cache = false; // Disable cache for real-time data

    public function getData()
    {
        return [
            'summary' => $this->getSummary(),
        ];
    }

    public function generateOutput($data)
    {
        $html = '<div class="advanced-widget" id="advancedWidget_' . $this->getUniqueId() . '">';

        // Header with refresh button
        $html .= '<div class="widget-header">
            <span>' . $this->getTitle() . '</span>
            <button class="btn btn-xs btn-default" onclick="refreshAdvancedWidget()">
                <i class="fa fa-refresh"></i>
            </button>
        </div>';

        // Content
        $html .= '<div class="widget-content" id="advancedWidgetContent_' . $this->getUniqueId() . '">';
        $html .= $this->renderSummary($data['summary']);
        $html .= '</div>';

        // AJAX Script
        $html .= '<script>
            function refreshAdvancedWidget() {
                var container = $("#advancedWidgetContent_' . $this->getUniqueId() . '");
                container.html("<div class=\"text-center\"><i class=\"fa fa-spinner fa-spin\"></i></div>");

                $.post("' . \App::getSystemURL() . 'modules/widgets/ajax/AdvancedWidget.php", {
                    action: "refresh",
                    token: "' . generate_token() . '"
                }, function(response) {
                    container.html(response.html);
                    if (response.chartData) {
                        renderChart(response.chartData);
                    }
                }, "json");
            }

            $(document).ready(function() {
                // Initial load
                refreshAdvancedWidget();

                // Auto-refresh every 60 seconds
                setInterval(refreshAdvancedWidget, 60000);
            });
        </script>';

        $html .= '</div>';

        return $html;
    }

    protected function renderSummary(array $summary): string
    {
        $html = '<div class="row">';

        foreach ($summary as $item) {
            $html .= '<div class="col-xs-6 col-md-3 text-center">
                <div class="stat-item">
                    <div class="stat-value">' . $item['value'] . '</div>
                    <div class="stat-label">' . $item['label'] . '</div>
                </div>
            </div>';
        }

        $html .= '</div>';

        return $html;
    }

    public function getPermissions(): array
    {
        return ['View Dashboard', 'View Orders'];
    }
}
```

## AJAX Handler

```php
<?php
// /modules/widgets/ajax/AdvancedWidget.php

if (!defined("WHMCS")) {
    die("This file cannot be accessed directly");
}

require __DIR__ . '/../../init.php';

checkAjaxRequest();

$action = $_POST['action'] ?? $_GET['action'] ?? '';

switch ($action) {
    case 'refresh':
        echo json_encode([
            'html' => getRefreshedContent(),
            'chartData' => getChartData(),
        ]);
        break;

    case 'getDetails':
        $id = (int)($_POST['id'] ?? 0);
        echo json_encode([
            'html' => getDetailsHtml($id),
        ]);
        break;

    case 'export':
        exportData();
        break;

    default:
        echo json_encode(['error' => 'Unknown action']);
}

function getRefreshedContent(): string
{
    $stats = getStats();

    $html = '<div class="row">';
    foreach ($stats as $stat) {
        $html .= '<div class="col-xs-6 col-md-3 text-center">
            <div class="stat-item">
                <div class="stat-value">' . $stat['value'] . '</div>
                <div class="stat-label">' . $stat['label'] . '</div>
            </div>
        </div>';
    }
    $html .= '</div>';

    return $html;
}

function getChartData(): array
{
    $labels = [];
    $values = [];

    for ($i = 6; $i >= 0; $i--) {
        $date = date('Y-m-d', strtotime("-$i days"));
        $labels[] = date('M j', strtotime($date));
        $values[] = rand(0, 100); // Replace with actual data
    }

    return [
        'labels' => $labels,
        'values' => $values,
    ];
}

function getStats(): array
{
    return [
        [
            'label' => 'New Today',
            'value' => \Illuminate\Database\Capsule\Manager::table('tblorders')
                ->whereDate('date', date('Y-m-d'))
                ->count(),
        ],
        [
            'label' => 'Revenue Today',
            'value' => '$' . number_format(
                \Illuminate\Database\Capsule\Manager::table('tblinvoices')
                    ->where('status', 'Paid')
                    ->whereDate('date', date('Y-m-d'))
                    ->sum('total'),
                2
            ),
        ],
        [
            'label' => 'Open Tickets',
            'value' => \Illuminate\Database\Capsule\Manager::table('tbltickets')
                ->where('status', 'Open')
                ->count(),
        ],
        [
            'label' => 'Pending',
            'value' => \Illuminate\Database\Capsule\Manager::table('tblorders')
                ->where('status', 'Pending')
                ->count(),
        ],
    ];
}
```

## Widget Registration

```php
<?php
// /modules/widgets/Widgets.php

if (!defined("WHMCS")) {
    die("This file cannot be accessed directly");
}

return [
    'YourWidget',
    'AdvancedWidget',
    // Other default widgets:
    // 'AnnouncementsWidget',
    // 'ActivityLogWidget',
    // 'OrdersWidget',
    // 'IncomeWidget',
    // 'SupportWidget',
    // 'TicketUpdatersWidget',
    // 'ClientWidget',
];
```

## Best Practices

1. **Caching**: Use appropriate cache expiry times for data-heavy widgets
2. **Permissions**: Implement proper permission checks
3. **Error Handling**: Handle database errors gracefully
4. **Responsive Design**: Make widgets work on different screen sizes
5. **AJAX Updates**: Use AJAX for real-time data without page refresh
6. **Chart Libraries**: Use Chart.js or similar for visualizations
7. **Unique IDs**: Ensure widget instances have unique IDs
8. **Performance**: Optimize queries and avoid N+1 problems
9. **Security**: Validate all inputs in AJAX handlers
10. **Documentation**: Document widget configuration options
