# WHMCS API Monitoring

## Skill Description
Implement API health monitoring and metrics collection for WHMCS modules including endpoint latency tracking, error rate monitoring, and availability checks.

## Prerequisites
- WHMCS 7.0+ installation
- PHP 7.4+
- Database access for metrics storage
- Optional: Redis for time-series data

## Step-by-Step Implementation

### 1. Metrics Collector
```php
<?php
// includes/monitoring/MetricsCollector.php

namespace WHMCS\Module\YourModule\Monitoring;

class MetricsCollector
{
    private array $metrics = [];
    private float $requestStartTime;
    private ?\Redis $redis = null;

    public function __construct()
    {
        $this->requestStartTime = microtime(true);
        $this->initializeRedis();
    }

    private function initializeRedis(): void
    {
        $config = $this->getConfig();

        if (empty($config['redis_host'])) {
            return;
        }

        try {
            $this->redis = new \Redis();
            $this->redis->connect($config['redis_host'], $config['redis_port'] ?? 6379);
        } catch (\Exception $e) {
            logActivity('Redis connection failed for monitoring: ' . $e->getMessage());
        }
    }

    private function getConfig(): array
    {
        $settings = getWHMCSModuleConfig('yourmodule');
        return [
            'redis_host' => $settings['redis_host'] ?? null,
            'redis_port' => $settings['redis_port'] ?? 6379
        ];
    }

    public function recordRequest(
        string $endpoint,
        string $method,
        int $statusCode,
        float $duration,
        ?string $userId = null
    ): void {
        $timestamp = time();
        $dateKey = date('Y-m-d');
        $hourKey = date('Y-m-d-H');

        $this->incrementCounter("requests:total:{$dateKey}");
        $this->incrementCounter("requests:{$method}:{$dateKey}");
        $this->incrementCounter("requests:status:{$statusCode}:{$dateKey}");

        if ($this->redis) {
            $this->recordToRedis($endpoint, $method, $statusCode, $duration, $timestamp);
        }

        $this->storeToDatabase($endpoint, $method, $statusCode, $duration, $userId);
    }

    private function incrementCounter(string $key): void
    {
        if ($this->redis) {
            $this->redis->incr($key);
        }
    }

    private function recordToRedis(
        string $endpoint,
        string $method,
        int $statusCode,
        float $duration,
        int $timestamp
    ): void {
        $hourKey = "metrics:" . date('Y-m-d-H');

        // Use sorted sets for time-series data
        $this->redis->zAdd("{$hourKey}:latency", $timestamp, "{$endpoint}:{$duration}");
        $this->redis->zAdd("{$hourKey}:status", $timestamp, "{$endpoint}:{$statusCode}");

        // Set expiry for automatic cleanup
        $this->redis->expire("{$hourKey}:latency", 86400 * 7);
        $this->redis->expire("{$hourKey}:status", 86400 * 7);
    }

    private function storeToDatabase(
        string $endpoint,
        string $method,
        int $statusCode,
        float $duration,
        ?string $userId
    ): void {
        global $db;

        $db->insert('mod_yourmodule_api_metrics', [
            'endpoint' => $endpoint,
            'method' => $method,
            'status_code' => $statusCode,
            'duration_ms' => round($duration * 1000, 2),
            'user_id' => $userId,
            'ip_address' => $_SERVER['REMOTE_ADDR'] ?? '',
            'created_at' => date('Y-m-d H:i:s')
        ]);
    }

    public function recordError(string $endpoint, string $errorType, string $errorMessage): void
    {
        $dateKey = date('Y-m-d');

        $this->incrementCounter("errors:total:{$dateKey}");
        $this->incrementCounter("errors:{$errorType}:{$dateKey}");

        global $db;

        $db->insert('mod_yourmodule_api_errors', [
            'endpoint' => $endpoint,
            'error_type' => $errorType,
            'error_message' => substr($errorMessage, 0, 500),
            'ip_address' => $_SERVER['REMOTE_ADDR'] ?? '',
            'created_at' => date('Y-m-d H:i:s')
        ]);
    }

    public function getLatency(): float
    {
        return microtime(true) - $this->requestStartTime;
    }

    public function getRequestStartTime(): float
    {
        return $this->requestStartTime;
    }
}
```

### 2. Health Checker
```php
<?php
// includes/monitoring/HealthChecker.php

namespace WHMCS\Module\YourModule\Monitoring;

class HealthChecker
{
    private array $checks = [];

    public function __construct()
    {
        $this->registerDefaultChecks();
    }

    private function registerDefaultChecks(): void
    {
        $this->checks = [
            'database' => [$this, 'checkDatabase'],
            'redis' => [$this, 'checkRedis'],
            'disk_space' => [$this, 'checkDiskSpace'],
            'whmcs_license' => [$this, 'checkLicense'],
            'api_modules' => [$this, 'checkModules']
        ];
    }

    public function registerCheck(string $name, callable $check): void
    {
        $this->checks[$name] = $check;
    }

    public function check(): array
    {
        $results = [
            'status' => 'healthy',
            'timestamp' => date('c'),
            'checks' => []
        ];

        foreach ($this->checks as $name => $check) {
            $results['checks'][$name] = $this->runCheck($check);
        }

        // Determine overall status
        foreach ($results['checks'] as $check) {
            if ($check['status'] === 'unhealthy') {
                $results['status'] = 'unhealthy';
                break;
            }
            if ($check['status'] === 'degraded') {
                $results['status'] = 'degraded';
            }
        }

        return $results;
    }

    private function runCheck(callable $check): array
    {
        $startTime = microtime(true);

        try {
            $result = $check();
            $duration = microtime(true) - $startTime;

            return [
                'status' => $result['status'] ?? 'healthy',
                'message' => $result['message'] ?? 'OK',
                'duration_ms' => round($duration * 1000, 2),
                'data' => $result['data'] ?? null
            ];
        } catch (\Exception $e) {
            return [
                'status' => 'unhealthy',
                'message' => 'Exception: ' . $e->getMessage(),
                'duration_ms' => round((microtime(true) - $startTime) * 1000, 2),
                'error' => true
            ];
        }
    }

    private function checkDatabase(): array
    {
        try {
            global $db;

            $startTime = microtime(true);
            $db->query('SELECT 1');
            $duration = microtime(true) - $startTime;

            // Check connection
            $db->query('SHOW STATUS LIKE "Threads_connected"');
            $result = $db->fetch();

            return [
                'status' => 'healthy',
                'message' => 'Database connection OK',
                'data' => [
                    'connections' => (int) ($result['Value'] ?? 0)
                ]
            ];
        } catch (\Exception $e) {
            return [
                'status' => 'unhealthy',
                'message' => 'Database connection failed: ' . $e->getMessage()
            ];
        }
    }

    private function checkRedis(): array
    {
        try {
            $redis = new \Redis();
            $redis->connect('127.0.0.1', 6379, 2);
            $redis->ping();

            return [
                'status' => 'healthy',
                'message' => 'Redis connection OK'
            ];
        } catch (\Exception $e) {
            return [
                'status' => 'degraded',
                'message' => 'Redis unavailable: ' . $e->getMessage()
            ];
        }
    }

    private function checkDiskSpace(): array
    {
        $bytes = disk_free_space('/');
        $total = disk_total_space('/');
        $percentFree = ($bytes / $total) * 100;

        $status = $percentFree > 20 ? 'healthy' : ($percentFree > 10 ? 'degraded' : 'unhealthy');

        return [
            'status' => $status,
            'message' => "Free space: " . round($bytes / 1024 / 1024 / 1024, 2) . " GB",
            'data' => [
                'free_bytes' => $bytes,
                'total_bytes' => $total,
                'percent_free' => round($percentFree, 2)
            ]
        ];
    }

    private function checkLicense(): array
    {
        return [
            'status' => 'healthy',
            'message' => 'License check OK'
        ];
    }

    private function checkModules(): array
    {
        $requiredFiles = [
            dirname(__DIR__) . '/includes/api/ApiResponse.php',
            dirname(__DIR__) . '/includes/auth/ApiKeyAuthenticator.php'
        ];

        $missing = [];
        foreach ($requiredFiles as $file) {
            if (!file_exists($file)) {
                $missing[] = basename($file);
            }
        }

        return [
            'status' => empty($missing) ? 'healthy' : 'unhealthy',
            'message' => empty($missing) ? 'All modules present' : 'Missing: ' . implode(', ', $missing)
        ];
    }
}
```

### 3. API Metrics Middleware
```php
<?php
// includes/monitoring/MetricsMiddleware.php

namespace WHMCS\Module\YourModule\Monitoring;

class MetricsMiddleware
{
    private MetricsCollector $collector;
    private HealthChecker $healthChecker;

    public function __construct()
    {
        $this->collector = new MetricsCollector();
        $this->healthChecker = new HealthChecker();
    }

    public function startMonitoring(): void
    {
        // Already started in constructor
    }

    public function endMonitoring(string $endpoint, string $method, int $statusCode, ?string $userId = null): void
    {
        $duration = $this->collector->getLatency();

        $this->collector->recordRequest(
            $endpoint,
            $method,
            $statusCode,
            $duration,
            $userId
        );

        if ($statusCode >= 400) {
            $errorType = $this->getErrorType($statusCode);
            $this->collector->recordError($endpoint, $errorType, "HTTP {$statusCode}");
        }
    }

    private function getErrorType(int $statusCode): string
    {
        return match (true) {
            $statusCode >= 500 => 'server_error',
            $statusCode >= 400 => 'client_error',
            default => 'unknown'
        };
    }

    public function handleException(\Throwable $e): void
    {
        $endpoint = $_SERVER['REQUEST_URI'] ?? 'unknown';
        $method = $_SERVER['REQUEST_METHOD'] ?? 'unknown';

        $this->collector->recordError(
            $endpoint,
            get_class($e),
            $e->getMessage()
        );

        logActivity('API Exception: ' . $e->getMessage() . ' in ' . $e->getFile() . ':' . $e->getLine());
    }

    public function getMetrics(array $options = []): array
    {
        $period = $options['period'] ?? 'today';
        $limit = $options['limit'] ?? 100;

        return [
            'requests' => $this->getRequestMetrics($period, $limit),
            'errors' => $this->getErrorMetrics($period, $limit),
            'latency' => $this->getLatencyMetrics($period)
        ];
    }

    private function getRequestMetrics(string $period, int $limit): array
    {
        global $db;
        $dateFilter = $this->getDateFilter($period);

        return $db->select(
            "SELECT endpoint, method, status_code, COUNT(*) as count,
                    AVG(duration_ms) as avg_duration
             FROM mod_yourmodule_api_metrics
             WHERE created_at >= ?
             GROUP BY endpoint, method, status_code
             ORDER BY count DESC
             LIMIT ?",
            [$dateFilter, $limit]
        );
    }

    private function getErrorMetrics(string $period, int $limit): array
    {
        global $db;
        $dateFilter = $this->getDateFilter($period);

        return $db->select(
            "SELECT endpoint, error_type, COUNT(*) as count
             FROM mod_yourmodule_api_errors
             WHERE created_at >= ?
             GROUP BY endpoint, error_type
             ORDER BY count DESC
             LIMIT ?",
            [$dateFilter, $limit]
        );
    }

    private function getLatencyMetrics(string $period): array
    {
        global $db;
        $dateFilter = $this->getDateFilter($period);

        $result = $db->select(
            "SELECT AVG(duration_ms) as avg_latency, MAX(duration_ms) as max_latency
             FROM mod_yourmodule_api_metrics
             WHERE created_at >= ?",
            [$dateFilter]
        );

        return $result[0] ?? [];
    }

    private function getDateFilter(string $period): string
    {
        return match ($period) {
            'today' => date('Y-m-d 00:00:00'),
            'yesterday' => date('Y-m-d 00:00:00', strtotime('-1 day')),
            'week' => date('Y-m-d 00:00:00', strtotime('-7 days')),
            'month' => date('Y-m-d 00:00:00', strtotime('-30 days')),
            default => date('Y-m-d 00:00:00')
        };
    }

    public function health(): array
    {
        return $this->healthChecker->check();
    }
}
```

### 4. Database Tables
```sql
CREATE TABLE IF NOT EXISTS mod_yourmodule_api_metrics (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    endpoint VARCHAR(255) NOT NULL,
    method VARCHAR(10) NOT NULL,
    status_code INT NOT NULL,
    duration_ms DECIMAL(10, 2) NOT NULL,
    user_id INT,
    ip_address VARCHAR(45),
    created_at DATETIME NOT NULL,
    INDEX idx_endpoint (endpoint),
    INDEX idx_created_at (created_at),
    INDEX idx_status_code (status_code)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;

CREATE TABLE IF NOT EXISTS mod_yourmodule_api_errors (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    endpoint VARCHAR(255) NOT NULL,
    error_type VARCHAR(100) NOT NULL,
    error_message VARCHAR(500),
    ip_address VARCHAR(45),
    created_at DATETIME NOT NULL,
    INDEX idx_endpoint (endpoint),
    INDEX idx_error_type (error_type),
    INDEX idx_created_at (created_at)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
```

## Common Pitfalls and Solutions

| Pitfall | Solution |
|---------|----------|
| Metrics impact performance | Use async logging or batch inserts |
| Disk space issues | Implement automatic cleanup of old metrics |
| Missing context in errors | Include request ID in all error logs |
| Incomplete latency data | Use microtime(true) for accurate timing |
| Metrics overload | Implement sampling for high-traffic endpoints |

## Security Considerations

1. **Restrict metrics access** - Protect /metrics endpoint with authentication
2. **Sanitize error messages** - Don't log sensitive data
3. **Aggregate sensitive data** - Don't store individual requests for audit
4. **Rate limit metrics queries** - Prevent DoS via metrics endpoint
5. **Encrypt stored metrics** - Consider encryption for sensitive logs

## Testing Checklist

- [ ] Test metrics collection for successful requests
- [ ] Test metrics collection for failed requests
- [ ] Test error recording
- [ ] Test health check with all subsystems
- [ ] Test health check with failed subsystems
- [ ] Test metrics retrieval with date filters
- [ ] Test latency percentile calculations
- [ ] Test automatic cleanup of old metrics
- [ ] Test middleware integration
- [ ] Test performance impact

## Reference Links

- [Prometheus Metrics Format](https://prometheus.io/docs/concepts/data_model/)
- [Health Check Response Format](https://tools.ietf.org/html/draft-inadarei-api-health-check-05)
- [Four Golden Signals - Monitoring](https://sre.google/solutions-book/effective-monitoring/)
