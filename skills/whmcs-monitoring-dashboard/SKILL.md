# WHMCS Monitoring Dashboard Skill
# Version: 1.0 | Updated: 2026-05-29

## Purpose

Guide for building monitoring dashboards in WHMCS to track system health, service status, and business metrics.

## When to Use

- Building admin monitoring dashboards
- Displaying server health metrics
- Tracking service uptime
- Real-time business intelligence

## Monitoring Dashboard Patterns

### 1. Dashboard Data Collector

```php
<?php
namespace WHMCS\Monitoring;

class DashboardCollector {
    private int $cacheTtl = 60; // 1 minute cache

    public function getSystemOverview(): array {
        $cache = new DashboardCache();
        $key = 'system_overview';

        if ($data = $cache->get($key)) {
            return $data;
        }

        $data = [
            'timestamp' => date('Y-m-d H:i:s'),
            'clients' => $this->getClientStats(),
            'services' => $this->getServiceStats(),
            'invoices' => $this->getInvoiceStats(),
            'tickets' => $this->getTicketStats(),
            'servers' => $this->getServerStats(),
            'revenue' => $this->getRevenueStats(),
        ];

        $cache->set($key, $data, $this->cacheTtl);

        return $data;
    }

    private function getClientStats(): array {
        $active = Capsule::table('tblclients')
            ->where('status', 'Active')
            ->count();

        $inactive = Capsule::table('tblclients')
            ->where('status', '!=', 'Active')
            ->count();

        $newThisMonth = Capsule::table('tblclients')
            ->where('datecreated', '>=', date('Y-m-01'))
            ->count();

        return [
            'active' => $active,
            'inactive' => $inactive,
            'new_this_month' => $newThisMonth,
            'total' => $active + $inactive,
        ];
    }

    private function getServiceStats(): array {
        $active = Capsule::table('tblhosting')
            ->where('domainstatus', 'Active')
            ->count();

        $suspended = Capsule::table('tblhosting')
            ->where('domainstatus', 'Suspended')
            ->count();

        $pending = Capsule::table('tblhosting')
            ->where('domainstatus', 'Pending')
            ->count();

        $expiringIn7Days = Capsule::table('tblhosting')
            ->where('domainstatus', 'Active')
            ->where('nextduedate', '<=', date('Y-m-d', strtotime('+7 days')))
            ->count();

        return [
            'active' => $active,
            'suspended' => $suspended,
            'pending' => $pending,
            'expiring_soon' => $expiringIn7Days,
            'total' => $active + $suspended + $pending,
        ];
    }

    private function getInvoiceStats(): array {
        $paid = Capsule::table('tblinvoices')
            ->selectRaw('SUM(total) as amount, COUNT(*) as count')
            ->where('status', 'Paid')
            ->where('invoicedate', '>=', date('Y-m-01'))
            ->first();

        $pending = Capsule::table('tblinvoices')
            ->selectRaw('SUM(total) as amount, COUNT(*) as count')
            ->where('status', 'Pending')
            ->first();

        $overdue = Capsule::table('tblinvoices')
            ->selectRaw('SUM(total) as amount, COUNT(*) as count')
            ->where('status', 'Overdue')
            ->count();

        $cancelled = Capsule::table('tblinvoices')
            ->where('status', 'Cancelled')
            ->where('invoicedate', '>=', date('Y-m-01'))
            ->count();

        return [
            'paid' => $paid->count ?? 0,
            'paid_amount' => $paid->amount ?? 0,
            'pending_count' => $pending->count ?? 0,
            'pending_amount' => $pending->amount ?? 0,
            'overdue_count' => $overdue,
            'cancelled_count' => $cancelled,
        ];
    }

    private function getTicketStats(): array {
        $open = Capsule::table('tbltickets')
            ->where('status', 'Open')
            ->count();

        $inProgress = Capsule::table('tbltickets')
            ->whereIn('status', ['Open', 'In Progress', 'Awaiting Reply'])
            ->count();

        $answered = Capsule::table('tbltickets')
            ->where('status', 'Answered')
            ->count();

        $newToday = Capsule::table('tbltickets')
            ->where('created_at', '>=', date('Y-m-d'))
            ->count();

        return [
            'open' => $open,
            'in_progress' => $inProgress,
            'answered' => $answered,
            'new_today' => $newToday,
        ];
    }

    private function getServerStats(): array {
        $total = Capsule::table('tblservers')
            ->count();

        $active = Capsule::table('tblservers')
            ->where('disabled', 0)
            ->count();

        $disabled = Capsule::table('tblservers')
            ->where('disabled', 1)
            ->count();

        return [
            'total' => $total,
            'active' => $active,
            'disabled' => $disabled,
        ];
    }

    private function getRevenueStats(): array {
        $monthly = Capsule::table('tblinvoices')
            ->selectRaw('SUM(total) as amount')
            ->where('status', 'Paid')
            ->where('invoicedate', '>=', date('Y-m-01'))
            ->first();

        $yearly = Capsule::table('tblinvoices')
            ->selectRaw('SUM(total) as amount')
            ->where('status', 'Paid')
            ->where('invoicedate', '>=', date('Y-01-01'))
            ->first();

        $lastMonth = Capsule::table('tblinvoices')
            ->selectRaw('SUM(total) as amount')
            ->where('status', 'Paid')
            ->where('invoicedate', '>=', date('Y-m-01', strtotime('-1 month')))
            ->where('invoicedate', '<', date('Y-m-01'))
            ->first();

        $growth = $lastMonth->amount > 0
            ? (($monthly->amount - $lastMonth->amount) / $lastMonth->amount) * 100
            : 0;

        return [
            'monthly' => $monthly->amount ?? 0,
            'yearly' => $yearly->amount ?? 0,
            'growth_percent' => round($growth, 1),
        ];
    }
}
```

### 2. Charts Data Provider

```php
<?php
<?php
namespace WHMCS\Monitoring;

class ChartDataProvider {
    public function getRevenueChart(int $months = 12): array {
        $data = [];
        $labels = [];

        for ($i = $months - ; $i >= 0; $i--) {
            $month = date('Y-m', strtotime("-$i months"));
            $startDate = date('Y-m-01', strtotime("-$i months"));
            $endDate = date('Y-m-t', strtotime("-$i months"));

            $revenue = Capsule::table('tblinvoices')
                ->selectRaw('SUM(total) as amount')
                ->where('status', 'Paid')
                ->whereBetween('invoicedate', [$startDate, $endDate])
                ->first();

            $data[] = (float) ($revenue->amount ?? 0);
            $labels[] = date('M Y', strtotime("-$i months"));
        }

        return [
            'labels' => $labels,
            'datasets' => [
                [
                    'label' => 'Monthly Revenue',
                    'data' => $data,
                    'fill' => true,
                    'borderColor' => '#3498db',
                    'backgroundColor' => 'rgba(52, 152, 219, 0.1)',
                ],
            ],
        ];
    }

    public function getServiceStatusChart(): array {
        $statuses = Capsule::table('tblhosting')
            ->selectRaw('domainstatus, COUNT(*) as count')
            ->groupBy('domainstatus')
            ->get()
            ->pluck('count', 'domainstatus')
            ->toArray();

        $colors = [
            'Active' => '#27ae60',
            'Suspended' => '#f39c12',
            'Terminated' => '#e74c3c',
            'Pending' => '#95a5a6',
            'Cancelled' => '#7f8c8d',
        ];

        $labels = [];
        $data = [];
        $backgroundColor = [];

        foreach ($statuses as $status => $count) {
            $labels[] = $status;
            $data[] = $count;
            $backgroundColor[] = $colors[$status] ?? '#95a5a6';
        }

        return [
            'labels' => $labels,
            'datasets' => [
                [
                    'data' => $data,
                    'backgroundColor' => $backgroundColor,
                ],
            ],
        ];
    }

    public function getNewClientsChart(int $days = 30): array {
        $data = [];
        $labels = [];

        for ($i = $days - 1; $i >= 0; $i--) {
            $date = date('Y-m-d', strtotime("-$i days"));

            $count = Capsule::table('tblclients')
                ->where('datecreated', '>=', $date . ' 00:00:00')
                ->where('datecreated', '<=', $date . ' 23:59:59')
                ->count();

            $data[] = $count;
            $labels[] = date('M j', strtotime("-$i days"));
        }

        return [
            'labels' => $labels,
            'datasets' => [
                [
                    'label' => 'New Clients',
                    'data' => $data,
                    'borderColor' => '#9b59b6',
                    'backgroundColor' => 'rgba(155, 89, 182, 0.1)',
                    'fill' => true,
                ],
            ],
        ];
    }

    public function getTopProductsChart(int $limit = 10): array {
        $products = Capsule::table('tblhosting')
            ->selectRaw('tblproducts.name as product, COUNT(*) as count')
            ->join('tblproducts', 'tblhosting.packageid', '=', 'tblproducts.id')
            ->where('tblhosting.domainstatus', 'Active')
            ->groupBy('tblproducts.name')
            ->orderBy('count', 'desc')
            ->limit($limit)
            ->get()
            ->toArray();

        $labels = array_map(fn($p) => $p->product, $products);
        $data = array_map(fn($p) => $p->count, $products);

        return [
            'labels' => $labels,
            'datasets' => [
                [
                    'label' => 'Active Services',
                    'data' => $data,
                    'backgroundColor' => $this->generateColors(count($products)),
                ],
            ],
        ];
    }

    private function generateColors(int $count): array {
        $colors = [
            '#3498db', '#2ecc71', '#e74c3c', '#f1c40f', '#9b59b6',
            '#1abc9c', '#34495e', '#e67e22', '#2980b9', '#27ae60',
        ];

        return array_slice($colors, 0, $count);
    }
}
```

### 3. Real-Time Server Monitoring

```php
<?php
namespace WHMCS\Monitoring;

class ServerMonitor {
    private int $timeout = 5;

    public function getServerHealth(int $serverId): array {
        $server = Capsule::table('tblservers')->find($serverId);

        $health = [
            'server_id' => $serverId,
            'name' => $server->name,
            'hostname' => $server->hostname,
            'ip' => $server->ipaddress,
            'status' => $server->disabled ? 'disabled' : 'active',
            'online' => false,
            'load' => null,
            'memory' => null,
            'disk' => null,
            'services' => [],
            'checked_at' => date('Y-m-d H:i:s'),
        ];

        if (!$server->disabled) {
            try {
                $health['online'] = $this->pingServer($server->ipaddress);

                if ($health['online']) {
                    $metrics = $this->getServerMetrics($server);
                    $health = array_merge($health, $metrics);
                    $health['services'] = $this->checkServices($server);
                }
            } catch (\Exception $e) {
                $health['error'] = $e->getMessage();
            }
        }

        return $health;
    }

    private function pingServer(string $ip): bool {
        $ch = curl_init();
        curl_setopt_array($ch, [
            CURLOPT_URL => "http://{$ip}:{$GLOBALS['CONFIG']['ServerPort'] ?? 80}/status",
            CURLOPT_RETURNTRANSFER => true,
            CURLOPT_TIMEOUT => $this->timeout,
            CURLOPT_CONNECTTIMEOUT => $this->timeout,
        ]);

        $response = curl_exec($ch);
        curl_close($ch);

        return $response !== false;
    }

    private function getServerMetrics(object $server): array {
        // Simulated metrics - replace with actual API call to server monitoring agent
        return [
            'load' => [
                '1min' => rand(0, 100) / 100,
                '5min' => rand(0, 100) / 100,
                '15min' => rand(0, 100) / 100,
            ],
            'memory' => [
                'used' => rand(1000, 8000),
                'total' => 16384,
                'percent' => rand(10, 90),
            ],
            'disk' => ethis->disk = [
                'used' => rand(100, 500),
                'total' => 1000,
                'percent' => rand(10, 90),
            ],
        ];
    }

    private function checkServices(object $server): array {
        // Check critical services
        $services = ['http', 'mysql', 'ssh'];

        $status = [];
        foreach ($services as $service) {
            $status[$service] = [
                'name' => $service,
                'running' => rand(0, 1) === 1,
                'status' => 'ok',
            ];
        }

        return $status;
    }

    public function getAllServersHealth(): array {
        $servers = Capsule::table('tblservers')->get();

        $health = [];
        foreach ($servers as $server) {
            $health[] = $this->getServerHealth($server->id);
        }

        return $health;
    }

    public function getServerUptime(int $serverId, int $days = 30): array {
        $metrics = Capsule::table('mod_server_metrics')
            ->where('server_id', $serverId)
            ->where('date', '>=', date('Y-m-d', strtotime("-$days days")))
            ->orderBy('date')
            ->get();

        $uptimeData = [];
        $totalMinutes = 0;
        $availableMinutes = 0;

        foreach ($metrics as $metric) {
            $uptimeData[] = [
                'date' => $metric->date,
                'uptime_percent' => $metric->uptime_percent,
                'downtime_minutes' => $metric->downtime_minutes,
            ];

            $totalMinutes += 1440; // Minutes per day
            $availableMinutes += 1440 - $metric->downtime_minutes;
        }

        $overallUptime = $totalMinutes > 0
            ? ($availableMinutes / $totalMinutes) * 100
            : 0;

        return [
            'server_id' => $serverId,
            'period_days' => $days,
            'overall_uptime_percent' => round($overallUptime, 2),
            'daily_data' => $uptimeData,
        ];
    }
}
```

### 4. Dashboard UI Builder

```php
<?php
// modules/addons/{module}/admin/dashboard.php
if (!defined("WHMCS")) { die("Direct access denied"); }

use WHMCS\Monitoring\DashboardCollector;
use WHMCS\Monitoring\ChartDataProvider;

$collector = new DashboardCollector();
$overview = $collector->getSystemOverview();

$charts = new ChartDataProvider();
$revenueChart = $charts->getRevenueChart();
$serviceChart = $charts->getServiceStatusChart();
$newClientsChart = $charts->getNewClientsChart();
$topProductsChart = $charts->getTopProductsChart();

// Render dashboard
echo <<<HTML
<div class="monitoring-dashboard">
    <div class="dashboard-header">
        <h1>System Monitoring Dashboard</h1>
        <p class="text-muted">Last updated: {$overview['timestamp']}</p>
    </div>

    <div class="stats-grid">
        <div class="stat-card">
            <h3>Active Clients</h3>
            <p class="stat-value">{$overview['clients']['active']}</p>
            <p class="stat-change">+{$overview['clients']['new_this_month']} this month</p>
        </div>

        <div class="stat-card">
            <h3>Active Services</h3>
            <p class="stat-value">{$overview['services']['active']}</p>
            <p class="stat-warning">{$overview['services']['expiring_soon']} expiring soon</p>
        </div>

        <div class="stat-card">
            <h3>Monthly Revenue</h3>
            <p class="stat-value">\$number_format({$overview['revenue']['monthly']}, 2)</p>
            <p class="stat-change" style="color: {($overview['revenue']['growth_percent'] >= 0 ? 'green' : 'red')}">
                {$overview['revenue']['growth_percent']}% vs last month
            </p>
        </div>

        <div class="stat-card">
            <h3>Open Tickets</h3>
            <p class="stat-value">{$overview['tickets']['open']}</p>
            <p class="stat-info">{$overview['tickets']['new_today']} new today</p>
        </div>
    </div>

    <div class="charts-grid">
        <div class="chart-container">
            <h3>Revenue Trend</h3>
            <canvas id="revenueChart" data-chart='{json_encode($revenueChart)}'></canvas>
        </div>

        <div class="chart-container">
            <h3>Service Status</h3>
            <canvas id="serviceChart" data-chart='{json_encode($serviceChart)}'></canvas>
        </div>
    </div>

    <div class="servers-section">
        <h3>Server Health</h3>
        <div class="server-list">
            <!-- Server health cards populated by JS -->
        </div>
    </div>
</div>

<script>
// Initialize charts with Chart.js
document.addEventListener('DOMContentLoaded', function() {
    initChart('revenueChart', 'line', JSON.parse(document.getElementById('revenueChart').dataset.chart));
    initChart('serviceChart', 'doughnut', JSON.parse(document.getElementById('serviceChart').dataset.chart));
    loadServerHealth();
});

async function loadServerHealth() {
    const response = await fetch('ajax.php?action=getServerHealth');
    const servers = await response.json();

    // Render server cards
}
</script>
HTML;
```

### 5. Real-Time Updates

```php
<?php
namespace WHMCS\Monitoring;

class RealtimeUpdater {
    private string $redisHost;
    private int $redisPort = 6379;

    public function __construct() {
        $config = Capsule::table('tblconfiguration')
            ->whereIn('setting', ['RedisHost', 'RedisPort'])
            ->pluck('value', 'setting')
            ->toArray();

        $this->redisHost = $config['RedisHost'] ?? '127.0.0.1';
        $this->redisPort = (int) ($config['RedisPort'] ?? 6379);
    }

    public function publishUpdate(string $channel, array $data): void {
        // Using Redis pub/sub for real-time updates
        $redis = new \Redis();
        $redis->connect($this->redisHost, $this->redisPort);

        $redis->publish($channel, json_encode([
            'timestamp' => time(),
            'data' => $data,
        ]));
    }

    public function subscribeToUpdates(string $channel, callable $callback): void {
        // This would be used in a WebSocket or SSE endpoint
        $redis = new \Redis();
        $redis->connect($this->redisHost, $this->redisPort);
        $redis->subscribe([$channel], function($redis, $channel, $message) use ($callback) {
            $data = json_decode($message, true);
            $callback($data['data']);
        });
    }

    public function setDashboardMetric(string $key, $value, int $ttl = 60): void {
        $redis = new \Redis();
        $redis->connect($this->redisHost, $this->redisPort);

        $redis->setex("dashboard:{$key}", $ttl, json_encode([
            'value' => $value,
            'updated' => time(),
        ]));
    }

    public function getDashboardMetric(string $key): ?array {
        $redis = new \Redis();
        $redis->connect($this->redisHost, $this->redisPort);

        $data = $redis->get("dashboard:{$key}");

        return $data ? json_decode($data, true) : null;
    }
}
```

### 6. Database Schema

```php
<?php
function createMonitoringTables(): void {
    Capsule::schema()->create('mod_server_metrics', function($t) {
        $t->increments('id');
        $t->integer('server_id')->unsigned();
        $t->date('date');
        $t->decimal('uptime_percent', 5, 2)->default(100);
        $t->integer('downtime_minutes')->unsigned()->default(0);
        $t->decimal('avg_load', 5, 2)->default(0);
        $t->integer('memory_percent')->unsigned()->default(0);
        $t->integer('disk_percent')->unsigned()->default(0);

        $t->foreign('server_id')->references('id')->on('tblservers')->onDelete('cascade');
        $t->unique(['server_id', 'date']);
    });

    Capsule::schema()->create('mod_dashboard_cache', function($t) {
        $t->string('key', 64)->primary();
        $t->text('data');
        $t->integer('ttl')->unsigned();
        $t->timestamp('expires_at');
        $t->timestamp('created_at');
    });

    Capsule::schema()->create('mod_alert_rules', function($t) {
        $t->increments('id');
        $t->string('name');
        $t->string('metric'); // server_load, disk_space, etc.
        $t->string('condition'); // gt, lt, eq
        $t->decimal('threshold', 10, 2);
        $t->string('severity'); // info, warning, critical
        $t->boolean('active')->default(1);
        $t->timestamps();
    });

    Capsule::schema()->create('mod_alerts', function($t) {
        $t->increments('id');
        $t->integer('rule_id')->unsigned();
        $t->string('entity_type'); // server, service
        $t->integer('entity_id')->unsigned();
        $t->string('severity');
        $t->string('message');
        $t->string('status')->default('pending'); // pending, acknowledged, resolved
        $t->timestamp('created_at');
        $t->timestamp('acknowledged_at')->nullable();
        $t->integer('acknowledged_by')->unsigned()->nullable();
    });
}
```

### 7. Hook Integration

```php
<?php
// hooks.php
add_hook('DailyCronJob', 1, function($vars) {
    $monitor = new ServerMonitor();
    $servers = Capsule::table('tblservers')->where('disabled', 0)->get();

    foreach ($servers as $server) {
        $health = $monitor->getServerHealth($server->id);

        // Record metrics
        Capsule::table('mod_server_metrics')->updateOrInsert(
            ['server_id' => $server->id, 'date' => date('Y-m-d')],
            [
                'uptime_percent' => $health['online'] ? 100 : 0,
                'downtime_minutes' => $health['online'] ? 0 : 1440,
                'avg_load' => $health['load']['5min'] ?? 0,
                'memory_percent' => $health['memory']['percent'] ?? 0,
                'disk_percent' => $health['disk']['percent'] ?? 0,
            ]
        );

        // Check thresholds
        $alerts = new AlertChecker();
        $alerts->checkServerHealth($server->id, $health);
    }
});

add_hook('AfterModuleCreate', 1, function($vars) {
    $updater = new RealtimeUpdater();
    $updater->publishUpdate('services', [
        'action' => 'created',
        'service_id' => $vars['serviceid'],
    ]);
});
```

## Checklist

- [ ] Dashboard data collector
- [ ] Chart data providers
- [ ] Server health monitoring
- [ ] Real-time updates (Redis pub/sub)
- [ ] Alert system
- [ ] Admin dashboard UI
- [ ] Caching layer
- [ ] Cron job metrics collection
- [ ] Uptime tracking
- [ ] Notification integration

---

**Related Skills:**
- whmcs-monitoring
- whmcs-metrics-analytics
- whmcs-reporting
- whmcs-notification-builder
