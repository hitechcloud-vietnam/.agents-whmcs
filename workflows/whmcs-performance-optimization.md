# WHMCS Performance Optimization Workflow

## Purpose

Complete guide to optimizing WHMCS performance. Covers database optimization, caching strategies, PHP tuning, CDN integration, query optimization, and monitoring.

## Prerequisites

- WHMCS 7.0+ installation
- SSH access to server
- Database administration tools
- PHP knowledge
- Basic performance monitoring tools

## Workflow Steps

### Step 1: Database Optimization

```php
<?php
/**
 * WHMCS Database Optimization
 */

// ===========================================
// Query Optimization Crons
// ===========================================

#!/usr/bin/php
<?php
/**
 * Database Query Optimization Cron
 *
 * Identifies and optimizes slow queries.
 */

define('ROOTDIR', dirname(__DIR__, 4));
require_once ROOTDIR . '/init.php';

class DatabaseQueryOptimizer
{
    private $slowQueryThreshold = 1.0; // seconds
    private $logTable = 'mod_query_optimization_log';

    /**
     * Analyze and optimize database
     */
    public function optimize(): array
    {
        $results = [
            'indexes_added' => 0,
            'tables_optimized' => 0,
            'queries_analyzed' => 0,
            'recommendations' => [],
        ];

        // Analyze slow query log
        $slowQueries = $this->getSlowQueries();

        foreach ($slowQueries as $query) {
            $results['queries_analyzed']++;

            $recommendation = $this->analyzeQuery($query);
            if ($recommendation) {
                $results['recommendations'][] = $recommendation;
            }
        }

        // Optimize frequently used queries
        $results['tables_optimized'] = $this->optimizeTables();

        // Add missing indexes
        $results['indexes_added'] = $this->addMissingIndexes();

        return $results;
    }

    /**
     * Get slow queries from logs
     */
    private function getSlowQueries(): array
    {
        // Enable slow query log temporarily
        Capsule::statement('SET profiling = 1');

        // Run common WHMCS queries and collect timing
        $queries = [
            'GetClients' => "SELECT * FROM tblclients LIMIT 100",
            'ActiveServices' => "SELECT * FROM tblhosting WHERE domainstatus = 'Active'",
            'RecentInvoices' => "SELECT * FROM tblinvoices WHERE status IN ('Unpaid','Overdue')",
        ];

        $results = [];
        foreach ($queries as $name => $sql) {
            Capsule::statement("SET profiling = 1");
            $start = microtime(true);
            Capsule::select($sql);
            $duration = microtime(true) - $start;

            $results[$name] = [
                'sql' => $sql,
                'duration' => $duration,
                'slow' => $duration > $this->slowQueryThreshold,
            ];
        }

        Capsule::statement('SET profiling = 0');

        return $results;
    }

    /**
     * Analyze query for optimization opportunities
     */
    private function analyzeQuery(array $query): ?array
    {
        $sql = $query['sql'];
        $recommendations = [];

        // Check for missing WHERE clause
        if (preg_match('/^(SELECT|UPDATE|DELETE)/i', $sql) && !preg_match('/\bWHERE\b/i', $sql)) {
            $recommendations[] = 'Query lacks WHERE clause - may affect all rows';
        }

        // Check for SELECT *
        if (strpos($sql, 'SELECT *') === 0) {
            $recommendations[] = 'Use specific columns instead of SELECT *';
        }

        // Check for missing indexes in JOINs
        if (preg_match_all('/JOIN\s+(\w+)/i', $sql, $matches)) {
            foreach ($matches[1] as $table) {
                if (!$this->hasProperIndexes($table)) {
                    $recommendations[] = "Table {$table} may need indexes for JOIN";
                }
            }
        }

        if (!empty($recommendations)) {
            return [
                'query' => $sql,
                'duration' => $query['duration'],
                'recommendations' => $recommendations,
            ];
        }

        return null;
    }

    /**
     * Optimize database tables
     */
    private function optimizeTables(): int
    {
        $tables = [
            'tblactivitylog',
            'tblmodulelog',
            'tbllogs',
            'tblsessions',
        ];

        $optimized = 0;
        foreach ($tables as $table) {
            try {
                if (Capsule::schema()->hasTable($table)) {
                    Capsule::statement("OPTIMIZE TABLE {$table}");
                    $optimized++;
                }
            } catch (\Exception $e) {
                logActivity("Failed to optimize {$table}: " . $e->getMessage());
            }
        }

        return $optimized;
    }

    /**
     * Add missing indexes based on query patterns
     */
    private function addMissingIndexes(): int
    {
        $indexesToAdd = [
            ['tblhosting', ['server', 'domainstatus', 'userid'], 'idx_server_status_user'],
            ['tblinvoices', ['userid', 'status', 'duedate'], 'idx_user_status_due'],
            ['tblorders', ['userid', 'status', 'createdat'], 'idx_user_status_created'],
            ['tblticketmessages', ['tid', 'date'], 'idx_ticket_date'],
        ];

        $added = 0;
        foreach ($indexesToAdd as $index) {
            try {
                $table = $index[0];
                $columns = $index[1];
                $name = $index[2];

                if (Capsule::schema()->hasTable($table)) {
                    Capsule::statement("CREATE INDEX IF NOT EXISTS {$name} ON {$table} (" . implode(',', $columns) . ")");
                    $added++;
                }
            } catch (\Exception $e) {
                // Index may already exist
            }
        }

        return $added;
    }

    private function hasProperIndexes(string $table): bool
    {
        $indexes = Capsule::select("SHOW INDEX FROM {$table}");
        return count($indexes) > 0;
    }
}

// Run optimization
if (php_sapi_name() === 'cli') {
    $optimizer = new DatabaseQueryOptimizer();
    $results = $optimizer->optimize();

    echo "Optimization Results:\n";
    echo "Indexes added: {$results['indexes_added']}\n";
    echo "Tables optimized: {$results['tables_optimized']}\n";
    echo "Recommendations: " . count($results['recommendations']) . "\n";

    foreach ($results['recommendations'] as $rec) {
        echo "- Query {$rec['query']}: " . implode(', ', $rec['recommendations']) . "\n";
    }
}
```

### Step 2: Caching Implementation

```php
<?php
/**
 * WHMCS Caching Layer
 *
 * Multi-tier caching implementation for optimal performance.
 */

class WHMCS_Cache_Manager
{
    private $drivers = [];
    private $defaultTTL = 3600; // 1 hour

    /**
     * Initialize cache drivers
     */
    public function __construct()
    {
        // File cache
        $this->drivers['file'] = new FileCacheDriver([
            'path' => ROOTDIR . '/cache/vendor/',
        ]);

        // Redis cache (if available)
        if (extension_loaded('redis')) {
            try {
                $this->drivers['redis'] = new RedisCacheDriver([
                    'host' => getenv('REDIS_HOST') ?: '127.0.0.1',
                    'port' => getenv('REDIS_PORT') ?: 6379,
                    'prefix' => 'whmcs_',
                ]);
            } catch (\Exception $e) {
                logActivity("Redis connection failed: " . $e->getMessage());
            }
        }

        // Memcached cache (if available)
        if (extension_loaded('memcached')) {
            try {
                $this->drivers['memcached'] = new MemcachedCacheDriver([
                    'host' => getenv('MEMCACHED_HOST') ?: '127.0.0.1',
                    'port' => getenv('MEMCACHED_PORT') ?: 11211,
                ]);
            } catch (\Exception $e) {
                logActivity("Memcached connection failed: " . $e->getMessage());
            }
        }
    }

    /**
     * Get cache driver (auto-select best available)
     */
    public function getDriver(string $preferred = null): CacheDriverInterface
    {
        if ($preferred && isset($this->drivers[$preferred])) {
            return $this->drivers[$preferred];
        }

        // Priority: Redis > Memcached > File
        foreach (['redis', 'memcached', 'file'] as $driver) {
            if (isset($this->drivers[$driver])) {
                return $this->drivers[$driver];
            }
        }

        // Fallback to file cache
        return $this->drivers['file'];
    }

    /**
     * Get cached value
     */
    public function get(string $key, $default = null, int $ttl = null)
    {
        $driver = $this->getDriver();
        $value = $driver->get($key);

        return $value !== null ? $value : $default;
    }

    /**
     * Set cached value
     */
    public function set(string $key, $value, int $ttl = null): bool
    {
        $driver = $this->getDriver();
        $ttl = $ttl ?? $this->defaultTTL;

        return $driver->set($key, $value, $ttl);
    }

    /**
     * Delete cached value
     */
    public function delete(string $key): bool
    {
        $driver = $this->getDriver();
        return $driver->delete($key);
    }

    /**
     * Clear all cache
     */
    public function flush(): bool
    {
        $success = true;
        foreach ($this->drivers as $driver) {
            if (!$driver->flush()) {
                $success = false;
            }
        }
        return $success;
    }

    /**
     * Remember pattern - get from cache or execute callback
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
}

/**
 * Cache interface
 */
interface CacheDriverInterface
{
    public function get(string $key);
    public function set(string $key, $value, int $ttl): bool;
    public function delete(string $key): bool;
    public function flush(): bool;
}

/**
 * File cache driver
 */
class FileCacheDriver implements CacheDriverInterface
{
    private $path;
    private $maxSize = 10485760; // 10MB

    public function __construct(array $config)
    {
        $this->path = $config['path'];
        if (!is_dir($this->path)) {
            mkdir($this->path, 0755, true);
        }
    }

    public function get(string $key): mixed
    {
        $file = $this->getFilePath($key);

        if (!file_exists($file)) {
            return null;
        }

        $data = unserialize(file_get_contents($file));

        if ($data['expires'] < time()) {
            unlink($file);
            return null;
        }

        return $data['value'];
    }

    public function set(string $key, $value, int $ttl): bool
    {
        $file = $this->getFilePath($key);
        $data = [
            'value' => $value,
            'expires' => time() + $ttl,
        ];

        return file_put_contents($file, serialize($data)) !== false;
    }

    public function delete(string $key): bool
    {
        $file = $this->getFilePath($key);

        if (file_exists($file)) {
            return unlink($file);
        }

        return true;
    }

    public function flush(): bool
    {
        $files = glob($this->path . '/*.cache');

        foreach ($files as $file) {
            if (is_file($file)) {
                unlink($file);
            }
        }

        return true;
    }

    private function getFilePath(string $key): string
    {
        $hash = md5($key);
        return $this->path . '/' . $hash . '.cache';
    }
}

/**
 * Redis cache driver
 */
class RedisCacheDriver implements CacheDriverInterface
{
    private $redis;
    private $prefix;

    public function __construct(array $config)
    {
        $this->redis = new \Redis();
        $this->redis->connect($config['host'], $config['port']);
        $this->prefix = $config['prefix'] ?? 'whmcs_';
    }

    public function get(string $key): mixed
    {
        $value = $this->redis->get($this->prefix . $key);

        if ($value === false) {
            return null;
        }

        return unserialize($value);
    }

    public function set(string $key, $value, int $ttl): bool
    {
        return $this->redis->setex(
            $this->prefix . $key,
            $ttl,
            serialize($value)
        );
    }

    public function delete(string $key): bool
    {
        return $this->redis->del($this->prefix . $key) > 0;
    }

    public function flush(): bool
    {
        $keys = $this->redis->keys($this->prefix . '*');
        foreach ($keys as $key) {
            $this->redis->del($key);
        }
        return true;
    }
}

/**
 * Usage example - Cached WHMCS data retrieval
 */
add_hook('ClientAreaPageRun', 1, function() {
    $cache = new WHMCS_Cache_Manager();

    // Cache client统计数据
    $clientId = $_SESSION['uid'] ?? 0;

    if ($clientId) {
        $stats = $cache->remember(
            "client_stats_{$clientId}",
            300, // 5 minutes
            function() use ($clientId) {
                return [
                    'active_services' => Capsule::table('tblhosting')
                        ->where('userid', $clientId)
                        ->where('domainstatus', 'Active')
                        ->count(),
                    'total_spent' => Capsule::table('tblinvoices')
                        ->where('userid', $clientId)
                        ->where('status', 'Paid')
                        ->sum('total'),
                ];
            }
        );

        // Make available in template
        return ['clientStats' => $stats];
    }
});
```

### Step 3: PHP Optimization

```ini
; php.ini optimization settings

; Memory and execution settings
memory_limit = 256M
max_execution_time = 300
max_input_time = 300

; OpCache settings (PHP 8.0+)
opcache.enable = 1
opcache.enable_cli = 0
opcache.memory_consumption = 256
opcache.interned_strings_buffer = 32
opcache.max_accelerated_files = 20000
opcache.validate_timestamps = 0 ; Disable in production
opcache.revalidate_freq = 0
opcache.fast_shutdown = 1
opcache.enable_file_override = 1

; JIT settings (PHP 8.0+)
opcache.jit_buffer_size = 128M
opcache.jit = function

; Session optimization
session.gc_maxlifetime = 3600
session.cookie_lifetime = 3600
session.cache_expire = 180
session.use_strict_mode = 1
session.use_only_cookies = 1
session.cookie_httponly = 1
session.cookie_secure = 1

; File upload optimization
upload_max_filesize = 50M
post_max_size = 60M
max_file_uploads = 20
```

### Step 4: Query Caching for Common Operations

```php
<?php
/**
 * WHMCS Query Caching Hooks
 *
 * Caches common expensive queries for improved performance.
 */

// Cache product catalog
add_hook('ProductDetailsPreload', 1, function(array $vars) {
    $cacheKey = "product_details_{$vars['pid']}";
    $cache = new WHMCS_Cache_Manager();

    return $cache->remember($cacheKey, 1800, function() use ($vars) {
        // Expensive product query with relations
        return Capsule::table('tblproducts')
            ->join('tblproduct_groups', 'tblproducts.gid', '=', 'tblproduct_groups.id')
            ->leftJoin('tblpricing', function($join) {
                $join->on('tblpricing.relid', '=', 'tblproducts.id')
                     ->where('tblpricing.type', 'product');
            })
            ->where('tblproducts.id', $vars['pid'])
            ->first();
    });
});

// Cache pricing data
add_hook('GetCartItems', 1, function() {
    $cache = new WHMCS_Cache_Manager();

    return $cache->remember('pricing_cache', 900, function() {
        return Capsule::table('tblpricing')
            ->where('type', 'product')
            ->get()
            ->keyBy('relid');
    });
});

// Cache currency exchange rates
add_hook('DailyCronJob', 1, function() {
    $cache = new WHMCS_Cache_Manager();

    // Refresh exchange rates cache
    $rates = fetchExchangeRates();
    $cache->set('exchange_rates', $rates, 3600);

    return true;
});

// Cache country list
add_hook('ClientAreaPageRun', 1, function() {
    $cache = new WHMCS_Cache_Manager();

    return $cache->remember('countries_list', 86400, function() {
        return Capsule::table('tblcountries')
            ->orderBy('country')
            ->get()
            ->toArray();
    });
});
```

### Step 5: Lazy Loading Implementation

```php
<?php
/**
 * WHMCS Lazy Loading Implementation
 */

// Lazy load service details
class LazyServiceLoader
{
    private $services = [];
    private $loaded = false;

    /**
     * Get services by IDs (lazy loading)
     */
    public function getServices(array $serviceIds): array
    {
        $missing = [];

        foreach ($serviceIds as $id) {
            if (!isset($this->services[$id])) {
                $missing[] = $id;
            }
        }

        if (!empty($missing)) {
            $loaded = Capsule::table('tblhosting')
                ->whereIn('id', $missing)
                ->get()
                ->keyBy('id');

            $this->services = array_merge($this->services, $loaded->toArray());
        }

        return array_intersect_key($this->services, array_flip($serviceIds));
    }

    /**
     * Get single service with caching
     */
    public function getService(int $serviceId): ?array
    {
        $cache = new WHMCS_Cache_Manager();
        $cacheKey = "service_{$serviceId}";

        $service = $cache->remember($cacheKey, 3600, function() use ($serviceId) {
            return Capsule::table('tblhosting')
                ->where('id', $serviceId)
                ->first();
        });

        return $service;
    }
}

/**
 * Pagination with cursor-based loading
 */
class EfficientPaginator
{
    private $pageSize = 50;
    private $cursor;

    /**
     * Get paginated results efficiently
     */
    public function getPage(string $table, array $conditions, $cursor = null): array
    {
        $query = Capsule::table($table)
            ->where($conditions)
            ->orderBy('id', 'desc')
            ->limit($this->pageSize + 1);

        if ($cursor) {
            $query->where('id', '<', $cursor);
        }

        $results = $query->get();

        $hasMore = count($results) > $this->pageSize;
        if ($hasMore) {
            array_pop($results);
        }

        $nextCursor = $hasMore && !empty($results) ? end($results)->id : null;

        return [
            'items' => $results,
            'next_cursor' => $nextCursor,
            'has_more' => $hasMore,
        ];
    }
}
```

### Step 6: Background Processing

```php
<?php
/**
 * WHMCS Background Job Processing
 *
 * Move expensive operations to background processing.
 */

// Queue expensive operations
add_hook('AfterModuleCreate', 50, function(array $vars) {
    // Don't send welcome emails synchronously
    $this->queueBackgroundJob('send_welcome_email', [
        'service_id' => $vars['serviceid'],
        'client_id' => $vars['userid'],
    ]);

    // Don't sync to external systems synchronously
    $this->queueBackgroundJob('sync_to_crm', [
        'service_id' => $vars['serviceid'],
    ]);

    return true;
});

/**
 * Job queue handler
 */
class BackgroundJobQueue
{
    private $queueTable = 'mod_background_jobs';
    private $maxRetries = 3;

    public function __construct()
    {
        if (!Capsule::schema()->hasTable($this->queueTable)) {
            Capsule::schema()->create($this->queueTable, function($table) {
                $table->increments('id');
                $table->string('job_name', 100);
                $table->longText('payload');
                $table->string('status', 20)->default('pending');
                $table->integer('attempts')->default(0);
                $table->timestamp('scheduled_at')->useCurrent();
                $table->text('error_message')->nullable();
                $table->index(['status', 'scheduled_at']);
            });
        }
    }

    public function queue(string $jobName, array $payload, int $delaySeconds = 0): int
    {
        return Capsule::table($this->queueTable)->insertGetId([
            'job_name' => $jobName,
            'payload' => json_encode($payload),
            'status' => 'pending',
            'scheduled_at' => date('Y-m-d H:i:s', time() + $delaySeconds),
        ]);
    }

    public function process(): int
    {
        $jobs = Capsule::table($this->queueTable)
            ->where('status', 'pending')
            ->where('scheduled_at', '<=', date('Y-m-d H:i:s'))
            ->limit(10)
            ->get();

        $processed = 0;

        foreach ($jobs as $job) {
            try {
                $this->executeJob($job);

                Capsule::table($this->queueTable)
                    ->where('id', $job->id)
                    ->update([
                        'status' => 'completed',
                        'completed_at' => date('Y-m-d H:i:s'),
                    ]);

                $processed++;
            } catch (\Exception $e) {
                Capsule::table($this->queueTable)
                    ->where('id', $job->id)
                    ->update([
                        'attempts' => $job->attempts + 1,
                        'error_message' => $e->getMessage(),
                        'status' => $job->attempts + 1 >= $this->maxRetries ? 'failed' : 'pending',
                    ]);
            }
        }

        return $processed;
    }

    private function executeJob(object $job): void
    {
        $payload = json_decode($job->payload, true);

        switch ($job->job_name) {
            case 'send_welcome_email':
                sendMessage('Service Provisioned', $payload['service_id']);
                break;

            case 'sync_to_crm':
                $crm = new ExternalCRM();
                $crm->syncService($payload['service_id']);
                break;

            case 'generate_invoice':
                $this->generateInvoiceForService($payload['service_id']);
                break;

            default:
                logActivity("Unknown job: {$job->job_name}");
        }
    }
}

/**
 * Background job processing cron
 */
add_hook('DailyCronJob', 1, function() {
    $queue = new BackgroundJobQueue();
    $processed = $queue->process();

    if ($processed > 0) {
        logActivity("Processed {$processed} background jobs");
    }
});
```

## Performance Monitoring

```php
<?php
/**
 * Performance Monitoring
 */

class PerformanceMonitor
{
    private $metricsTable = 'mod_performance_metrics';

    public function __construct()
    {
        if (!Capsule::schema()->hasTable($this->metricsTable)) {
            Capsule::schema()->create($this->metricsTable, function($table) {
                $table->increments('id');
                $table->string('metric_name', 100);
                $table->decimal('value', 20, 4);
                $table->string('unit', 20);
                $table->timestamp('recorded_at')->useCurrent();
                $table->index(['metric_name', 'recorded_at']);
            });
        }
    }

    public function recordMetric(string $name, float $value, string $unit = ''): void
    {
        Capsule::table($this->metricsTable)->insert([
            'metric_name' => $name,
            'value' => $value,
            'unit' => $unit,
            'recorded_at' => date('Y-m-d H:i:s'),
        ]);
    }

    public function getMetrics(string $name, int $hours = 24): array
    {
        return Capsule::table($this->metricsTable)
            ->where('metric_name', $name)
            ->where('recorded_at', '>=', date('Y-m-d H:i:s', time() - ($hours * 3600)))
            ->orderBy('recorded_at', 'asc')
            ->get()
            ->toArray();
    }

    public function getAverageResponseTime(int $hours = 24): array
    {
        return Capsule::table($this->metricsTable)
            ->where('metric_name', 'page_response_time')
            ->where('recorded_at', '>=', date('Y-m-d H:i:s', time() - ($hours * 3600)))
            ->selectRaw('AVG(value) as avg_value, MAX(value) as max_value, MIN(value) as min_value')
            ->first();
    }
}

// Track page load times
add_hook('ClientAreaPageRun', 1, function() {
    if (!isset($_SESSION['request_start'])) {
        $_SESSION['request_start'] = $_SERVER['REQUEST_TIME_FLOAT'] ?? microtime(true);
        return;
    }

    $loadTime = (microtime(true) - $_SESSION['request_start']) * 1000; // ms

    $monitor = new PerformanceMonitor();
    $monitor->recordMetric('page_response_time', $loadTime, 'ms');

    // Track specific pages
    $route = $_GET['route'] ?? 'unknown';
    $monitor->recordMetric("page_{$route}_time", $loadTime, 'ms');
});
```

## Verification Checklist

```
Database:
□ Slow query log enabled
□ Indexes on frequently queried columns
□ Query caching implemented
□ Table optimization scheduled

Caching:
□ Multi-tier cache configured
□ Page caching for static content
□ Query caching for expensive operations
□ Cache invalidation working

PHP:
□ OpCache enabled and configured
□ JIT enabled (PHP 8.0+)
□ Memory limits appropriate
□ Session optimization configured

Application:
□ Lazy loading for relationships
□ Background job processing
□ Database connection pooling
□ Response compression enabled

Infrastructure:
□ CDN configured for static assets
□ Load balancer in place
□ Database read replicas configured
□ Proper server resource allocation
```

## WHMCS ClassDocs References

- [Caching Strategies](https://developers.whmcs.com/advanced/caching/)
- [Database Optimization](https://developers.whmcs.com/advanced/database-optimization/)
- [PHP Configuration](https://developers.whmcs.com/advanced/php-configuration/)
- [logActivity()](https://developers.whmcs.com/advanced/logging/)
