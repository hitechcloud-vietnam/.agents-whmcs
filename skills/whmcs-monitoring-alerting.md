# WHMCS Monitoring and Alerting

## Skill Description
Set up monitoring and alerting for WHMCS modules to track performance, detect issues, and receive notifications.

## Prerequisites
- Monitoring tools (Prometheus, Grafana)
- Alerting system
- Log aggregation

## Step-by-Step Implementation

### 1. Monitoring Service
```php
<?php
// includes/monitoring/Monitor.php

namespace WHMCS\Module\YourModule\Monitoring;

class Monitor
{
    private array $metrics = [];
    private array $alerts = [];

    public function recordMetric(string $name, $value, array $tags = []): void
    {
        $this->metrics[] = [
            'name' => $name,
            'value' => $value,
            'tags' => $tags,
            'timestamp' => time()
        ];
    }

    public function checkThreshold(string $metric, float $threshold, string $operator = '>'): void
    {
        $recent = $this->getRecentMetric($metric);

        if ($recent === null) {
            return;
        }

        $triggered = match ($operator) {
            '>' => $recent > $threshold,
            '<' => $recent < $threshold,
            '>=' => $recent >= $threshold,
            '<=' => $recent <= $threshold,
            default => false
        };

        if ($triggered) {
            $this->triggerAlert($metric, $recent, $threshold);
        }
    }

    private function triggerAlert(string $metric, $value, float $threshold): void
    {
        $this->alerts[] = [
            'metric' => $metric,
            'value' => $value,
            'threshold' => $threshold,
            'triggered_at' => time()
        ];

        $this->sendNotification($metric, $value, $threshold);
    }

    private function sendNotification(string $metric, $value, float $threshold): void
    {
        // Send to Slack, email, etc.
        $message = sprintf(
            "Alert: %s is %s (threshold: %s)",
            $metric,
            $value,
            $threshold
        );

        logActivity($message);
    }

    private function getRecentMetric(string $name): ?float
    {
        foreach (array_reverse($this->metrics) as $metric) {
            if ($metric['name'] === $name) {
                return (float) $metric['value'];
            }
        }

        return null;
    }

    public function getMetrics(): array
    {
        return $this->metrics;
    }

    public function getAlerts(): array
    {
        return $this->alerts;
    }
}
```

### 2. Health Check Endpoint
```php
<?php
// api/health.php

namespace WHMCS\Module\YourModule\Api;

class HealthController
{
    public function check(): array
    {
        $checks = [
            'database' => $this->checkDatabase(),
            'cache' => $this->checkCache(),
            'disk_space' => $this->checkDiskSpace(),
            'dependencies' => $this->checkDependencies()
        ];

        $healthy = !in_array(false, array_column($checks, 'healthy'));

        return [
            'status' => $healthy ? 'healthy' : 'unhealthy',
            'checks' => $checks,
            'timestamp' => date('c')
        ];
    }

    private function checkDatabase(): array
    {
        try {
            global $db;
            $db->query('SELECT 1');

            return ['healthy' => true, 'message' => 'Database connection OK'];
        } catch (\Exception $e) {
            return ['healthy' => false, 'message' => $e->getMessage()];
        }
    }

    private function checkCache(): array
    {
        try {
            $redis = new \Redis();
            $redis->connect('127.0.0.1', 6379);

            return ['healthy' => true, 'message' => 'Cache connection OK'];
        } catch (\Exception $e) {
            return ['healthy' => false, 'message' => $e->getMessage()];
        }
    }

    private function checkDiskSpace(): array
    {
        $free = disk_free_space('/');
        $total = disk_total_space('/');
        $percentFree = ($free / $total) * 100;

        return [
            'healthy' => $percentFree > 10,
            'message' => sprintf('%.1f%% free space', $percentFree)
        ];
    }

    private function checkDependencies(): array
    {
        $requiredFiles = [
            'vendor/autoload.php',
            'config.php'
        ];

        foreach ($requiredFiles as $file) {
            if (!file_exists($file)) {
                return ['healthy' => false, 'message' => "Missing: {$file}"];
            }
        }

        return ['healthy' => true, 'message' => 'All dependencies present'];
    }
}
```

## Common Pitfalls and Solutions

| Pitfall | Solution |
|---------|----------|
| Too many alerts | Set appropriate thresholds |
| Missing metrics | Add comprehensive checks |
| Alert fatigue | Prioritize alerts |

## Security Considerations

1. **Secure health endpoint** - Protect from abuse
2. **Limit metric data** - Don't expose sensitive info
3. **Secure notifications** - Use secure channels

## Testing Checklist

- [ ] Test health check
- [ ] Test alerts
- [ ] Test notifications

## Reference Links

- [Prometheus](https://prometheus.io/)
- [Grafana](https://grafana.com/)
