# WHMCS Metrics Collection and Analysis Workflow

## Purpose

Implement comprehensive metrics collection and analysis for WHMCS to understand system behavior, optimize performance, and make data-driven decisions. This workflow covers metrics sources, collection methods, storage, and analysis techniques.

## Prerequisites

- WHMCS installation (version 8.x)
- Metrics collection system (Prometheus, InfluxDB, Datadog)
- Time-series database for storage
- Visualization tool (Grafana, Kibana)
- Data analysis capabilities

## Workflow Steps

### Step 1: Metrics Sources Identification

```
┌─────────────────────────────────────────────────────────────┐
│                    Metrics Sources                          │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  ┌──────────────────────────────────────────────────────┐  │
│  │                    Infrastructure                     │  │
│  │  • CPU Usage        • Memory Usage                   │  │
│  │  • Disk I/O         • Network Traffic                │  │
│  │  • Disk Space       • Load Average                   │  │
│  └──────────────────────────────────────────────────────┘  │
│                                                             │
│  ┌──────────────────────────────────────────────────────┐  │
│  │                    Application                        │  │
│  │  • Request Count    • Error Rate                     │  │
│  │  • Response Time    • Active Users                   │  │
│  │  • API Calls        • Cache Hit Rate                 │  │
│  └──────────────────────────────────────────────────────┘  │
│                                                             │
│  ┌──────────────────────────────────────────────────────┐  │
│  │                    Database                           │  │
│  │  • Query Count      • Slow Queries                   │  │
│  │  • Connection Pool  • Replication Lag                │  │
│  │  • Table Size       • Index Usage                    │  │
│  └──────────────────────────────────────────────────────┘  │
│                                                             │
│  ┌──────────────────────────────────────────────────────┐  │
│  │                    Business                           │  │
│  │  • New Orders       • Revenue                        │  │
│  │  • Active Clients   • Support Tickets                │  │
│  │  • Churn Rate       • Conversion Rate                │  │
│  └──────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────┘
```

### Step 2: Infrastructure Metrics (Node Exporter)

```bash
# /etc/prometheus/node_exporter.conf
# Node Exporter configuration

collector:
  enabled: true
  textfile:
    directory: /var/lib/node_exporter/textfile_collector

# Disable problematic collectors
disable-defaults: false

# Enable specific collectors
collectors:
  enabled:
    - cpu
    - diskstats
    - filesystem
    - loadavg
    - meminfo
    - netdev
    - netstat
    - stat
    - time
    - uname
    - vmstat

# Custom collector for WHMCS-specific metrics
cat <<'EOF' > /var/lib/node_exporter/textfile_collector/whmcs.prom
# WHMCS specific metrics
whmcs_storage_size_bytes{type="attachments"} $(du -sb /var/www/whmcs/attachments | cut -f1)
whmcs_storage_size_bytes{type="uploads"} $(du -sb /var/www/whmcs/uploads | cut -f1)
whmcs_storage_size_bytes{type="logs"} $(du -sb /var/www/whmcs/storage/logs | cut -f1)
EOF
```

### Step 3: Application Metrics Collection

```php
<?php
// /var/www/whmcs/includes/hooks/metrics_hook.php
// WHMCS application metrics collection

use WHMCS\Database\Capsule;

// Request timing
$requestStart = defined('START_TIME') ? START_TIME : microtime(true);

add_hook('AfterApplicationBootstrap', 1, function($vars) {
    global $metrics;
    $metrics['bootstrap_time'] = microtime(true) - $GLOBALS['START_TIME'];
});

// Database query metrics
add_hook('DatabaseQueryListener', 1, function($vars) {
    $query = $vars['query'];
    $duration = $vars['duration'];
    
    // Track slow queries
    if ($duration > 1.0) {
        logActivity("Slow Query (>1s): " . substr($query, 0, 200));
    }
    
    // Record metrics
    Metrics::recordQuery($query, $duration);
});

// Cache metrics
add_hook('CacheHit', 1, function($vars) {
    Metrics::recordCacheHit($vars['key']);
});

add_hook('CacheMiss', 1, function($vars) {
    Metrics::recordCacheMiss($vars['key']);
});

// API metrics
add_hook('APIFault', 1, function($vars) {
    Metrics::recordApiCall($vars['action'], false, $vars['error'] ?? '');
});

add_hook('APISuccess', 1, function($vars) {
    Metrics::recordApiCall($vars['action'], true);
});

// Session metrics
add_hook('UserLogin', 1, function($vars) {
    Metrics::recordUserLogin($vars['userid']);
});

add_hook('UserLogout', 1, function($vars) {
    Metrics::recordUserLogout($vars['userid']);
});

// Business metrics
add_hook('OrderCreated', 1, function($vars) {
    Metrics::recordOrder($vars['orderid'], $vars['amount']);
});

add_hook('InvoicePaid', 1, function($vars) {
    Metrics::recordPayment($vars['invoiceid'], $vars['amount']);
});

// Metrics class
class Metrics {
    private static $prefix = 'whmcs_metrics_';
    private static $redis;
    
    public static function getRedis() {
        if (!self::$redis) {
            self::$redis = new Redis();
            self::$redis->connect('127.0.0.1', 6379);
        }
        return self::$redis;
    }
    
    public static function recordQuery(string $query, float $duration): void {
        $redis = self::getRedis();
        $bucket = self::getDurationBucket($duration);
        
        $redis->incr(self::$prefix . 'db_query_total');
        $redis->incr(self::$prefix . "db_query_duration_bucket_$bucket");
        
        if ($duration > 1.0) {
            $redis->incr(self::$prefix . 'db_query_slow_total');
        }
    }
    
    public static function recordCacheHit(string $key): void {
        self::getRedis()->incr(self::$prefix . 'cache_hits_total');
    }
    
    public static function recordCacheMiss(string $key): void {
        self::getRedis()->incr(self::$prefix . 'cache_misses_total');
    }
    
    public static function recordApiCall(string $action, bool $success, string $error = ''): void {
        $redis = self::getRedis();
        
        $redis->incr(self::$prefix . "api_calls_total{action=\"$action\"}");
        
        if (!$success) {
            $redis->incr(self::$prefix . "api_errors_total{action=\"$action\",error=\"$error\"}");
        }
    }
    
    public static function recordUserLogin(int $userId): void {
        self::getRedis()->incr(self::$prefix . 'user_logins_total');
        
        // Track active sessions
        $sessions = self::getRedis()->zrange(self::$prefix . 'active_sessions', 0, -1);
        if (count($sessions) > 1000) { // Cleanup old sessions
            self::getRedis()->zremrangebyrank(self::$prefix . 'active_sessions', 0, count($sessions) - 1001);
        }
        
        self::getRedis()->zadd(self::$prefix . 'active_sessions', time(), $userId . ':' . session_id());
    }
    
    public static function recordUserLogout(int $userId): void {
        self::getRedis()->zrem(self::$prefix . 'active_sessions', $userId . ':*');
    }
    
    public static function recordOrder(int $orderId, float $amount): void {
        $redis = self::getRedis();
        
        $redis->incr(self::$prefix . 'orders_total');
        $redis->incrbyfloat(self::$prefix . 'orders_revenue_total', $amount);
        
        // Time series for orders
        $hourKey = date('Y-m-d-H');
        $redis->incr(self::$prefix . "orders_hourly{$hourKey}");
    }
    
    public static function recordPayment(int $invoiceId, float $amount): void {
        $redis = self::getRedis();
        
        $redis->incr(self::$prefix . 'payments_total');
        $redis->incrbyfloat(self::$prefix . 'payments_revenue_total', $amount);
        
        // Daily revenue tracking
        $dayKey = date('Y-m-d');
        $redis->incrbyfloat(self::$prefix . "revenue_daily{$dayKey}", $amount);
    }
    
    private static function getDurationBucket(float $duration): string {
        if ($duration < 0.01) return '0.01';
        if ($duration < 0.05) return '0.05';
        if ($duration < 0.1) return '0.1';
        if ($duration < 0.5) return '0.5';
        if ($duration < 1.0) return '1.0';
        return '5.0+';
    }
}
```

### Step 4: Database Metrics Collection

```sql
-- MySQL metrics queries for monitoring

-- Table sizes and growth
SELECT 
    table_name,
    ROUND(data_length / 1024 / 1024, 2) as data_mb,
    ROUND(index_length / 1024 / 1024, 2) as index_mb,
    ROUND((data_length + index_length) / 1024 / 1024, 2) as total_mb,
    table_rows,
    ROUND((data_length + index_length) / NULLIF(table_rows, 0), 0) as avg_row_bytes
FROM information_schema.tables
WHERE table_schema = 'whmcs_main'
ORDER BY (data_length + index_length) DESC
LIMIT 20;

-- Slow queries
SELECT 
    start_time,
    query_time,
    lock_time,
    rows_sent,
    rows_examined,
    db,
    LEFT(sql_text, 200) as query_preview
FROM mysql.slow_log
WHERE start_time > DATE_SUB(NOW(), INTERVAL 24 HOUR)
ORDER BY query_time DESC
LIMIT 10;

-- Connection status
SELECT 
    threads_connected,
    threads_running,
    threads_cached,
    connections,
    max_connections,
    (threads_connected / max_connections * 100) as connection_usage_pct
FROM (
    SELECT 
        @@threads_connected as threads_connected,
        @@threads_running as threads_running,
        @@threads_cached as threads_cached,
        (SELECT COUNT(*) FROM information_schema.processlist) as connections,
        @@max_connections as max_connections
) as status;

-- Index usage analysis
SELECT 
    table_name,
    index_name,
    cardinality,
    seq_in_index,
    column_name,
    NON_UNIQUE as non_unique
FROM information_schema.statistics
WHERE table_schema = 'whmcs_main'
ORDER BY table_name, index_name, seq_in_index;

-- Replication status
SHOW SLAVE STATUS\G

-- Query performance
SELECT 
    SUM_TIMER_WAIT / 1000000000000000 as total_time_sec,
    COUNT_STAR as execution_count,
    SUM_TIMER_WAIT / COUNT_STAR / 1000000000000 as avg_time_ms,
    SUM_ROWS_EXAMINED as rows_scanned,
    SUM_ROWS_SENT as rows_sent,
    DIGEST_TEXT
FROM performance_schema.events_statements_summary_by_digest
WHERE DIGEST_TEXT IS NOT NULL
ORDER BY SUM_TIMER_WAIT DESC
LIMIT 20;
```

### Step 5: Business Metrics Dashboard

```php
<?php
// /opt/scripts/business_metrics.php
// Business metrics collection for WHMCS

class BusinessMetricsCollector {
    private $pdo;
    private $redis;
    
    public function __construct() {
        $this->pdo = new PDO('mysql:host=localhost', 'whmcs', 'password');
        $this->redis = new Redis();
        $this->redis->connect('127.0.0.1', 6379);
    }
    
    public function collectDaily(): array {
        $metrics = [];
        
        // Client metrics
        $metrics['clients'] = [
            'total' => $this->getTotalClients(),
            'active' => $this->getActiveClients(),
            'new_today' => $this->getNewClientsToday(),
            'churned_30d' => $this->getChurnedClients(30)
        ];
        
        // Order metrics
        $metrics['orders'] = [
            'total_today' => $this->getOrdersToday(),
            'revenue_today' => $this->getRevenueToday(),
            'pending' => $this->getPendingOrders(),
            'avg_order_value' => $this->getAvgOrderValue()
        ];
        
        // Service metrics
        $metrics['services'] = [
            'active' => $this->getActiveServices(),
            'suspended' => $this->getSuspendedServices(),
            'terminated' => $this->getTerminatedServices(),
            'upcoming_renewals' => $this->getUpcomingRenewals()
        ];
        
        // Support metrics
        $metrics['support'] = [
            'open_tickets' => $this->getOpenTickets(),
            'avg_response_time' => $this->getAvgResponseTime(),
            'satisfaction_score' => $this->getSatisfactionScore()
        ];
        
        // Store in Redis with TTL
        $this->storeMetrics($metrics);
        
        return $metrics;
    }
    
    private function getTotalClients(): int {
        return $this->pdo->query("SELECT COUNT(*) FROM tblclients")->fetchColumn();
    }
    
    private function getActiveClients(): int {
        return $this->pdo->query("SELECT COUNT(*) FROM tblclients WHERE status = 'Active'")->fetchColumn();
    }
    
    private function getNewClientsToday(): int {
        return $this->pdo->query("
            SELECT COUNT(*) FROM tblclients 
            WHERE DATE(created_at) = CURDATE()
        ")->fetchColumn();
    }
    
    private function getChurnedClients(int $days): int {
        return $this->pdo->query("
            SELECT COUNT(*) FROM tblclients 
            WHERE status = 'Inactive' 
            AND updated_at > DATE_SUB(CURDATE(), INTERVAL $days DAY)
        ")->fetchColumn();
    }
    
    private function getOrdersToday(): int {
        return $this->pdo->query("
            SELECT COUNT(*) FROM tblorders 
            WHERE DATE(date) = CURDATE()
        ")->fetchColumn();
    }
    
    private function getRevenueToday(): float {
        return $this->pdo->query("
            SELECT COALESCE(SUM(amount), 0) FROM tblinvoices 
            WHERE status = 'Paid' 
            AND DATE(paymentdate) = CURDATE()
        ")->fetchColumn();
    }
    
    private function getPendingOrders(): int {
        return $this->pdo->query("
            SELECT COUNT(*) FROM tblorders 
            WHERE status IN ('Pending', 'Active')
        ")->fetchColumn();
    }
    
    private function getAvgOrderValue(): float {
        return $this->pdo->query("
            SELECT COALESCE(AVG(amount), 0) FROM tblorders 
            WHERE status IN ('Pending', 'Active', 'Completed')
            AND date > DATE_SUB(CURDATE(), INTERVAL 30 DAY)
        ")->fetchColumn();
    }
    
    private function getActiveServices(): int {
        return $this->pdo->query("
            SELECT COUNT(*) FROM tblhosting 
            WHERE domainstatus = 'Active'
        ")->fetchColumn();
    }
    
    private function getSuspendedServices(): int {
        return $this->pdo->query("
            SELECT COUNT(*) FROM tblhosting 
            WHERE domainstatus = 'Suspended'
        ")->fetchColumn();
    }
    
    private function getTerminatedServices(): int {
        return $this->pdo->query("
            SELECT COUNT(*) FROM tblhosting 
            WHERE domainstatus = 'Terminated'
        ")->fetchColumn();
    }
    
    private function getUpcomingRenewals(): int {
        return $this->pdo->query("
            SELECT COUNT(*) FROM tblhosting 
            WHERE nextduedate BETWEEN CURDATE() AND DATE_ADD(CURDATE(), INTERVAL 7 DAY)
            AND domainstatus = 'Active'
        ")->fetchColumn();
    }
    
    private function getOpenTickets(): int {
        return $this->pdo->query("
            SELECT COUNT(*) FROM tbltickets 
            WHERE status NOT IN ('Closed', 'Answered')
        ")->fetchColumn();
    }
    
    private function getAvgResponseTime(): int {
        $result = $this->pdo->query("
            SELECT AVG(TIMESTAMPDIFF(HOUR, created_at, first_reply_at)) as avg_hours
            FROM tbltickets 
            WHERE first_reply_at IS NOT NULL
            AND created_at > DATE_SUB(CURDATE(), INTERVAL 7 DAY)
        ")->fetchColumn();
        
        return (int) ($result ?? 0);
    }
    
    private function getSatisfactionScore(): float {
        $result = $this->pdo->query("
            SELECT AVG(rating) 
            FROM tblticketfeedback 
            WHERE rating IS NOT NULL
            AND created_at > DATE_SUB(CURDATE(), INTERVAL 30 DAY)
        ")->fetchColumn();
        
        return round((float) ($result ?? 0), 2);
    }
    
    private function storeMetrics(array $metrics): void {
        $key = 'whmcs_metrics_' . date('Y-m-d-H');
        $this->redis->setex($key, 86400 * 30, json_encode($metrics)); // 30 day TTL
    }
}

// Run collection
$collector = new BusinessMetricsCollector();
$metrics = $collector->collectDaily();
print_r($metrics);
```

### Step 6: Metrics Analysis and Reporting

```python
#!/usr/bin/env python3
# /opt/scripts/metrics_analysis.py
# Analytics script for WHMCS metrics

import json
from datetime import datetime, timedelta
from typing import Dict, List

class MetricsAnalyzer:
    def __init__(self):
        self.data = self.load_metrics()
    
    def analyze_performance_trends(self) -> Dict:
        """Analyze performance trends over time"""
        
        # Response time trend
        response_trend = self.analyze_trend('response_time')
        
        # Error rate trend
        error_trend = self.analyze_trend('error_rate')
        
        # Recommendations
        recommendations = []
        if response_trend['direction'] == 'increasing':
            recommendations.append({
                'severity': 'warning',
                'metric': 'response_time',
                'message': 'Response time increasing. Consider scaling or optimization.',
                'suggestion': 'Review slow queries, optimize caching, scale infrastructure'
            })
        
        return {
            'response_time_trend': response_trend,
            'error_rate_trend': error_trend,
            'recommendations': recommendations
        }
    
    def analyze_capacity(self) -> Dict:
        """Analyze capacity utilization"""
        
        current_usage = self.get_current_usage()
        capacity_limits = self.get_capacity_limits()
        
        utilization = {}
        for resource, usage in current_usage.items():
            limit = capacity_limits.get(resource, 100)
            utilization[resource] = {
                'usage_percent': (usage / limit) * 100,
                'remaining_percent': ((limit - usage) / limit) * 100,
                'days_until_full': self.estimate_days_until_full(resource)
            }
        
        # Alerts
        alerts = []
        for resource, data in utilization.items():
            if data['usage_percent'] > 80:
                alerts.append({
                    'severity': 'critical' if data['usage_percent'] > 90 else 'warning',
                    'resource': resource,
                    'usage': f"{data['usage_percent']:.1f}%",
                    'message': f"{resource} utilization is high"
                })
        
        return {
            'utilization': utilization,
            'alerts': alerts
        }
    
    def generate_daily_report(self) -> str:
        """Generate daily metrics report"""
        
        report = []
        report.append("=" * 60)
        report.append("WHMCS Daily Metrics Report")
        report.append(f"Generated: {datetime.now().strftime('%Y-%m-%d %H:%M:%S')}")
        report.append("=" * 60)
        report.append("")
        
        # Performance summary
        performance = self.analyze_performance_trends()
        report.append("Performance Summary:")
        report.append(f"  Response Time: {self.get_avg('response_time'):.2f}ms")
        report.append(f"  Error Rate: {self.get_avg('error_rate') * 100:.2f}%")
        report.append(f"  Request Rate: {self.get_avg('requests_per_sec'):.1f}/s")
        report.append("")
        
        # Business summary
        business = self.get_business_metrics()
        report.append("Business Summary:")
        report.append(f"  Active Clients: {business['active_clients']}")
        report.append(f"  Orders Today: {business['orders_today']}")
        report.append(f"  Revenue Today: ${business['revenue_today']:.2f}")
        report.append("")
        
        # Capacity summary
        capacity = self.analyze_capacity()
        report.append("Capacity Alerts:")
        for alert in capacity['alerts']:
            report.append(f"  [{alert['severity'].upper()}] {alert['message']}")
        
        report.append("")
        report.append("=" * 60)
        
        return "\n".join(report)
    
    def get_avg(self, metric: str) -> float:
        """Calculate average for a metric"""
        values = self.data.get(metric, [])
        return sum(values) / len(values) if values else 0.0
    
    def analyze_trend(self, metric: str) -> Dict:
        """Analyze trend direction"""
        values = self.data.get(metric, [])
        if len(values) < 2:
            return {'direction': 'stable', 'change_percent': 0}
        
        recent = sum(values[-7:]) / min(7, len(values))
        older = sum(values[-14:-7]) / 7 if len(values) >= 14 else recent
        
        change = ((recent - older) / older * 100) if older else 0
        
        return {
            'direction': 'increasing' if change > 5 else 'decreasing' if change < -5 else 'stable',
            'change_percent': change
        }

# Run analysis
analyzer = MetricsAnalyzer()
print(analyzer.generate_daily_report())
```

## Key Metrics to Track

| Category | Metric | Target | Alert Threshold |
|----------|--------|--------|-----------------|
| Performance | Response Time (P95) | < 500ms | > 2s |
| Performance | Error Rate | < 0.1% | > 1% |
| Performance | Cache Hit Rate | > 90% | < 80% |
| Database | Query Time (avg) | < 50ms | > 200ms |
| Database | Slow Queries | < 10/hour | > 50/hour |
| Capacity | CPU Usage | < 70% | > 90% |
| Capacity | Memory Usage | < 80% | > 95% |
| Capacity | Disk Usage | < 80% | > 90% |
| Business | Orders/Hour | Baseline | < 50% baseline |
| Business | Revenue/Hour | Baseline | < 50% baseline |

## Best Practices

1. **Instrument Everything**: Collect all meaningful metrics
2. **Set Baselines**: Know what's normal for your system
3. **Correlate Data**: Connect application and business metrics
4. **Automate Analysis**: Regular automated reports
5. **Alert on Trends**: Not just thresholds
6. **Retention Policy**: Keep historical data for analysis

## Common Pitfalls

- **Too Much Data**: Collect everything without analysis plan
- **Missing Context**: Metrics without baseline comparison
- **No Business Metrics**: Only technical metrics
- **No Retention Plan**: Data deleted too soon
- **Analysis Paralysis**: Collecting but not acting on data

## Verification Checklist

- [ ] All infrastructure metrics collected
- [ ] Application metrics instrumented
- [ ] Database metrics tracked
- [ ] Business metrics dashboard created
- [ ] Automated reports configured
- [ ] Trend analysis running
- [ ] Alert thresholds set appropriately
- [ ] Data retention policy defined

## Related Documentation

- [WHMCS Monitoring and Alerts](whmcs-monitoring-alerts.md)
- [WHMCS Performance Audit](whmcs-performance-audit.md)
- [WHMCS Cost Optimization](whmcs-cost-optimization.md)