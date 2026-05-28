# WHMCS Monitoring and Alerting Workflow

## Purpose

Implement comprehensive monitoring and alerting for WHMCS to detect issues early, ensure system health, and maintain service quality. This workflow covers metrics collection, alerting rules, notification channels, and escalation procedures.

## Prerequisites

- WHMCS installation (version 8.x)
- Monitoring system (Prometheus, Zabbix, CloudWatch, Datadog)
- Alerting platform (PagerDuty, OpsGenie, Slack webhook)
- Access to server metrics
- Database monitoring capability

## Workflow Steps

### Step 1: Define Monitoring Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                   Monitoring Architecture                   │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  ┌─────────┐  ┌─────────┐  ┌─────────┐  ┌─────────┐         │
│  │  Web    │  │  PHP    │  │  MySQL  │  │   App   │         │
│  │ Server  │  │   FPM   │  │   DB    │  │  Layer  │         │
│  └────┬────┘  └────┬────┘  └────┬────┘  └────┬────┘         │
│       │            │            │            │              │
│       └────────────┴────────────┴────────────┘              │
│                          │                                  │
│              ┌───────────▼───────────┐                     │
│              │   Metrics Collector   │                     │
│              │     (Prometheus)      │                     │
│              └───────────┬───────────┘                     │
│                          │                                  │
│              ┌───────────▼───────────┐                     │
│              │     Alert Manager     │                     │
│              │      (AlertManager)   │                     │
│              └───────────┬───────────┘                     │
│                          │                                  │
│              ┌───────────▼───────────┐                     │
│              │    Notification       │                     │
│              │   Channels            │                     │
│              └───────────────────────┘                     │
└─────────────────────────────────────────────────────────────┘
```

### Step 2: Prometheus Configuration

```yaml
# /etc/prometheus/prometheus.yml

global:
  scrape_interval: 15s
  evaluation_interval: 15s

alerting:
  alertmanagers:
    - static_configs:
        - targets:
          - localhost:9093

rule_files:
  - "/etc/prometheus/alert_rules.yml"

scrape_configs:
  # WHMCS application metrics
  - job_name: 'whmcs'
    metrics_path: '/whmcs/includes/metrics.php'
    static_configs:
      - targets: ['localhost']
        labels:
          environment: 'production'
          service: 'whmcs'

  # Nginx metrics
  - job_name: 'nginx'
    static_configs:
      - targets: ['localhost:9113']
        labels:
          service: 'whmcs-web'

  # PHP-FPM metrics
  - job_name: 'php-fpm'
    static_configs:
      - targets: ['localhost:9254']
        labels:
          service: 'whmcs-php'

  # MySQL metrics
  - job_name: 'mysql'
    static_configs:
      - targets: ['localhost:9104']
        labels:
          service: 'whmcs-database'
```

### Step 3: Alert Rules Configuration

```yaml
# /etc/prometheus/alert_rules.yml

groups:
  - name: whmcs_alerts
    rules:
      # WHMCS Application Down
      - alert: WHMCSDown
        expr: up{job="whmcs"} == 0
        for: 2m
        labels:
          severity: critical
          team: infrastructure
        annotations:
          summary: "WHMCS instance is down"
          description: "WHMCS has been down for more than 2 minutes"

      # High Error Rate
      - alert: WHMCSHighErrorRate
        expr: rate(http_requests_total{status=~"5.."}[5m]) > 0.05
        for: 5m
        labels:
          severity: warning
          team: infrastructure
        annotations:
          summary: "High HTTP 5xx error rate"
          description: "Error rate is {{ $value | printf \"%.2f\" }}% (threshold: 5%)"

      # Slow Response Time
      - alert: WHMCSSlowResponseTime
        expr: histogram_quantile(0.95, http_request_duration_seconds_bucket{job="whmcs"}) > 2
        for: 5m
        labels:
          severity: warning
          team: infrastructure
        annotations:
          summary: "WHMCS response time is slow"
          description: "95th percentile response time is {{ $value | printf \"%.2f\" }}s"

      # Database Connection Pool Exhausted
      - alert: MySQLConnectionPoolExhausted
        expr: mysql_global_status_threads_connected / mysql_global_variable_max_connections > 0.8
        for: 5m
        labels:
          severity: critical
          team: database
        annotations:
          summary: "MySQL connection pool exhausted"
          description: "Database connections at {{ $value | printf \"%.0f\" }}% capacity"

      # PHP-FPM Workers Exhausted
      - alert: PHPFPMWorkersExhausted
        expr: php_fpm_pool_ActiveProcesses / php_fpm_pool_MaxChildren > 0.9
        for: 5m
        labels:
          severity: warning
          team: infrastructure
        annotations:
          summary: "PHP-FPM workers exhausted"
          description: "Active processes at {{ $value | printf \"%.0f\" }}% of max"

      # Disk Space Low
      - alert: DiskSpaceLow
        expr: (node_filesystem_avail_bytes{mountpoint="/"} / node_filesystem_size_bytes) < 0.1
        for: 10m
        labels:
          severity: warning
          team: infrastructure
        annotations:
          summary: "Disk space is low"
          description: "Only {{ $value | printf \"%.1f\" }}% disk space remaining"

      # SSL Certificate Expiring
      - alert: SSLCertExpiring
        expr: ssl_cert_expiry_seconds < 2592000  # 30 days
        for: 1h
        labels:
          severity: warning
          team: security
        annotations:
          summary: "SSL certificate expiring soon"
          description: "SSL certificate expires in {{ $value | printf \"%.0f\" }} seconds"
```

### Step 4: WHMCS Metrics Endpoint

```php
<?php
// /var/www/whmcs/includes/metrics.php
// Prometheus metrics endpoint for WHMCS

declare(strict_types=1);

require_once __DIR__ . '/init.php';

// Check authentication for metrics endpoint
$apiKey = $_SERVER['HTTP_X_API_KEY'] ?? '';
if ($apiKey !== getenv('METRICS_API_KEY')) {
    http_response_code(401);
    exit('Unauthorized');
}

// Prevent caching
header('Content-Type: text/plain; version=0.0.4; charset=utf-8');
header('Cache-Control: no-cache');

// Collect metrics
$metrics = [];

// Application metrics
$metrics[] = '# HELP whmcs_version WHMCS version info';
$metrics[] = '# TYPE whmcs_version gauge';
$metrics[] = 'whmcs_version{' . http_build_query([
    'version' => \App::getVersion(),
    'php_version' => PHP_VERSION
], '', ',') . '} 1';

// Request metrics
$startTime = defined('REQUEST_START_TIME') ? REQUEST_START_TIME : microtime(true);
$requestDuration = microtime(true) - $startTime;
$metrics[] = '# HELP http_request_duration_seconds HTTP request duration';
$metrics[] = '# TYPE http_request_duration_seconds histogram';
$buckets = [0.005, 0.01, 0.025, 0.05, 0.1, 0.25, 0.5, 1, 2.5, 5, 10];
foreach ($buckets as $bucket) {
    $le = $bucket < $requestDuration ? $bucket : $requestDuration;
    $metrics[] = "http_request_duration_seconds_bucket{le=\"$bucket\"} 1";
}
$metrics[] = "http_request_duration_seconds_bucket{le=\"+Inf\"} 1";
$metrics[] = "http_request_duration_seconds_sum $requestDuration";
$metrics[] = "http_request_duration_seconds_count 1";

// Database metrics
try {
    $pdo = Capsule::connection()->getPdo();
    
    // Query performance
    $stmt = $pdo->query("SHOW GLOBAL STATUS LIKE 'Questions'");
    $questions = $stmt->fetch(PDO::FETCH_ASSOC);
    $metrics[] = '# HELP whmcs_db_queries_total Total database queries';
    $metrics[] = '# TYPE whmcs_db_queries_total counter';
    $metrics[] = "whmcs_db_queries_total " . $questions['Value'];
    
    // Slow queries
    $stmt = $pdo->query("SHOW GLOBAL STATUS LIKE 'Slow_queries'");
    $slowQueries = $stmt->fetch(PDO::FETCH_ASSOC);
    $metrics[] = '# HELP whmcs_db_slow_queries_total Slow database queries';
    $metrics[] = '# TYPE whmcs_db_slow_queries_total counter';
    $metrics[] = "whmcs_db_slow_queries_total " . $slowQueries['Value'];
} catch (Exception $e) {
    // Database connection failed
}

// Cache metrics
$cacheStats = Cache::getStats();
$metrics[] = '# HELP whmcs_cache_hits_total Cache hits';
$metrics[] = '# TYPE whmcs_cache_hits_total counter';
$metrics[] = "whmcs_cache_hits_total " . ($cacheStats['hits'] ?? 0);

$metrics[] = '# HELP whmcs_cache_misses_total Cache misses';
$metrics[] = '# TYPE whmcs_cache_misses_total counter';
$metrics[] = "whmcs_cache_misses_total " . ($cacheStats['misses'] ?? 0);

// Session metrics
$sessionDir = ROOTDIR . '/data/sessions';
if (is_dir($sessionDir)) {
    $sessionCount = count(scandir($sessionDir)) - 2; // . and ..
    $metrics[] = '# HELP whmcs_active_sessions Active user sessions';
    $metrics[] = '# TYPE whmcs_active_sessions gauge';
    $metrics[] = "whmcs_active_sessions $sessionCount";
}

echo implode("\n", $metrics) . "\n";
```

### Step 5: Alert Manager Configuration

```yaml
# /etc/alertmanager/alertmanager.yml

global:
  resolve_timeout: 5m

route:
  group_by: ['alertname', 'severity']
  group_wait: 30s
  group_interval: 5m
  repeat_interval: 4h
  receiver: 'default-receiver'
  
  routes:
    # Critical alerts - page immediately
    - match:
        severity: critical
      receiver: 'pagerduty'
      continue: true
    
    # Warning alerts - notify Slack
    - match:
        severity: warning
      receiver: 'slack-warnings'
      continue: true
    
    # Security alerts - dedicated channel
    - match:
        team: security
      receiver: 'security-alerts'

receivers:
  - name: 'default-receiver'
    email_configs:
      - to: 'admin@example.com'
        send_resolved: true
        headers:
          subject: 'WHMCS Alert: {{ .GroupLabels.alertname }}'

  - name: 'pagerduty'
    pagerduty_configs:
      - service_key: 'your-pagerduty-key'
        severity: critical
        description: '{{ .GroupLabels.alertname }}: {{ .Annotations.summary }}'
        details:
          info: '{{ .Annotations.description }}'

  - name: 'slack-warnings'
    slack_configs:
      - api_url: 'https://hooks.slack.com/services/XXX/YYY/ZZZ'
        channel: '#whmcs-alerts'
        title: 'WHMCS Alert'
        text: '{{ .Annotations.summary }}'
        fields:
          - title: 'Description'
            value: '{{ .Annotations.description }}'
            short: false

  - name: 'security-alerts'
    slack_configs:
      - api_url: 'https://hooks.slack.com/services/XXX/YYY/ZZZ'
        channel: '#security-alerts'
        title: 'Security Alert'
        text: '{{ .Annotations.summary }}'
        color: 'danger'

inhibit_rules:
  - source_match:
      severity: 'critical'
    target_match:
      severity: 'warning'
    equal: ['alertname']
```

### Step 6: Monitoring Dashboard (Grafana)

```json
{
  "dashboard": {
    "title": "WHMCS Overview",
    "tags": ["whmcs", "infrastructure"],
    "timezone": "browser",
    "panels": [
      {
        "title": "Response Time (P95)",
        "type": "graph",
        "targets": [
          {
            "expr": "histogram_quantile(0.95, rate(http_request_duration_seconds_bucket[5m]))",
            "legendFormat": "P95 Response Time"
          }
        ],
        "gridPos": {"h": 8, "w": 12, "x": 0, "y": 0}
      },
      {
        "title": "Request Rate",
        "type": "graph",
        "targets": [
          {
            "expr": "rate(http_requests_total[5m])",
            "legendFormat": "{{ status }} - {{ handler }}"
          }
        ],
        "gridPos": {"h": 8, "w": 12, "x": 12, "y": 0}
      },
      {
        "title": "Database Queries",
        "type": "graph",
        "targets": [
          {
            "expr": "whmcs_db_queries_total",
            "legendFormat": "Total Queries"
          },
          {
            "expr": "whmcs_db_slow_queries_total",
            "legendFormat": "Slow Queries"
          }
        ],
        "gridPos": {"h": 8, "w": 12, "x": 0, "y": 8}
      },
      {
        "title": "Cache Hit Rate",
        "type": "gauge",
        "targets": [
          {
            "expr": "whmcs_cache_hits_total / (whmcs_cache_hits_total + whmcs_cache_misses_total) * 100",
            "legendFormat": "Hit Rate %"
          }
        ],
        "gridPos": {"h": 8, "w": 6, "x": 12, "y": 8}
      },
      {
        "title": "Active Sessions",
        "type": "stat",
        "targets": [
          {
            "expr": "whmcs_active_sessions",
            "legendFormat": "Sessions"
          }
        ],
        "gridPos": {"h": 8, "w": 6, "x": 18, "y": 8}
      }
    ]
  }
}
```

### Step 7: Custom Health Check Endpoint

```php
<?php
// /var/www/whmcs/includes/health.php
// Comprehensive health check endpoint

declare(strict_types=1);

require_once __DIR__ . '/init.php';

header('Content-Type: application/json');

$health = [
    'status' => 'healthy',
    'timestamp' => date('c'),
    'version' => \App::getVersion(),
    'checks' => []
];

// Database check
try {
    $pdo = Capsule::connection()->getPdo();
    $pdo->query('SELECT 1');
    $health['checks']['database'] = [
        'status' => 'ok',
        'latency_ms' => measureQueryTime(function() use($pdo) {
            $pdo->query('SELECT 1');
        })
    ];
} catch (Exception $e) {
    $health['checks']['database'] = [
        'status' => 'error',
        'error' => $e->getMessage()
    ];
    $health['status'] = 'unhealthy';
}

// Redis check
try {
    $redis = new Redis();
    $redis->connect(getenv('REDIS_HOST') ?: 'localhost', 6379);
    $pingTime = measureQueryTime(function() use($redis) {
        $redis->ping();
    });
    $health['checks']['redis'] = [
        'status' => 'ok',
        'latency_ms' => $pingTime
    ];
} catch (Exception $e) {
    $health['checks']['redis'] = [
        'status' => 'error',
        'error' => $e->getMessage()
    ];
    $health['status'] = 'degraded';
}

// Disk space check
$diskFree = disk_free_space('/');
$diskTotal = disk_total_space('/');
$diskPercent = (($diskTotal - $diskFree) / $diskTotal) * 100;

$health['checks']['disk'] = [
    'status' => $diskPercent > 90 ? 'error' : 'ok',
    'used_percent' => round($diskPercent, 2),
    'free_gb' => round($diskFree / 1024 / 1024 / 1024, 2)
];

if ($diskPercent > 90) {
    $health['status'] = 'unhealthy';
}

// File permissions check
$configWritable = is_writable(ROOTDIR . '/configuration.php');
$storageWritable = is_writable(ROOTDIR . '/storage');

$health['checks']['permissions'] = [
    'status' => ($configWritable || $storageWritable) ? 'warning' : 'ok',
    'configuration_writable' => $configWritable,
    'storage_writable' => $storageWritable
];

// Cron health check
$cronLastRun = Capsule::table('tblactivitylog')
    ->where('description', 'LIKE', '%Cron%')
    ->orderBy('id', 'DESC')
    ->first();

$health['checks']['cron'] = [
    'status' => 'ok',
    'last_run' => $cronLastRun ? $cronLastRun->date : null
];

// Output status code
http_response_code($health['status'] === 'healthy' ? 200 : 503);
echo json_encode($health, JSON_PRETTY_PRINT);

function measureQueryTime(callable $callback): float {
    $start = microtime(true);
    $callback();
    return round((microtime(true) - $start) * 1000, 2);
}
```

## Alert Severity Levels

| Severity | Response Time | Examples | Notification |
|----------|---------------|----------|--------------|
| Critical | 15 minutes | Site down, DB down | PagerDuty, SMS, Phone |
| High | 30 minutes | Service degraded | PagerDuty, Slack |
| Medium | 2 hours | Performance issues | Slack, Email |
| Low | Next business day | Warnings, info | Email |

## Best Practices

1. **Define SLOs**: Clear service level objectives
2. **Alert on Symptoms**: Not causes (e.g., error rate, not disk space)
3. **Tune Thresholds**: Avoid alert fatigue
4. **Use Labeling**: Organize alerts by team and service
5. **Document Runbooks**: How to resolve each alert type
6. **Review Alerts**: Regular retrospective on alerts

## Common Pitfalls

- **Too Many Alerts**: Alert fatigue causes real issues to be missed
- **No Thresholds**: Alerting on every metric
- **No Escalation**: Critical alerts not reaching right people
- **No Runbooks**: Unclear how to respond
- **No Testing**: Alerts not verified until production

## Verification Checklist

- [ ] All critical services monitored
- [ ] Alert rules tested
- [ ] Notifications reaching correct people
- [ ] Escalation procedures working
- [ ] Runbooks documented for all alerts
- [ ] Dashboard views created
- [ ] On-call rotation configured
- [ ] Alert review process established

## Related Documentation

- [WHMCS Performance Audit](whmcs-performance-audit.md)
- [WHMCS Metrics Collection](whmcs-metrics-collection.md)
- [WHMCS Incident Response](whmcs-incident-response.md)