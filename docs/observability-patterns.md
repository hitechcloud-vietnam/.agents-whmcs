# Observability Best Practices

Observability enables understanding system behavior through external outputs. This guide covers implementing comprehensive observability in WHMCS modules.

## Metrics Collection

### Metrics Registry

```php
<?php
/**
 * Metrics registry for collecting application metrics
 */
class MetricsRegistry
{
    private static ?MetricsRegistry $instance = null;
    private array $counters = [];
    private array $gauges = [];
    private array $histograms = [];
    private array $timers = [];
    private array $tags = [];

    private function __construct()
    {
    }

    public static function getInstance(): self
    {
        if (self::$instance === null) {
            self::$instance = new self();
        }

        return self::$instance;
    }

    /**
     * Get or create a counter
     */
    public function counter(string $name, array $tags = []): Counter
    {
        $key = $this->buildKey($name, $tags);

        if (!isset($this->counters[$key])) {
            $this->counters[$key] = new Counter($name, $tags);
        }

        return $this->counters[$key];
    }

    /**
     * Get or create a gauge
     */
    public function gauge(string $name, callable $callback, array $tags = []): Gauge
    {
        $key = $this->buildKey($name, $tags);

        if (!isset($this->gauges[$key])) {
            $this->gauges[$key] = new Gauge($name, $callback, $tags);
        }

        return $this->gauges[$key];
    }

    /**
     * Get or create a histogram
     */
    public function histogram(string $name, array $buckets = null, array $tags = []): Histogram
    {
        $key = $this->buildKey($name, $tags);

        if (!isset($this->histograms[$key])) {
            $this->histograms[$key] = new Histogram($name, $buckets ?? Histogram::DEFAULT_BUCKETS, $tags);
        }

        return $this->histograms[$key];
    }

    /**
     * Get or create a timer
     */
    public function timer(string $name, array $tags = []): Timer
    {
        $key = $this->buildKey($name, $tags);

        if (!isset($this->timers[$key])) {
            $this->timers[$key] = new Timer($name, $tags);
        }

        return $this->timers[$key];
    }

    /**
     * Set global tags
     */
    public function setTags(array $tags): void
    {
        $this->tags = array_merge($this->tags, $tags);
    }

    /**
     * Get all metrics in Prometheus format
     */
    public function export(): string
    {
        $output = [];

        // Export counters
        foreach ($this->counters as $counter) {
            $output[] = $counter->export();
        }

        // Export gauges
        foreach ($this->gauges as $gauge) {
            $output[] = $gauge->export();
        }

        // Export histograms
        foreach ($this->histograms as $histogram) {
            $output = array_merge($output, $histogram->export());
        }

        return implode("\n", $output);
    }

    private function buildKey(string $name, array $tags): string
    {
        ksort($tags);
        return $name . ':' . json_encode($tags);
    }
}
```

### Counter Metric

```php
<?php
/**
 * Counter metric - monotonically increasing value
 */
class Counter
{
    private string $name;
    private array $tags;
    private float $value = 0;

    public function __construct(string $name, array $tags = [])
    {
        $this->name = $name;
        $this->tags = $tags;
    }

    public function increment(float $value = 1): void
    {
        $this->value += $value;
    }

    public function getValue(): float
    {
        return $this->value;
    }

    public function reset(): void
    {
        $this->value = 0;
    }

    public function export(): string
    {
        $tagsStr = $this->formatTags();

        return "# TYPE {$this->name} counter\n" .
               "# HELP {$this->name} Counter metric\n" .
               "{$this->name}{$tagsStr} {$this->value}";
    }

    private function formatTags(): string
    {
        if (empty($this->tags)) {
            return '';
        }

        $parts = [];
        foreach ($this->tags as $key => $value) {
            $parts[] = "{$key}=\"{$value}\"";
        }

        return '{' . implode(',', $parts) . '}';
    }
}
```

### Histogram Metric

```php
<?php
<?php
/**
 * Histogram metric - distribution of values
 */
class Histogram
{
    public const DEFAULT_BUCKETS = [0.005, 0.01, 0.025, 0.05, 0.1, 0.25, 0.5, 1, 2.5, 5, 10];

    private string $name;
    private array $buckets;
    private array $tags;
    private array $counts = [];
    private float $sum = 0;
    private int $totalCount = 0;

    public function __construct(string $name, array $buckets, array $tags = [])
    {
        $this->name = $name;
        $this->buckets = $buckets;
        $this->tags = $tags;

        foreach ($buckets as $bucket) {
            $this->counts[$bucket] = 0;
        }
    }

    public function observe(float $value): void
    {
        $this->sum += $value;
        $this->totalCount++;

        foreach ($this->buckets as $bucket) {
            if ($value <= $bucket) {
                $this->counts[$bucket]++;
            }
        }
    }

    public function export(): array
    {
        $output = [];
        $tagsStr = $this->formatTags();
        $cumulativeCount = 0;

        // Bucket counts
        foreach ($this->buckets as $bucket) {
            $cumulativeCount += $this->counts[$bucket];
            $bucketTags = $tagsStr ? rtrim($tagsStr, '}') . ",le=\"{$bucket}\"}" : "{le=\"{$bucket}\"}";

            $output[] = "# TYPE {$this->name}_bucket gauge\n" .
                        "{$this->name}_bucket{$bucketTags} {$cumulativeCount}";
        }

        // +Inf bucket
        $infTags = $tagsStr ? rtrim($tagsStr, '}') . ",le=\"+Inf\"}" : "{le=\"+Inf\"}";
        $output[] = "{$this->name}_bucket{$infTags} {$this->totalCount}";

        // Sum and count
        $output[] = "# TYPE {$this->name}_sum gauge\n{$this->name}_sum{$tagsStr} {$this->sum}";
        $output[] = "# TYPE {$this->name}_count counter\n{$this->name}_count{$tagsStr} {$this->totalCount}";

        return $output;
    }

    private function formatTags(): string
    {
        if (empty($this->tags)) {
            return '';
        }

        $parts = [];
        foreach ($this->tags as $key => $value) {
            $parts[] = "{$key}=\"{$value}\"";
        }

        return '{' . implode(',', $parts) . '}';
    }
}
```

### Timer Metric

```php
<?php
/**
 * Timer metric - measures duration
 */
class Timer
{
    private string $name;
    private array $tags;
    private Histogram $histogram;

    public function __construct(string $name, array $tags = [])
    {
        $this->name = $name;
        $this->tags = $tags;
        $this->histogram = new Histogram($name . '_seconds', [
            0.001, 0.005, 0.01, 0.025, 0.05, 0.1, 0.25, 0.5, 1, 2.5, 5, 10
        ], $tags);
    }

    /**
     * Time a callable
     */
    public function time(callable $callback)
    {
        $start = microtime(true);

        try {
            return $callback();
        } finally {
            $duration = microtime(true) - $start;
            $this->histogram->observe($duration);
        }
    }

    /**
     * Start timing
     */
    public function start(): Stopwatch
    {
        return new Stopwatch($this->histogram);
    }

    public function observe(float $seconds): void
    {
        $this->histogram->observe($seconds);
    }
}

/**
 * Stopwatch for manual timing
 */
class Stopwatch
{
    private Histogram $histogram;
    private float $startTime;

    public function __construct(Histogram $histogram)
    {
        $this->histogram = $histogram;
        $this->startTime = microtime(true);
    }

    public function stop(): float
    {
        $duration = microtime(true) - $this->startTime;
        $this->histogram->observe($duration);
        return $duration;
    }
}
```

## Logging with Structure

### Structured Logger

```php
<?php
/**
 * Structured logger for observability
 */
class StructuredLogger
{
    private string $channel;
    private array $defaultContext = [];
    private array $handlers = [];
    private static array $channels = [];

    public function __construct(string $channel)
    {
        $this->channel = $channel;
    }

    public static function channel(string $name): self
    {
        if (!isset(self::$channels[$name])) {
            self::$channels[$name] = new self($name);
        }

        return self::$channels[$name];
    }

    public function withContext(array $context): self
    {
        $clone = clone $this;
        $clone->defaultContext = array_merge($this->defaultContext, $context);
        return $clone;
    }

    public function addHandler(LogHandlerInterface $handler): void
    {
        $this->handlers[] = $handler;
    }

    public function debug(string $message, array $context = []): void
    {
        $this->log('DEBUG', $message, $context);
    }

    public function info(string $message, array $context = []): void
    {
        $this->log('INFO', $message, $context);
    }

    public function warning(string $message, array $context = []): void
    {
        $this->log('WARNING', $message, $context);
    }

    public function error(string $message, array $context = []): void
    {
        $this->log('ERROR', $message, $context);
    }

    public function critical(string $message, array $context = []): void
    {
        $this->log('CRITICAL', $message, $context);
    }

    private function log(string $level, string $message, array $context = []): void
    {
        $record = [
            'timestamp' => date('Y-m-d\TH:i:s.uP'),
            'channel' => $this->channel,
            'level' => $level,
            'message' => $message,
            'context' => array_merge($this->defaultContext, $context),
        ];

        foreach ($this->handlers as $handler) {
            try {
                $handler->handle($record);
            } catch (\Throwable $e) {
                // Log handler error - don't fail silently
                error_log("Log handler error: " . $e->getMessage());
            }
        }
    }
}

/**
 * Log handler interface
 */
interface LogHandlerInterface
{
    public function handle(array $record): void;
}
```

### JSON File Handler

```php
<?php
/**
 * JSON file log handler
 */
class JsonFileHandler implements LogHandlerInterface
{
    private string $path;
    private $fileHandle;

    public function __construct(string $path)
    {
        $this->path = $path;
    }

    public function handle(array $record): void
    {
        if ($this->fileHandle === null) {
            $this->fileHandle = fopen($this->path, 'a');
        }

        fwrite($this->fileHandle, json_encode($record) . "\n");
    }

    public function __destruct()
    {
        if ($this->fileHandle !== null) {
            fclose($this->fileHandle);
        }
    }
}

/**
 * Syslog handler
 */
class SyslogHandler implements LogHandlerInterface
{
    private string $ident;
    private array $facilities = [
        'user' => LOG_USER,
        'local0' => LOG_LOCAL0,
        'local1' => LOG_LOCAL1,
        'local2' => LOG_LOCAL2,
        'local3' => LOG_LOCAL3,
        'local4' => LOG_LOCAL4,
        'local5' => LOG_LOCAL5,
        'local6' => LOG_LOCAL6,
        'local7' => LOG_LOCAL7,
    ];

    public function __construct(string $ident = 'whmcs', string $facility = 'user')
    {
        $this->ident = $ident;
        $facility = $this->facilities[$facility] ?? LOG_USER;
        openlog($ident, LOG_PID, $facility);
    }

    public function handle(array $record): void
    {
        $levelMap = [
            'DEBUG' => LOG_DEBUG,
            'INFO' => LOG_INFO,
            'WARNING' => LOG_WARNING,
            'ERROR' => LOG_ERR,
            'CRITICAL' => LOG_CRIT,
        ];

        $priority = $levelMap[$record['level']] ?? LOG_INFO;
        $message = json_encode($record);

        syslog($priority, $message);
    }
}
```

## Health Checks

### Health Check Service

```php
<?php
/**
 * Health check service
 */
class HealthCheckService
{
    private array $checks = [];
    private array $statuses = [];

    /**
     * Register a health check
     */
    public function register(string $name, callable $check): void
    {
        $this->checks[$name] = $check;
    }

    /**
     * Run all health checks
     */
    public function check(): array
    {
        $results = [
            'status' => 'healthy',
            'timestamp' => date('Y-m-d\TH:i:sP'),
            'checks' => [],
        ];

        foreach ($this->checks as $name => $check) {
            try {
                $start = microtime(true);
                $result = $check();
                $duration = microtime(true) - $start;

                $status = is_array($result) ? ($result['status'] ?? 'healthy') : 'healthy';
                $message = is_array($result) ? ($result['message'] ?? 'OK') : 'OK';

                $results['checks'][$name] = [
                    'status' => $status,
                    'message' => $message,
                    'duration_ms' => round($duration * 1000, 2),
                ];

                if ($status === 'unhealthy') {
                    $results['status'] = 'unhealthy';
                } elseif ($status === 'degraded' && $results['status'] === 'healthy') {
                    $results['status'] = 'degraded';
                }
            } catch (\Throwable $e) {
                $results['checks'][$name] = [
                    'status' => 'unhealthy',
                    'message' => $e->getMessage(),
                    'duration_ms' => 0,
                ];
                $results['status'] = 'unhealthy';
            }
        }

        return $results;
    }

    /**
     * Get health status for a single check
     */
    public function checkOne(string $name): array
    {
        if (!isset($this->checks[$name])) {
            return ['status' => 'unknown', 'message' => 'Check not found'];
        }

        try {
            $result = $this->checks[$name]();
            return is_array($result) ? $result : ['status' => 'healthy', 'message' => 'OK'];
        } catch (\Throwable $e) {
            return ['status' => 'unhealthy', 'message' => $e->getMessage()];
        }
    }
}
```

### Common Health Checks

```php
<?php
/**
 * Database health check
 */
class DatabaseHealthCheck
{
    public function __invoke(): array
    {
        try {
            $start = microtime(true);
            Capsule::select('SELECT 1');
            $duration = microtime(true) - $start;

            return [
                'status' => 'healthy',
                'message' => 'Database connection OK',
                'duration_ms' => round($duration * 1000, 2),
            ];
        } catch (\Throwable $e) {
            return [
                'status' => 'unhealthy',
                'message' => 'Database connection failed: ' . $e->getMessage(),
            ];
        }
    }
}

/**
 * External service health check
 */
class ExternalServiceHealthCheck
{
    private string $serviceName;
    private string $url;

    public function __construct(string $serviceName, string $url)
    {
        $this->serviceName = $serviceName;
        $this->url = $url;
    }

    public function __invoke(): array
    {
        try {
            $ch = curl_init();
            curl_setopt_array($ch, [
                CURLOPT_URL => $this->url,
                CURLOPT_RETURNTRANSFER => true,
                CURLOPT_TIMEOUT => 5,
                CURLOPT_CONNECTTIMEOUT => 2,
            ]);

            $start = microtime(true);
            $response = curl_exec($ch);
            $duration = microtime(true) - $start;
            $httpCode = curl_getinfo($ch, CURLINFO_HTTP_CODE);
            curl_close($ch);

            if ($httpCode >= 200 && $httpCode < 300) {
                return [
                    'status' => 'healthy',
                    'message' => "{$this->serviceName} is reachable",
                    'duration_ms' => round($duration * 1000, 2),
                ];
            }

            return [
                'status' => 'degraded',
                'message' => "{$this->serviceName} returned HTTP {$httpCode}",
                'duration_ms' => round($duration * 1000, 2),
            ];
        } catch (\Throwable $e) {
            return [
                'status' => 'unhealthy',
                'message' => "{$this->serviceName} unreachable: " . $e->getMessage(),
            ];
        }
    }
}

/**
 * Disk space health check
 */
class DiskSpaceHealthCheck
{
    private int $warningThreshold;
    private int $criticalThreshold;

    public function __construct(int $warningThreshold = 90, int $criticalThreshold = 95)
    {
        $this->warningThreshold = $warningThreshold;
        $this->criticalThreshold = $criticalThreshold;
    }

    public function __invoke(): array
    {
        $path = defined('WHMCS') ? WHMCS::getRootPath() : __DIR__ . '/../..';
        $total = disk_total_space($path);
        $free = disk_free_space($path);
        $used = $total - $free;
        $percentage = ($used / $total) * 100;

        $status = 'healthy';
        $message = "Disk usage: " . round($percentage, 1) . "%";

        if ($percentage >= $this->criticalThreshold) {
            $status = 'unhealthy';
        } elseif ($percentage >= $this->warningThreshold) {
            $status = 'degraded';
        }

        return [
            'status' => $status,
            'message' => $message,
            'used_bytes' => $used,
            'total_bytes' => $total,
            'percentage' => round($percentage, 1),
        ];
    }
}
```

## Observability Integration

### WHMCS Integration

```php
<?php
/**
 * Initialize observability
 */
add_hook('preAutoload', 1, function ($vars) {
    $metrics = MetricsRegistry::getInstance();

    // Set global tags
    $metrics->setTags([
        'environment' => App::isProduction() ? 'production' : 'development',
        'whmcs_version' => App::getVersion(),
        'module_version' => '1.0.0',
    ]);

    // Setup logging
    $logger = StructuredLogger::channel('whmcs');
    $logger->addHandler(new JsonFileHandler(__DIR__ . '/../../logs/observability.json'));
    $logger->addHandler(new SyslogHandler('whmcs', 'local0'));
});

/**
 * Register health checks
 */
add_hook('AdminAreaPage', 1, function ($vars) {
    $health = App::make(HealthCheckService::class);

    // Database check
    $health->register('database', new DatabaseHealthCheck());

    // External services
    $health->register('payment_gateway', new ExternalServiceHealthCheck(
        'Payment Gateway',
        'https://api.paymentprovider.com/health'
    ));

    // System checks
    $health->register('disk_space', new DiskSpaceHealthCheck());

    // Store for later use
    $_SESSION['health_check_service'] = $health;
});

/**
 * API endpoint for health checks
 */
add_hook('ApiStart', 1, function ($vars) {
    if (($vars['action'] ?? '') === 'HealthCheck') {
        $health = $_SESSION['health_check_service'] ?? new HealthCheckService();
        echo json_encode($health->check());
        exit;
    }
});
```

## Best Practices

1. **Use semantic logging** - Log meaningful events, not just messages
2. **Include correlation IDs** - Link logs to traces and requests
3. **Set up alerting thresholds** - Define SLOs and alert on violations
4. **Sample in production** - Full tracing in staging, sampling in production
5. **Export structured data** - JSON logs for easy parsing
6. **Use histograms for latency** - Percentiles reveal distribution
7. **Health checks must be fast** - Timeout after a few seconds
8. **Document your metrics** - Help teams understand what you're measuring

## Related Patterns

- [Distributed Tracing](./distributed-tracing.md) - Trace propagation
- [Queue Processing](./queue-processing.md) - Async metrics collection
- [Service Layer](./service-layer.md) - Service-level observability
