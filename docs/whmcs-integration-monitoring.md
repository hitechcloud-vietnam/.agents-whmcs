# WHMCS Monitoring Integration

Complete guide for system monitoring and alerting.

## Overview

Implement monitoring solutions for WHMCS health and performance.

## Health Checks

### Service Health Check

```php
<?php
/**
 * Health check endpoint
 */
function performHealthCheck(): array
{
    $checks = [
        'database' => checkDatabaseHealth(),
        'disk_space' => checkDiskSpace(),
        'cache' => checkCacheHealth(),
        'external_services' => checkExternalServices(),
        'cron_jobs' => checkCronJobs(),
    ];
    
    $overallStatus = 'healthy';
    foreach ($checks as $check) {
        if ($check['status'] === 'critical') {
            $overallStatus = 'critical';
            break;
        } elseif ($check['status'] === 'warning' && $overallStatus !== 'critical') {
            $overallStatus = 'warning';
        }
    }
    
    return [
        'status' => $overallStatus,
        'timestamp' => date('c'),
        'checks' => $checks,
    ];
}

/**
 * Database health check
 */
function checkDatabaseHealth(): array
{
    try {
        $start = microtime(true);
        Capsule::select('SELECT 1');
        $latency = round((microtime(true) - $start) * 1000, 2);
        
        // Check connection pool
        $connections = Capsule::select('SHOW STATUS LIKE "Threads_connected"');
        $connectionCount = $connections[0]->Value ?? 0;
        
        $status = 'healthy';
        $message = "Connected ({$latency}ms)";
        
        if ($latency > 1000) {
            $status = 'warning';
            $message = "High latency ({$latency}ms)";
        }
        
        if ($connectionCount > 100) {
            $status = 'warning';
            $message .= ", {$connectionCount} connections";
        }
        
        return [
            'status' => $status,
            'message' => $message,
            'latency_ms' => $latency,
            'connections' => (int)$connectionCount,
        ];
        
    } catch (Exception $e) {
        return [
            'status' => 'critical',
            'message' => 'Database connection failed: ' . $e->getMessage(),
        ];
    }
}

/**
 * Disk space check
 */
function checkDiskSpace(): array
{
    $freeSpace = disk_free_space(__DIR__);
    $totalSpace = disk_total_space(__DIR__);
    $usedPercent = (($totalSpace - $freeSpace) / $totalSpace) * 100;
    
    $status = 'healthy';
    if ($usedPercent > 95) {
        $status = 'critical';
    } elseif ($usedPercent > 85) {
        $status = 'warning';
    }
    
    return [
        'status' => $status,
        'used_percent' => round($usedPercent, 2),
        'free_bytes' => $freeSpace,
        'total_bytes' => $totalSpace,
    ];
}

/**
 * Cache health check
 */
function checkCacheHealth(): array
{
    $cache = \WHMCS\File\Cache::factory('HealthCheck');
    $testKey = 'health_check_' . uniqid();
    
    try {
        $cache->store($testKey, 'test_value', 60);
        $value = $cache->retrieve($testKey);
        $cache->delete($testKey);
        
        if ($value === 'test_value') {
            return [
                'status' => 'healthy',
                'message' => 'Cache working',
            ];
        }
        
        return [
            'status' => 'warning',
            'message' => 'Cache read/write mismatch',
        ];
        
    } catch (Exception $e) {
        return [
            'status' => 'critical',
            'message' => 'Cache unavailable: ' . $e->getMessage(),
        ];
    }
}
```

## Performance Monitoring

### Metrics Collector

```php
<?php
/**
 * Performance metrics collector
 */
class MetricsCollector
{
    private array $metrics = [];
    
    /**
     * Record a metric
     */
    public function record(string $name, float $value, array $tags = []): void
    {
        $this->metrics[] = [
            'name' => $name,
            'value' => $value,
            'tags' => $tags,
            'timestamp' => microtime(true),
        ];
    }
    
    /**
     * Record timing
     */
    public function timing(string $name, float $duration, array $tags = []): void
    {
        $this->record($name, $duration, array_merge($tags, ['unit' => 'ms']));
    }
    
    /**
     * Increment counter
     */
    public function increment(string $name, int $value = 1, array $tags = []): void
    {
        $this->record($name, $value, array_merge($tags, ['type' => 'counter']));
    }
    
    /**
     * Flush metrics
     */
    public function flush(): array
    {
        $metrics = $this->metrics;
        $this->metrics = [];
        return $metrics;
    }
}

/**
 * Monitor request performance
 */
function monitorRequestPerformance(): void
{
    global $startTime;
    
    $duration = (microtime(true) - $startTime) * 1000;
    $metrics = new MetricsCollector();
    
    $metrics->timing('http_request_duration', $duration, [
        'uri' => $_SERVER['REQUEST_URI'] ?? 'cli',
        'method' => $_SERVER['REQUEST_METHOD'] ?? 'CLI',
    ]);
    
    $metrics->increment('http_requests_total', 1, [
        'status' => http_response_code() >= 400 ? 'error' : 'success',
    ]);
    
    // Store metrics
    foreach ($metrics->flush() as $metric) {
        Capsule::table('mod_performance_metrics')->insert($metric);
    }
}
```

## Prometheus Integration

```php
<?php
/**
 * Prometheus metrics exporter
 */
class PrometheusExporter
{
    private array $metrics = [];
    
    /**
     * Add gauge metric
     */
    public function gauge(string $name, float $value, array $labels = []): void
    {
        $this->metrics[] = [
            'type' => 'gauge',
            'name' => $name,
            'value' => $value,
            'labels' => $labels,
        ];
    }
    
    /**
     * Add counter metric
     */
    public function counter(string $name, float $value, array $labels = []): void
    {
        $this->metrics[] = [
            'type' => 'counter',
            'name' => $name,
            'value' => $value,
            'labels' => $labels,
        ];
    }
    
    /**
     * Export in Prometheus format
     */
    public function export(): string
    {
        $output = '';
        
        $grouped = [];
        foreach ($this->metrics as $metric) {
            $key = $metric['name'];
            if (!isset($grouped[$key])) {
                $grouped[$key] = [];
            }
            $grouped[$key][] = $metric;
        }
        
        foreach ($grouped as $name => $metrics) {
            $type = $metrics[0]['type'];
            $output .= "# HELP {$name} {$name}\n";
            $output .= "# TYPE {$name} {$type}\n";
            
            foreach ($metrics as $metric) {
                $labels = '';
                if (!empty($metric['labels'])) {
                    $labelStr = implode(',', array_map(
                        fn($k, $v) => "{$k}=\"{$v}\"",
                        array_keys($metric['labels']),
                        $metric['labels']
                    ));
                    $labels = "{{$labelStr}}";
                }
                
                $output .= "{$name}{$labels} {$metric['value']}\n";
            }
        }
        
        return $output;
    }
}

/**
 * Prometheus endpoint
 */
function prometheusMetrics(): void
{
    header('Content-Type: text/plain; version=0.0.4');
    
    $exporter = new PrometheusExporter();
    
    // System metrics
    $exporter->gauge('whmcs_disk_free_bytes', disk_free_space(__DIR__));
    $exporter->gauge('whmcs_disk_total_bytes', disk_total_space(__DIR__));
    
    // Database metrics
    $connections = Capsule::select('SHOW STATUS LIKE "Threads_connected"');
    $exporter->gauge('whmcs_db_connections', $connections[0]->Value ?? 0);
    
    // Service metrics
    $exporter->gauge('whmcs_active_services', 
        Capsule::table('tblhosting')->where('domainstatus', 'Active')->count());
    
    $exporter->gauge('whmcs_pending_services',
        Capsule::table('tblhosting')->where('domainstatus', 'Pending')->count());
    
    // Client metrics
    $exporter->gauge('whmcs_total_clients',
        Capsule::table('tblclients')->count());
    
    echo $exporter->export();
}
```

## Alerting

### Alert Manager

```php
<?php
/**
 * Alert manager
 */
class AlertManager
{
    private array $alerts = [];
    
    /**
     * Send alert
     */
    public function alert(string $severity, string $title, string $message, array $context = []): void
    {
        $alert = [
            'severity' => $severity,
            'title' => $title,
            'message' => $message,
            'context' => $context,
            'timestamp' => date('c'),
        ];
        
        $this->alerts[] = $alert;
        
        // Store alert
        Capsule::table('mod_alerts')->insert([
            'severity' => $severity,
            'title' => $title,
            'message' => $message,
            'context' => json_encode($context),
            'created_at' => date('Y-m-d H:i:s'),
        ]);
        
        // Notify based on severity
        if ($severity === 'critical') {
            $this->notifySlack($alert);
            $this->notifyEmail($alert);
        } elseif ($severity === 'warning') {
            $this->notifyEmail($alert);
        }
    }
    
    /**
     * Send to Slack
     */
    private function notifySlack(array $alert): void
    {
        $color = match($alert['severity']) {
            'critical' => 'danger',
            'warning' => 'warning',
            default => 'good',
        };
        
        $webhook = getenv('SLACK_WEBHOOK_URL');
        if (!$webhook) return;
        
        $payload = [
            'attachments' => [[
                'color' => $color,
                'title' => $alert['title'],
                'text' => $alert['message'],
                'fields' => array_map(
                    fn($k, $v) => ['title' => $k, 'value' => $v, 'short' => true],
                    array_keys($alert['context']),
                    $alert['context']
                ),
                'footer' => 'WHMCS Alert',
                'ts' => time(),
            ]],
        ];
        
        $ch = curl_init($webhook);
        curl_setopt_array($ch, [
            CURLOPT_POST => true,
            CURLOPT_POSTFIELDS => json_encode($payload),
            CURLOPT_RETURNTRANSFER => true,
        ]);
        curl_exec($ch);
        curl_close($ch);
    }
    
    /**
     * Send email notification
     */
    private function notifyEmail(array $alert): void
    {
        $adminEmail = Capsule::table('tblconfiguration')
            ->where('setting', 'SupportEmail')
            ->first()->value ?? '';
        
        if (!$adminEmail) return;
        
        mail($adminEmail, 
            "[{$alert['severity']}] {$alert['title']}",
            $alert['message'] . "\n\n" . json_encode($alert['context'], JSON_PRETTY_PRINT)
        );
    }
}
```

### Alerting Hooks

```php
<?php
/**
 * Monitor service health
 */
function checkServiceAlerts(): void
{
    $alertManager = new AlertManager();
    
    // Check for overdue invoices
    $overdueCount = Capsule::table('tblinvoices')
        ->where('status', 'Overdue')
        ->where('duedate', '<', date('Y-m-d', strtotime('-30 days')))
        ->count();
    
    if ($overdueCount > 100) {
        $alertManager->alert('warning', 'High Overdue Invoice Count', 
            "There are {$overdueCount} invoices overdue for more than 30 days.");
    }
    
    // Check for failed cron
    $lastCron = Capsule::table('tblactivitylog')
        ->where('description', 'LIKE', '%Cron Job%')
        ->orderBy('id', 'desc')
        ->first();
    
    if ($lastCron) {
        $hoursSince = (time() - strtotime($lastCron->date)) / 3600;
        
        if ($hoursSince > 25) {
            $alertManager->alert('critical', 'Cron Job Not Running',
                "Last cron job was {$hoursSince} hours ago.", [
                    'last_cron' => $lastCron->date,
                    'hours_ago' => round($hoursSince, 1),
                ]);
        }
    }
    
    // Check disk space
    $usedPercent = (disk_total_space(__DIR__) - disk_free_space(__DIR__)) / disk_total_space(__DIR__) * 100;
    
    if ($usedPercent > 95) {
        $alertManager->alert('critical', 'Disk Space Critical',
            "Disk usage is at {$usedPercent}%.");
    }
}
```

## Dashboard Widget

```php
<?php
/**
 * Monitoring dashboard widget
 */
add_hook('AdminHomepage', 1, function($vars) {
    $health = performHealthCheck();
    $recentAlerts = Capsule::table('mod_alerts')
        ->where('created_at', '>', date('Y-m-d H:i:s', strtotime('-24 hours')))
        ->count();
    
    return [
        'name' => 'System Health',
        'template' => 'monitoring-widget',
        'vars' => [
            'health_status' => $health['status'],
            'recent_alerts' => $recentAlerts,
            'checks' => $health['checks'],
        ],
    ];
});
```

## Best Practices

1. **Health endpoints** - Provide monitoring endpoints
2. **Set thresholds** - Define alert levels
3. **Multiple channels** - Use multiple notification methods
4. **Rate limiting** - Don't overwhelm notification systems
5. **Historical data** - Store metrics for analysis
6. **Dashboard** - Visualize system health

## Related Documentation

- [whmcs-integration-logging.md](whmcs-integration-logging.md)
- [whmcs-integration-api.md](whmcs-integration-api.md)
