# WHMCS Monitoring Setup Workflow
# Version: 1.0 | Created: 2026-05-28

## Purpose

Comprehensive guide to setting up monitoring for WHMCS including system health monitoring, service uptime tracking, performance metrics, alerting systems, and incident response procedures.

## Prerequisites

- WHMCS installation with admin access
- Monitoring tools (Prometheus, Grafana, etc.)
- Database access for metrics queries
- Email/SMS alerting capability

## Workflow Steps

### Step 1: Set Up System Health Monitoring

Configure comprehensive system health checks:

```php
// modules/addons/monitoring/monitoring.php

function monitoring_config(): array
{
    return [
        'name'        => 'WHMCS Monitoring',
        'description' => 'System health and performance monitoring',
        'version'     => '1.0',
    ];
}

function monitoring_activate(): array
{
    Capsule::schema()->create('mod_monitoring_metrics', function($t) {
        $t->increments('id');
        $t->string('metric_name');
        $t->string('metric_type'); // gauge, counter
        $t->decimal('value', 10, 4);
        $t->text('tags'); // JSON tags
        $t->timestamp('recorded_at')->useCurrent();
    });

    Capsule::schema()->create('mod_monitoring_alerts', function($t) {
        $t->increments('id');
        $t->string('alert_name');
        $t->string('severity'); // critical, warning, info
        $t->text('message');
        $t->boolean('acknowledged')->default(0);
        $t->timestamps();
    });

    return ['status' => 'success'];
}

/**
 * Record system metrics
 */
add_hook('DailyCronJob', 1, function(array $vars) {
    $collector = new MetricsCollector();
    $collector->collectAllMetrics();
});

class MetricsCollector
{
    public function collectAllMetrics(): void
    {
        $this->recordCpuUsage();
        $this->recordMemoryUsage();
        $this->recordDiskUsage();
        $this->recordDatabaseSize();
        $this->recordActiveUsers();
        $this->recordPendingTickets();
        $this->recordOverdueInvoices();
    }

    public function recordCpuUsage(): void
    {
        $loadAvg = sys_getloadavg();
        $cpuUsage = $loadAvg[0] * 100 / 4; // Assuming 4 cores

        Capsule::table('mod_monitoring_metrics')->insert([
            'metric_name' => 'system_cpu_usage',
            'metric_type'  => 'gauge',
            'value'       => min($cpuUsage, 100),
            'tags'        => json_encode(['host' => gethostname()]),
            'recorded_at'  => date('Y-m-d H:i:s'),
        ]);
    }

    public function recordMemoryUsage(): void
    {
        $memInfo = file_get_contents('/proc/meminfo');
        preg_match('/MemTotal:\s+(\d+)/', $memInfo, $total);
        preg_match('/MemAvailable:\s+(\d+)/', $memInfo, $available);

        $totalKb = $total[1] ?? 0;
        $availableKb = $available[1] ?? 0;
        $usedKb = $totalKb - $availableKb;
        $usagePercent = ($totalKb > 0) ? ($usedKb / $totalKb) * 100 : 0;

        Capsule::table('mod_monitoring_metrics')->insert([
            'metric_name' => 'system_memory_usage',
            'metric_type'  => 'gauge',
            'value'       => $usagePercent,
            'tags'        => json_encode(['host' => gethostname()]),
            'recorded_at'  => date('Y-m-d H:i:s'),
        ]);
    }

    public function recordDiskUsage(): void
    {
        $df = shell_exec('df -h / | tail -1');
        preg_match('/(\d+)%/', $df, $matches);
        $usagePercent = $matches[1] ?? 0;

        Capsule::table('mod_monitoring_metrics')->insert([
            'metric_name' => 'system_disk_usage',
            'metric_type'  => 'gauge',
            'value'       => $usagePercent,
            'tags'        => json_encode(['mount' => '/', 'host' => gethostname()]),
            'recorded_at'  => date('Y-m-d H:i:s'),
        ]);
    }

    public function recordDatabaseSize(): void
    {
        $pdo = Capsule::connection()->getPdo();
        $result = $pdo->query(
            "SELECT ROUND(SUM(data_length + index_length) / 1024 / 1024, 2) as size_mb
             FROM information_schema.tables
             WHERE table_schema = DATABASE()"
        );
        $dbSize = $result->fetch(PDO::FETCH_ASSOC)['size_mb'] ?? 0;

        Capsule::table('mod_monitoring_metrics')->insert([
            'metric_name' => 'database_size_mb',
            'metric_type'  => 'gauge',
            'value'       => $dbSize,
            'tags'        => json_encode(['database' => $_ENV['DB_NAME'] ?? 'whmcs']),
            'recorded_at'  => date('Y-m-d H:i:s'),
        ]);
    }

    public function recordActiveUsers(): void
    {
        $activeClients = Capsule::table('tblclients')
            ->where('status', 'Active')
            ->count();

        Capsule::table('mod_monitoring_metrics')->insert([
            'metric_name' => 'active_clients',
            'metric_type'  => 'gauge',
            'value'       => $activeClients,
            'tags'        => json_encode(['host' => gethostname()]),
            'recorded_at'  => date('Y-m-d H:i:s'),
        ]);
    }

    public function recordPendingTickets(): void
    {
        $pendingTickets = Capsule::table('tbltickets')
            ->whereIn('status', ['Open', 'Awaiting Reply'])
            ->count();

        Capsule::table('mod_monitoring_metrics')->insert([
            'metric_name' => 'pending_support_tickets',
            'metric_type'  => 'gauge',
            'value'       => $pendingTickets,
            'tags'        => json_encode(['host' => gethostname()]),
            'recorded_at'  => date('Y-m-d H:i:s'),
        ]);
    }

    public function recordOverdueInvoices(): void
    {
        $overdueInvoices = Capsule::table('tblinvoices')
            ->where('status', 'Overdue')
            ->count();

        Capsule::table('mod_monitoring_metrics')->insert([
            'metric_name' => 'overdue_invoices',
            'metric_type'  => 'gauge',
            'value'       => $overdueInvoices,
            'tags'        => json_encode(['host' => gethostname()]),
            'recorded_at'  => date('Y-m-d H:i:s'),
        ]);
    }
}
```

### Step 2: Implement Service Health Checks

Create service availability monitoring:

```php
// includes/classes/HealthChecker.php

class HealthChecker
{
    private array $checks = [];
    private array $thresholds = [
        'cpu_warning'       => 75,
        'cpu_critical'      => 90,
        'memory_warning'    => 80,
        'memory_critical'   => 95,
        'disk_warning'      => 80,
        'disk_critical'     => 95,
        'database_warning'  => 1000, // MB
        'database_critical'  => 5000, // MB
    ];

    public function runAllChecks(): array
    {
        return [
            'server'        => $this->checkServerHealth(),
            'database'       => $this->checkDatabaseHealth(),
            'whmcs'         => $this->checkWhmcsHealth(),
            'external'      => $this->checkExternalServices(),
            'payment'       => $this->checkPaymentGateways(),
        ];
    }

    private function checkServerHealth(): array
    {
        $health = [];

        // CPU Load
        $loadAvg = sys_getloadavg();
        $cpuPercent = min($loadAvg[0] * 25, 100);

        $health['cpu'] = $this->evaluateMetric(
            'CPU Usage',
            $cpuPercent,
            $this->thresholds['cpu_warning'],
            $this->thresholds['cpu_critical']
        );

        // Memory
        $memInfo = file_get_contents('/proc/meminfo');
        preg_match('/MemTotal:\s+(\d+)/', $memInfo, $total);
        preg_match('/MemAvailable:\s+(\d+)/', $memInfo, $available);
        $memPercent = (($total[1] - $available[1]) / $total[1]) * 100;

        $health['memory'] = $this->evaluateMetric(
            'Memory Usage',
            $memPercent,
            $this->thresholds['memory_warning'],
            $this->thresholds['memory_critical']
        );

        // Disk
        $diskPercent = disk_usage_percent();

        $health['disk'] = $this->evaluateMetric(
            'Disk Usage',
            $diskPercent,
            $this->thresholds['disk_warning'],
            $this->thresholds['disk_critical']
        );

        // Overall server health
        $health['overall'] = array_reduce($health, function($carry, $check) {
            return $carry === 'ok' ? $check['status'] : $carry;
        }, 'ok');

        return $health;
    }

    private function checkDatabaseHealth(): array
    {
        $health = [];

        // Connection test
        try {
            $pdo = Capsule::connection()->getPdo();
            $stmt = $pdo->query('SELECT 1');
            $health['connection'] = [
                'status' => 'ok',
                'message' => 'Database connection successful',
            ];
        } catch (Exception $e) {
            $health['connection'] = [
                'status' => 'critical',
                'message' => 'Database connection failed: ' . $e->getMessage(),
            ];
        }

        // Table integrity
        $corrupted = Capsule::select(
            "CHECK TABLE " . implode(', ', $this->getAllTables()) . " FOR UPGRADE"
        );

        $health['integrity'] = $this->evaluateMetric(
            'Tables',
            count(array_filter($corrupted, fn($t) => $t['Msg_type'] !== 'OK')),
            1,
            5
        );

        // Slow queries
        $slowQueries = $this->countSlowQueries();
        $health['slow_queries'] = $this->evaluateMetric(
            'Slow Queries',
            $slowQueries,
            10,
            50
        );

        return $health;
    }

    private function checkWhmcsHealth(): array
    {
        $health = [];

        // Cron job check
        $lastCron = Capsule::table('tblactivitylog')
            ->where('description', 'like', '%Cron%')
            ->orderBy('id', 'DESC')
            ->first();

        $hoursSinceCron = $lastCron
            ? (time() - strtotime($lastCron->date)) / 3600
            : -1;

        $health['cron'] = [
            'status' => ($hoursSinceCron < 1) ? 'ok' : (($hoursSinceCron < 24) ? 'warning' : 'critical'),
            'message' => "Last cron: " . round($hoursSinceCron, 1) . " hours ago",
            'hours_since' => $hoursSinceCron,
        ];

        // License status
        $license = $this->checkLicenseStatus();
        $health['license'] = [
            'status' => $license['valid'] ? 'ok' : 'critical',
            'message' => $license['message'],
        ];

        // Email queue
        $emailQueue = Capsule::table('mod_email_queue')->count();
        $health['email_queue'] = $this->evaluateMetric(
            'Email Queue',
            $emailQueue,
            100,
            500
        );

        return $health;
    }

    private function evaluateMetric(string $name, float $value, float $warning, float $critical): array
    {
        $status = 'ok';
        if ($value >= $critical) {
            $status = 'critical';
        } elseif ($value >= $warning) {
            $status = 'warning';
        }

        return [
            'name'    => $name,
            'status'  => $status,
            'value'   => $value,
            'warning' => $warning,
            'critical' => $critical,
            'message' => $this->getStatusMessage($name, $value, $status),
        ];
    }

    private function getStatusMessage(string $name, float $value, string $status): string
    {
        return match($status) {
            'ok'      => "{$name}: {$value}% - Normal",
            'warning' => "{$name}: {$value}% - Warning",
            'critical'=> "{$name}: {$value}% - Critical",
            default   => "{$name}: {$value}%",
        };
    }

    private function checkLicenseStatus(): array
    {
        // Check WHMCS license validity
        $licenseKey = Capsule::table('tblconfiguration')
            ->where('setting', 'LicenseKey')
            ->first()->value ?? '';

        // Simulated license check
        return [
            'valid'   => !empty($licenseKey),
            'message' => !empty($licenseKey) ? 'License active' : 'License key not found',
        ];
    }
}
```

### Step 3: Set Up Alerting System

Create comprehensive alerting:

```php
// includes/classes/AlertManager.php

class AlertManager
{
    private array $notificationChannels = [];
    private array $alertRules = [];

    public function __construct()
    {
        $this->loadAlertRules();
        $this->loadNotificationChannels();
    }

    private function loadAlertRules(): void
    {
        $this->alertRules = [
            [
                'name'       => 'High CPU Usage',
                'metric'     => 'system_cpu_usage',
                'condition'  => '>',
                'threshold'  => 90,
                'severity'   => 'critical',
                'channels'   => ['email', 'slack'],
                'cooldown'   => 300, // 5 minutes
            ],
            [
                'name'       => 'High Memory Usage',
                'metric'     => 'system_memory_usage',
                'condition'  => '>',
                'threshold'  => 85,
                'severity'   => 'warning',
                'channels'   => ['email'],
                'cooldown'   => 600,
            ],
            [
                'name'       => 'Disk Space Low',
                'metric'     => 'system_disk_usage',
                'condition'  => '>',
                'threshold'  => 90,
                'severity'   => 'critical',
                'channels'   => ['email', 'sms'],
                'cooldown'   => 300,
            ],
            [
                'name'       => 'Database Size Warning',
                'metric'     => 'database_size_mb',
                'condition'  => '>',
                'threshold'  => 1000,
                'severity'   => 'warning',
                'channels'   => ['email'],
                'cooldown'   => 3600,
            ],
            [
                'name'       => 'Overdue Invoices Spike',
                'metric'     => 'overdue_invoices',
                'condition'  => '>',
                'threshold'  => 50,
                'severity'   => 'warning',
                'channels'   => ['email'],
                'cooldown'   => 86400,
            ],
        ];
    }

    private function loadNotificationChannels(): void
    {
        $this->notificationChannels = [
            'email' => [
                'recipients' => [
                    'admin@yourcompany.com',
                    'ops@yourcompany.com',
                ],
            ],
            'slack' => [
                'webhook_url' => 'https://hooks.slack.com/services/XXX/YYY/ZZZ',
                'channel'    => '#alerts',
            ],
            'sms' => [
                'api_key'  => 'your_sms_api_key',
                'recipients' => ['+123456七八九零'],
            ],
        ];
    }

    /**
     * Check metrics and trigger alerts
     */
    public function checkAndAlert(): void
    {
        $currentMetrics = $this->getCurrentMetrics();

        foreach ($this->alertRules as $rule) {
            $this->evaluateRule($rule, $currentMetrics);
        }
    }

    private function evaluateRule(array $rule, array $metrics): void
    {
        $metricValue = $metrics[$rule['metric']] ?? null;

        if ($metricValue === null) {
            return;
        }

        // Check condition
        $triggered = match($rule['condition']) {
            '>'        => $metricValue > $rule['threshold'],
            '<'        => $metricValue < $rule['threshold'],
            '=='       => $metricValue == $rule['threshold'],
            '>='       => $metricValue >= $rule['threshold'],
            '<='       => $metricValue <= $rule['threshold'],
            default    => false,
        };

        if (!$triggered) {
            return;
        }

        // Check cooldown
        if ($this->isInCooldown($rule['name'], $rule['cooldown'])) {
            return;
        }

        // Trigger alert
        $this->triggerAlert($rule, $metricValue);
    }

    private function isInCooldown(string $alertName, int $cooldownSeconds): bool
    {
        $lastAlert = Capsule::table('mod_monitoring_alerts')
            ->where('alert_name', $alertName)
            ->where('acknowledged', 1)
            ->orderBy('created_at', 'DESC')
            ->first();

        if (!$lastAlert) {
            return false;
        }

        $timeSinceLastAlert = time() - strtotime($lastAlert->created_at);
        return $timeSinceLastAlert < $cooldownSeconds;
    }

    private function triggerAlert(array $rule, float $metricValue): void
    {
        $alertId = Capsule::table('mod_monitoring_alerts')->insertGetId([
            'alert_name' => $rule['name'],
            'severity'   => $rule['severity'],
            'message'    => "Alert triggered: {$rule['name']}. Current value: {$metricValue}",
            'acknowledged' => 0,
            'created_at' => date('Y-m-d H:i:s'),
        ]);

        // Send notifications
        foreach ($rule['channels'] as $channel) {
            $this->sendNotification($channel, $rule, $metricValue, $alertId);
        }

        logActivity("Alert triggered: {$rule['name']} (severity: {$rule['severity']})");
    }

    private function sendNotification(string $channel, array $rule, float $value, int $alertId): void
    {
        $channelConfig = $this->notificationChannels[$channel] ?? null;

        if (!$channelConfig) {
            return;
        }

        $message = $this->buildAlertMessage($rule, $value);

        switch ($channel) {
            case 'email':
                $this->sendEmailAlert($channelConfig, $message, $alertId);
                break;
            case 'slack':
                $this->sendSlackAlert($channelConfig, $message);
                break;
            case 'sms':
                $this->sendSmsAlert($channelConfig, $message);
                break;
        }
    }

    private function buildAlertMessage(array $rule, float $value): string
    {
        $emoji = match($rule['severity']) {
            'critical' => 'CRITICAL',
            'warning'  => 'WARNING',
            default    => 'INFO',
        };

        return "[{$emoji}] WHMCS Alert\n" .
               "Alert: {$rule['name']}\n" .
               "Current Value: {$value}\n" .
               "Threshold: {$rule['threshold']}\n" .
               "Time: " . date('Y-m-d H:i:s');
    }

    private function sendEmailAlert(array $config, string $message, int $alertId): void
    {
        foreach ($config['recipients'] as $recipient) {
            sendEmail([
                'to'      => $recipient,
                'subject' => "WHMCS Alert: " . date('Y-m-d H:i'),
                'body'    => $message,
            ]);
        }
    }

    private function sendSlackAlert(array $config, string $message): void
    {
        $slackColor = match(true) {
            str_contains($message, 'CRITICAL') => 'danger',
            str_contains($message, 'WARNING')  => 'warning',
            default                            => 'good',
        };

        $payload = json_encode([
            'channel' => $config['channel'],
            'attachments' => [[
                'color' => $slackColor,
                'text'  => $message,
            ]],
        ]);

        $ch = curl_init($config['webhook_url']);
        curl_setopt_array($ch, [
            CURLOPT_POST => true,
            CURLOPT_POSTFIELDS => $payload,
            CURLOPT_HTTPHEADER => ['Content-Type: application/json'],
        ]);
        curl_exec($ch);
        curl_close($ch);
    }

    private function sendSmsAlert(array $config, string $message): void
    {
        foreach ($config['recipients'] as $phone) {
            // SMS API integration
            $this->sendSms($phone, $message, $config['api_key']);
        }
    }

    private function getCurrentMetrics(): array
    {
        $metrics = Capsule::table('mod_monitoring_metrics')
            ->where('recorded_at', '>', date('Y-m-d H:i:s', strtotime('-5 minutes')))
            ->get()
            ->groupBy('metric_name');

        $current = [];
        foreach ($metrics as $name => $values) {
            $latest = collect($values)->sortByDesc('recorded_at')->first();
            $current[$name] = $latest->value;
        }

        return $current;
    }
}
```

### Step 4: Create Monitoring Dashboard

Build real-time monitoring interface:

```php
// modules/addons/monitoring/views/dashboard.php

<div class="monitoring-dashboard">
    <div class="dashboard-header">
        <h1>WHHCS System Monitoring</h1>
        <p>Last updated: <?= $lastUpdate ?></p>
        <button onclick="refreshDashboard()">Refresh</button>
    </div>

    <div class="health-overview">
        <h2>System Health</h2>
        <?php foreach ($healthStatus as $component => $status): ?>
        <div class="health-card <?= $status['overall'] ?>">
            <h3><?= ucfirst($component) ?></h3>
            <p class="status-badge <?= $status['overall'] ?>">
                <?= strtoupper($status['overall']) ?>
            </p>
        </div>
        <?php endforeach; ?>
    </div>

    <div class="metrics-grid">
        <div class="metric-chart">
            <h3>CPU Usage (24h)</h3>
            <canvas id="cpuChart"></canvas>
        </div>

        <div class="metric-chart">
            <h3>Memory Usage (24h)</h3>
            <canvas id="memoryChart"></canvas>
        </div>

        <div class="metric-chart">
            <h3>Disk Usage (24h)</h3>
            <canvas id="diskChart"></canvas>
        </div>

        <div class="metric-chart">
            <h3>Active Clients (7d)</h3>
            <canvas id="clientsChart"></canvas>
        </div>
    </div>

    <div class="alerts-section">
        <h2>Active Alerts</h2>
        <?php if (empty($activeAlerts)): ?>
        <p class="no-alerts">No active alerts</p>
        <?php else: ?>
        <table class="alerts-table">
            <thead>
                <tr>
                    <th>Severity</th>
                    <th>Alert</th>
                    <th>Message</th>
                    <th>Time</th>
                    <th>Actions</th>
                </tr>
            </thead>
            <tbody>
                <?php foreach ($activeAlerts as $alert): ?>
                <tr class="alert-row <?= $alert->severity ?>">
                    <td><span class="severity-badge <?= $alert->severity ?>">
                        <?= strtoupper($alert->severity) ?>
                    </span></td>
                    <td><?= $alert->alert_name ?></td>
                    <td><?= $alert->message ?></td>
                    <td><?= $alert->created_at ?></td>
                    <td>
                        <form method="POST" action="?module=monitoring&action=acknowledge">
                            <input type="hidden" name="alert_id" value="<?= $alert->id ?>">
                            <button type="submit">Acknowledge</button>
                        </form>
                    </td>
                </tr>
                <?php endforeach; ?>
            </tbody>
        </table>
        <?php endif; ?>
    </div>

    <div class="server-details">
        <h2>Server Details</h2>
        <table class="details-table">
            <tr>
                <th>Hostname</th><td><?= $serverInfo['hostname'] ?></td>
            </tr>
            <tr>
                <th>Uptime</th><td><?= $serverInfo['uptime'] ?></td>
            </tr>
            <tr>
                <th>Load Average</th><td><?= implode(', ', $serverInfo['load_average']) ?></td>
            </tr>
            <tr>
                <th>PHP Version</th><td><?= PHP_VERSION ?></td>
            </tr>
            <tr>
                <th>MySQL Version</th><td><?= $serverInfo['mysql_version'] ?></td>
            </tr>
        </table>
    </div>
</div>

<script>
async function refreshDashboard() {
    const response = await fetch('?module=monitoring&action=api');
    const data = await response.json();
    updateDashboard(data);
}

function updateDashboard(data) {
    // Update health status
    // Update charts
    // Update alerts
}
</script>
```

### Step 5: Integrate with External Monitoring Tools

Connect to external monitoring systems:

```php
// includes/classes/ExternalMonitoringIntegration.php

class ExternalMonitoringIntegration
{
    /**
     * Export metrics to Prometheus
     */
    public function exportToPrometheus(): string
    {
        $output = [];

        $metrics = Capsule::table('mod_monitoring_metrics')
            ->where('recorded_at', '>', date('Y-m-d H:i:s', strtotime('-1 hour')))
            ->get();

        foreach ($metrics as $metric) {
            $tags = json_decode($metric->tags, true);
            $labels = [];

            foreach ($tags as $key => $value) {
                $labels[] = "{$key}=\"{$value}\"";
            }

            $labelStr = empty($labels) ? '' : '{' . implode(',', $labels) . '}';
            $output[] = "whmcs_{$metric->metric_name}{$labelStr} {$metric->value}";
        }

        return implode("\n", $output);
    }

    /**
     * Prometheus metrics endpoint
     */
    public function prometheusMetricsEndpoint(): void
    {
        header('Content-Type: text/plain');

        $output = "# HELP whmcs_system_metrics WHMCS system metrics\n";
        $output .= "# TYPE whmcs_system_metrics gauge\n";
        $output .= $this->exportToPrometheus();

        echo $output;
        exit;
    }

    /**
     * Export to Datadog
     */
    public function exportToDatadog(): void
    {
        $apiKey = Capsule::table('tblconfiguration')
            ->where('setting', 'DatadogAPIKey')
            ->first()->value ?? '';

        if (empty($apiKey)) {
            return;
        }

        $metrics = Capsule::table('mod_monitoring_metrics')
            ->where('recorded_at', '>', date('Y-m-d H:i:s', strtotime('-5 minutes')))
            ->get();

        $series = [];

        foreach ($metrics as $metric) {
            $tags = json_decode($metric->tags, true) ?? [];

            $series[] = [
                'metric' => 'whmcs.' . $metric->metric_name,
                'points' => [[strtotime($metric->recorded_at), $metric->value]],
                'type'   => 'gauge',
                'tags'   => $tags,
            ];
        }

        $payload = json_encode($series);

        $ch = curl_init('https://api.datadoghq.com/api/v1/series');
        curl_setopt_array($ch, [
            CURLOPT_POST => true,
            CURLOPT_POSTFIELDS => $payload,
            CURLOPT_HTTPHEADER => [
                'Content-Type: application/json',
                "DD-API-KEY: {$apiKey}",
            ],
        ]);
        curl_exec($ch);
        curl_close($ch);
    }

    /**
     * Prometheus scrape configuration
     */
    public function getPrometheusConfig(): array
    {
        return [
            'job_name' => 'whmcs',
            'static_configs' => [
                'targets' => ['your-whmcs.com:443'],
                'labels'  => ['instance' => 'whmcs-primary'],
            ],
            'scrape_interval' => '60s',
            'metrics_path'    => '/metrics.php',
            'scheme'           => 'https',
        ];
    }
}

/**
 * Cron to run monitoring checks
 */
add_hook('DailyCronJob', 1, function() {
    $healthChecker = new HealthChecker();
    $healthResults = $healthChecker->runAllChecks();

    $alertManager = new AlertManager();
    $alertManager->checkAndAlert();

    $externalMonitoring = new ExternalMonitoringIntegration();
    $externalMonitoring->exportToDatadog();

    logActivity("Monitoring check completed at " . date('Y-m-d H:i:s'));
});
```

---

## Best Practices

1. **Set realistic thresholds** - Adjust alert thresholds to avoid noise
2. **Use multiple notification channels** - Ensure critical alerts reach administrators
3. **Implement alert acknowledgment** - Track alert management and resolution
4. **Monitor proactively** - Check system health before users notice issues
5. **Retain historical data** - Keep metrics for trend analysis
6. **Automate responses** - Implement auto-remediation for common issues
7. **Test alerts regularly** - Verify alerting system is working
8. **Document runbooks** - Provide procedures for handling each alert type
9. **Correlate metrics** - Link related metrics to find root causes
10. **Review and tune** - Regularly review alert effectiveness

---

## Verification Checklist

- [ ] System metrics collected and stored
- [ ] Health checker runs successfully
- [ ] All alert rules evaluated correctly
- [ ] Email notifications sent properly
- [ ] Slack notifications working
- [ ] Dashboard displays current status
- [ ] Historical data retained for 30+ days
- [ ] Prometheus metrics endpoint accessible
- [ ] External monitoring integration functional
- [ ] Alert acknowledgment tracking works
- [ ] On-call rotation configured
- [ ] Alert escalation working
