# WHMCS Monitoring Setup Workflow

## Overview
This workflow covers setting up comprehensive monitoring for WHMCS installations.

## Step 1: Monitoring Service

```php
<?php
// src/Service/MonitoringService.php

namespace WHMCS\Module\Addon\YourModule\Service;

use WHMCS\Database\Capsule;

class MonitoringService
{
    private $metrics = [];

    public function collectMetrics(): array
    {
        $metrics = [
            'timestamp' => date('Y-m-d H:i:s'),
            'system' => $this->getSystemMetrics(),
            'database' => $this->getDatabaseMetrics(),
            'application' => $this->getApplicationMetrics(),
            'services' => $this->getServiceMetrics()
        ];

        $this->storeMetrics($metrics);

        return $metrics;
    }

    private function getSystemMetrics(): array
    {
        return [
            'cpu_usage' => sys_getloadavg()[0] ?? 0,
            'memory_usage' => $this->getMemoryUsage(),
            'disk_usage' => disk_free_space('/') / disk_total_space('/'),
            'uptime' => shell_exec('uptime -p') ?: 'unknown'
        ];
    }

    private function getMemoryUsage(): float
    {
        $memory = memory_get_usage(true);
        $limit = ini_get('memory_limit');
        $limitBytes = $this->parseSize($limit);

        return round(($memory / $limitBytes) * 100, 2);
    }

    private function getDatabaseMetrics(): array
    {
        try {
            $queries = Capsule::connection()->select('SHOW STATUS LIKE "Questions"');
            $connections = Capsule::connection()->select('SHOW STATUS LIKE "Threads_connected"');
            $slowQueries = Capsule::connection()->select('SHOW STATUS LIKE "Slow_queries"');

            return [
                'total_queries' => $queries[0]->Value ?? 0,
                'active_connections' => $connections[0]->Value ?? 0,
                'slow_queries' => $slowQueries[0]->Value ?? 0
            ];
        } catch (\Exception $e) {
            return ['error' => $e->getMessage()];
        }
    }

    private function getApplicationMetrics(): array
    {
        return [
            'active_sessions' => session_status() === PHP_SESSION_ACTIVE ? 1 : 0,
            'memory_usage_mb' => round(memory_get_usage(true) / 1024 / 1024, 2),
            'php_version' => PHP_VERSION,
            'whmcs_version' => Capsule::config('version')
        ];
    }

    private function getServiceMetrics(): array
    {
        $services = [
            'active_services' => Capsule::table('tblhosting')
                ->where('domainstatus', 'Active')
                ->count(),
            'suspended_services' => Capsule::table('tblhosting')
                ->where('domainstatus', 'Suspended')
                ->count(),
            'pending_orders' => Capsule::table('tblorders')
                ->where('status', 'Pending')
                ->count(),
            'open_tickets' => Capsule::table('tbltickets')
                ->where('status', 'Open')
                ->count()
        ];

        return $services;
    }

    private function storeMetrics(array $metrics): void
    {
        Capsule::table('mod_monitoring_metrics')->insert([
            'metrics' => json_encode($metrics),
            'created_at' => date('Y-m-d H:i:s')
        ]);

        // Keep only last 7 days
        Capsule::table('mod_monitoring_metrics')
            ->where('created_at', '<', date('Y-m-d H:i:s', strtotime('-7 days')))
            ->delete();
    }

    private function parseSize(string $size): int
    {
        $unit = preg_replace('/[^a-zA-Z]/', '', $size);
        $value = (int)$size;

        return match (strtoupper($unit)) {
            'G' => $value * 1024 * 1024 * 1024,
            'M' => $value * 1024 * 1024,
            'K' => $value * 1024,
            default => $value
        };
    }

    public function checkHealth(): array
    {
        $checks = [
            'database' => $this->checkDatabaseHealth(),
            'filesystem' => $this->checkFilesystemHealth(),
            'cache' => $this->checkCacheHealth(),
            'sessions' => $this->checkSessionHealth()
        ];

        $healthy = !in_array(false, array_column($checks, 'healthy'));

        return [
            'healthy' => $healthy,
            'checks' => $checks,
            'checked_at' => date('Y-m-d H:i:s')
        ];
    }

    private function checkDatabaseHealth(): array
    {
        try {
            Capsule::connection()->getPdo();
            return ['healthy' => true, 'message' => 'Database connected'];
        } catch (\Exception $e) {
            return ['healthy' => false, 'message' => $e->getMessage()];
        }
    }

    private function checkFilesystemHealth(): array
    {
        $writableDirs = ['storage', 'attachments', 'downloads'];
        $issues = [];

        foreach ($writableDirs as $dir) {
            $path = dirname(__DIR__, 3) . '/' . $dir;
            if (!is_writable($path)) {
                $issues[] = "$dir is not writable";
            }
        }

        return [
            'healthy' => empty($issues),
            'message' => empty($issues) ? 'Filesystem OK' : implode(', ', $issues)
        ];
    }

    private function checkCacheHealth(): array
    {
        $cacheDir = dirname(__DIR__, 3) . '/storage/cache';
        return [
            'healthy' => is_dir($cacheDir) && is_writable($cacheDir),
            'message' => is_writable($cacheDir) ? 'Cache OK' : 'Cache not writable'
        ];
    }

    private function checkSessionHealth(): array
    {
        $sessionDir = ini_get('session.save_path');
        return [
            'healthy' => !empty($sessionDir) && is_writable($sessionDir),
            'message' => !empty($sessionDir) ? 'Sessions OK' : 'Session path not configured'
        ];
    }
}
```

## Step 2: Prometheus Metrics Exporter

```php
<?php
// public/metrics.php

require_once __DIR__ . '/init.php';

use WHMCS\Module\Addon\YourModule\Service\MonitoringService;

header('Content-Type: text/plain');

$monitoring = new MonitoringService();
$metrics = $monitoring->collectMetrics();

// Format as Prometheus metrics
echo "# HELP whmcs_active_services Number of active services\n";
echo "# TYPE whmcs_active_services gauge\n";
echo "whmcs_active_services " . $metrics['services']['active_services'] . "\n";

echo "# HELP whmcs_suspended_services Number of suspended services\n";
echo "# TYPE whmcs_suspended_services gauge\n";
echo "whmcs_suspended_services " . $metrics['services']['suspended_services'] . "\n";

echo "# HELP whmcs_pending_orders Number of pending orders\n";
echo "# TYPE whmcs_pending_orders gauge\n";
echo "whmcs_pending_orders " . $metrics['services']['pending_orders'] . "\n";

echo "# HELP whmcs_open_tickets Number of open tickets\n";
echo "# TYPE whmcs_open_tickets gauge\n";
echo "whmcs_open_tickets " . $metrics['services']['open_tickets'] . "\n";

echo "# HELP whmcs_memory_usage_bytes Memory usage in bytes\n";
echo "# TYPE whmcs_memory_usage_bytes gauge\n";
echo "whmcs_memory_usage_bytes " . memory_get_usage(true) . "\n";
```

## Verification Checklist

- [ ] Monitoring service implemented
- [ ] Metrics collection working
- [ ] Health checks implemented
- [ ] Prometheus exporter configured
- [ ] Grafana dashboards set up
- [ ] Alerts configured
