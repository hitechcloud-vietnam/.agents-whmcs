# WHMCS Performance Monitoring Workflow

## Purpose

Set up comprehensive performance monitoring for WHMCS installations to track response times, identify bottlenecks, optimize database queries, and ensure optimal user experience.

## Prerequisites

- WHMCS installation with admin access
- Access to server monitoring tools
- Database administrator access
- Optional: APM tool (New Relic, Blackfire, etc.)

## Workflow Steps

### Step 1: Performance Baseline Establishment

Define performance metrics and establish baselines:

```php
// includes/performance/baseline.php

class PerformanceBaseline
{
    private $metrics = [];
    private $checkpoints = [];
    
    /**
     * Record a checkpoint with timing
     */
    public function checkpoint(string $name): void
    {
        $this->checkpoints[$name] = microtime(true);
    }
    
    /**
     * Calculate duration between checkpoints
     */
    public function duration(string $start, string $end): float
    {
        if (!isset($this->checkpoints[$start]) || !isset($this->checkpoints[$end])) {
            return 0;
        }
        return ($this->checkpoints[$end] - $this->checkpoints[$start]) * 1000;
    }
    
    /**
     * Record metric value
     */
    public function recordMetric(string $name, $value): void
    {
        $this->metrics[$name] = [
            'value' => $value,
            'timestamp' => date('Y-m-d H:i:s'),
        ];
    }
    
    /**
     * Get baseline thresholds
     */
    public static function getThresholds(): array
    {
        return [
            'page_load_time_ms' => 2000,
            'api_response_ms' => 500,
            'db_query_ms' => 100,
            'template_render_ms' => 500,
            'memory_usage_mb' => 256,
            'db_connections' => 50,
            'cache_hit_ratio' => 0.90,
            'error_rate_percent' => 1,
        ];
    }
    
    /**
     * Check if metrics exceed thresholds
     */
    public function checkThresholds(): array
    {
        $thresholds = self::getThresholds();
        $violations = [];
        
        foreach ($thresholds as $metric => $limit) {
            if (isset($this->metrics[$metric]) && $this->metrics[$metric]['value'] > $limit) {
                $violations[] = [
                    'metric' => $metric,
                    'value' => $this->metrics[$metric]['value'],
                    'limit' => $limit,
                ];
            }
        }
        
        return $violations;
    }
}

// Global instance
$GLOBALS['perf_baseline'] = new PerformanceBaseline();
```

### Step 2: Database Query Monitoring

Monitor and optimize database queries:

```php
// includes/performance/db_monitor.php

class DatabaseQueryMonitor
{
    private static $queries = [];
    private static $enabled = false;
    
    /**
     * Enable query monitoring
     */
    public static function enable(): void
    {
        self::$enabled = true;
        self::$queries = [];
    }
    
    /**
     * Log a database query
     */
    public static function log(string $query, float $time, array $explain = []): void
    {
        if (!self::$enabled) return;
        
        $queryHash = md5($query);
        $backtrace = debug_backtrace(DEBUG_BACKTRACE_IGNORE_ARGS, 5);
        
        self::$queries[] = [
            'sql' => $query,
            'time_ms' => $time,
            'explain' => $explain,
            'hash' => $queryHash,
            'caller' => self::getCallerFile($backtrace),
            'timestamp' => date('Y-m-d H:i:s'),
        ];
        
        // Log slow queries
        if ($time > 100) {
            self::logSlowQuery($query, $time, $explain);
        }
    }
    
    /**
     * Get slow queries report
     */
    public static function getSlowQueries(int $limit = 20): array
    {
        $slow = array_filter(self::$queries, fn($q) => $q['time_ms'] > 100);
        usort($slow, fn($a, $b) => $b['time_ms'] <=> $a['time_ms']);
        
        return array_slice($slow, 0, $limit);
    }
    
    /**
     * Get query statistics
     */
    public static function getStatistics(): array
    {
        $times = array_column(self::$queries, 'time_ms');
        
        return [
            'total_queries' => count(self::$queries),
            'total_time_ms' => array_sum($times),
            'avg_time_ms' => count($times) ? array_sum($times) / count($times) : 0,
            'max_time_ms' => $times ? max($times) : 0,
            'slow_queries' => count(array_filter($times, fn($t) => $t > 100)),
        ];
    }
    
    private static function logSlowQuery(string $query, float $time, array $explain): void
    {
        Capsule::table('perf_slow_queries')->insert([
            'query_hash' => md5($query),
            'query' => substr($query, 0, 1000),
            'execution_time_ms' => $time,
            'explain_data' => json_encode($explain),
            'created_at' => date('Y-m-d H:i:s'),
        ]);
    }
    
    private static function getCallerFile(array $backtrace): string
    {
        foreach ($backtrace as $frame) {
            if (isset($frame['file']) && strpos($frame['file'], 'whmcs') !== false) {
                return basename($frame['file']) . ':' . ($frame['line'] ?? 0);
            }
        }
        return 'unknown';
    }
}

/**
 * Hook to capture database queries
 */
add_hook('AfterDatabaseConnect', 1, function($vars) {
    DatabaseQueryMonitor::enable();
});
```

### Step 3: Caching Strategy Implementation

Implement comprehensive caching:

```php
// includes/performance/cache_manager.php

class PerformanceCacheManager
{
    private $driver;
    private $prefix = 'whmcs_';
    
    public function __construct(string $driver = 'redis')
    {
        $this->driver = $driver;
    }
    
    /**
     * Get cached value
     */
    public function get(string $key, $default = null)
    {
        $cache = \WHMCS\ApplicationSupport\Helper\Cache\CacheManager::getInstance();
        
        try {
            $value = $cache->get($this->prefix . $key);
            return $value ?? $default;
        } catch (Exception $e) {
            return $default;
        }
    }
    
    /**
     * Set cached value with TTL
     */
    public function set(string $key, $value, int $ttl = 3600): bool
    {
        $cache = \WHMCS\ApplicationSupport\Helper\Cache\CacheManager::getInstance();
        
        try {
            $cache->set($this->prefix . $key, $value, $ttl);
            return true;
        } catch (Exception $e) {
            return false;
        }
    }
    
    /**
     * Delete cached value
     */
    public function delete(string $key): bool
    {
        $cache = \WHMCS\ApplicationSupport\Helper\Cache\CacheManager::getInstance();
        
        try {
            $cache->forget($this->prefix . $key);
            return true;
        } catch (Exception $e) {
            return false;
        }
    }
    
    /**
     * Cache with lazy loading
     */
    public function remember(string $key, int $ttl, callable $callback)
    {
        $value = $this->get($key);
        
        if ($value !== null) {
            return $value;
        }
        
        $value = $callback();
        $this->set($key, $value, $ttl);
        
        return $value;
    }
    
    /**
     * Cache database query results
     */
    public function cacheQuery(string $key, string $sql, int $ttl = 300): array
    {
        return $this->remember($key, $ttl, function() use ($sql) {
            return Capsule::select($sql);
        });
    }
}

/**
 * Cache frequently accessed data
 */
function getCachedProductPricing(int $productId): array
{
    $cache = new PerformanceCacheManager();
    
    return $cache->remember("product_pricing_{$productId}", 3600, function() use ($productId) {
        return Capsule::table('tblpricing')
            ->join('tblcurrencies', 'tblcurrencies.id', '=', 'tblpricing.currency')
            ->where('tblpricing.relid', $productId)
            ->where('tblpricing.type', 'product')
            ->get()
            ->toArray();
    });
}

/**
 * Cache client summary data
 */
function getCachedClientSummary(int $clientId): array
{
    $cache = new PerformanceCacheManager();
    
    return $cache->remember("client_summary_{$clientId}", 1800, function() use ($clientId) {
        $client = Capsule::table('tblclients')
            ->where('id', $clientId)
            ->first();
        
        $stats = Capsule::table('tblhosting')
            ->where('userid', $clientId)
            ->selectRaw('COUNT(*) as total_services, SUM(amount) as total_revenue')
            ->first();
        
        return [
            'client' => $client,
            'stats' => $stats,
        ];
    });
}
```

### Step 4: Performance Dashboard

Create real-time performance monitoring:

```php
// modules/addons/performance_dashboard/performance_dashboard.php

function perfdash_config(): array
{
    return [
        'name' => 'Performance Dashboard',
        'description' => 'Real-time WHMCS performance monitoring',
        'version' => '1.0',
    ];
}

function perfdash_activate(): array
{
    Capsule::schema()->create('perf_metrics', function($table) {
        $table->increments('id');
        $table->string('metric_name', 100);
        $table->decimal('value', 10, 4);
        $table->string('unit', 20);
        $table->timestamp('recorded_at')->useCurrent();
        $table->index(['metric_name', 'recorded_at']);
    });
    
    Capsule::schema()->create('perf_slow_queries', function($table) {
        $table->increments('id');
        $table->string('query_hash', 32);
        $table->text('query');
        $table->decimal('execution_time_ms', 10, 4);
        $table->text('explain_data');
        $table->timestamp('created_at')->useCurrent();
    });
    
    return ['status' => 'success'];
}

function perfdash_output(array $vars): void
{
    echo perfdash_render_dashboard();
}

function perfdash_render_dashboard(): string
{
    $stats = perfdash_get_stats();
    $recentQueries = perfdash_get_recent_slow_queries();
    $historical = perfdash_get_historical_data();
    
    $html = '<div class="performance-dashboard">';
    $html .= '<h2>Performance Dashboard</h2>';
    
    // Key metrics
    $html .= '<div class="metrics-grid">';
    $html .= perfdash_render_metric_card('Avg Response Time', $stats['avg_response_time'] . ' ms', 'success');
    $html .= perfdash_render_metric_card('DB Queries/Page', $stats['queries_per_page'], 'info');
    $html .= perfdash_render_metric_card('Cache Hit Rate', ($stats['cache_hit_rate'] * 100) . '%', 'success');
    $html .= perfdash_render_metric_card('Error Rate', $stats['error_rate'] . '%', 'danger');
    $html .= '</div>';
    
    // Historical chart placeholder
    $html .= '<div class="chart-container">';
    $html .= '<h3>Response Time History (24h)</h3>';
    $html .= '<canvas id="perfChart"></canvas>';
    $html .= '</div>';
    
    // Slow queries table
    $html .= '<h3>Recent Slow Queries</h3>';
    $html .= '<table class="datatable"><thead><tr>';
    $html .= '<th>Query</th><th>Time (ms)</th><th>Date</th>';
    $html .= '</tr></thead><tbody>';
    
    foreach ($recentQueries as $query) {
        $html .= '<tr>';
        $html .= '<td>' . htmlspecialchars(substr($query->query, 0, 100)) . '...</td>';
        $html .= '<td>' . $query->execution_time_ms . '</td>';
        $html .= '<td>' . $query->created_at . '</td>';
        $html .= '</tr>';
    }
    
    $html .= '</tbody></table>';
    $html .= '</div>';
    
    return $html;
}

function perfdash_get_stats(): array
{
    $cache = new PerformanceCacheManager();
    
    return $cache->remember('dashboard_stats', 60, function() {
        $hourAgo = date('Y-m-d H:i:s', strtotime('-1 hour'));
        
        $metrics = Capsule::table('perf_metrics')
            ->where('recorded_at', '>', $hourAgo)
            ->get()
            ->groupBy('metric_name');
        
        $stats = [];
        
        foreach ($metrics as $name => $values) {
            $numericValues = array_map(fn($m) => (float)$m->value, $values->toArray());
            $stats[$name] = array_sum($numericValues) / count($numericValues);
        }
        
        return [
            'avg_response_time' => round($stats['response_time_ms'] ?? 0, 2),
            'queries_per_page' => round($stats['db_queries'] ?? 0, 1),
            'cache_hit_rate' => round($stats['cache_hit_rate'] ?? 0, 4),
            'error_rate' => round($stats['error_rate'] ?? 0, 2),
        ];
    });
}

function perfdash_render_metric_card(string $title, string $value, string $type): string
{
    $iconMap = [
        'success' => 'fa-check-circle',
        'warning' => 'fa-exclamation-triangle',
        'danger' => 'fa-exclamation-circle',
        'info' => 'fa-info-circle',
    ];
    
    return <<<HTML
    <div class="metric-card metric-{$type}">
        <i class="fas {$iconMap[$type]}"></i>
        <h4>{$title}</h4>
        <p class="metric-value">{$value}</p>
    </div>
HTML;
}

/**
 * Record performance metrics hook
 */
add_hook('OutputBufferStart', 1, function($vars) {
    $GLOBALS['perf_start'] = microtime(true);
    DatabaseQueryMonitor::enable();
});

add_hook('OutputBufferEnd', 1, function($vars) {
    if (isset($GLOBALS['perf_start'])) {
        $duration = (microtime(true) - $GLOBALS['perf_start']) * 1000;
        
        Capsule::table('perf_metrics')->insert([
            'metric_name' => 'response_time_ms',
            'value' => $duration,
            'unit' => 'ms',
        ]);
        
        // Check threshold
        if ($duration > 3000) {
            logActivity("Slow page load detected: {$duration}ms - {$_SERVER['REQUEST_URI']}");
        }
    }
});
```

### Step 5: Server Resource Monitoring

Monitor server resources:

```bash
#!/bin/bash
# monitor_server.sh - Server monitoring script

#!/bin/bash

# CPU Usage
CPU_USAGE=$(top -bn1 | grep "Cpu(s)" | awk '{print $2}' | cut -d'%' -f1)
echo "CPU Usage: ${CPU_USAGE}%"

# Memory Usage
MEM_TOTAL=$(free -m | awk '/^Mem:/{print $2}')
MEM_USED=$(free -m | awk '/^Mem:/{print $3}')
MEM_PERCENT=$((MEM_USED * 100 / MEM_TOTAL))
echo "Memory Usage: ${MEM_PERCENT}% (${MEM_USED}/${MEM_TOTAL} MB)"

# Disk Usage
DISK_USAGE=$(df -h / | awk 'NR==2 {print $5}' | cut -d'%' -f1)
echo "Disk Usage: ${DISK_USAGE}%"

# MySQL Connections
MYSQL_CONNECTIONS=$(mysql -u whmcs_user -p$DB_PASS -e "SHOW STATUS LIKE 'Threads_connected';" | awk 'NR==2 {print $2}')
echo "MySQL Connections: ${MYSQL_CONNECTIONS}"

# Apache/Nginx Workers
APACHE_WORKERS=$(ps aux | grep -c '[a]pache2')
echo "Apache Workers: ${APACHE_WORKERS}"

# PHP-FPM Processes
PHPFPM_PROCESSES=$(ps aux | grep -c '[p]hp-fpm')
echo "PHP-FPM Processes: ${PHPFPM_PROCESSES}"

# Log to database
mysql -u whmcs_user -p$DB_PASS -e "
INSERT INTO perf_metrics (metric_name, value, unit, recorded_at) VALUES
('cpu_percent', ${CPU_USAGE}, 'percent', NOW()),
('memory_percent', ${MEM_PERCENT}, 'percent', NOW()),
('disk_percent', ${DISK_USAGE}, 'percent', NOW()),
('mysql_connections', ${MYSQL_CONNECTIONS}, 'count', NOW());
"
```

## Best Practices

1. **Monitor continuously** - Set up automated monitoring
2. **Establish baselines** - Know your normal metrics
3. **Alert on thresholds** - Get notified of issues
4. **Track trends** - Look for gradual degradation
5. **Optimize queries** - Fix slow queries promptly
6. **Use caching** - Reduce database load
7. **Review logs** - Check for recurring issues
8. **Capacity planning** - Plan for growth

## Common Pitfalls to Avoid

1. **Ignoring slow queries** - They compound over time
2. **No caching strategy** - Database becomes bottleneck
3. **Insufficient resources** - Monitor and upgrade proactively
4. **Not monitoring in production** - Staging doesn't reflect reality
5. **Alert fatigue** - Set meaningful thresholds
6. **Missing indexes** - Causes slow queries
7. **Large session data** - Memory bloat
8. **Not reviewing trends** - Historical data is valuable
