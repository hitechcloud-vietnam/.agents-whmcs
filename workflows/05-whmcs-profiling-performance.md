# WHMCS Profiling & Performance Workflow

## Overview
This workflow covers profiling WHMCS modules, identifying performance bottlenecks, and implementing optimizations.

## Prerequisites
- Blackfire.io or XHProf for profiling
- MySQL slow query log enabled
- APM monitoring tools

## Step 1: Performance Monitoring Setup

### Enable MySQL Slow Query Log

```ini
; /etc/mysql/mysql.conf.d/mysqld.cnf
[mysqld]
slow_query_log = 1
slow_query_log_file = /var/log/mysql/slow.log
long_query_time = 1
log_queries_not_using_indexes = 1
```

```bash
# Restart MySQL
sudo systemctl restart mysql

# View slow queries
sudo tail -f /var/log/mysql/slow.log
```

### Configure PHP Performance

```ini
; php.ini
memory_limit = 256M
max_execution_time = 60
opcache.enable = 1
opcache.memory_consumption = 256
opcache.max_accelerated_files = 20000
realpath_cache_size = 4096K
realpath_cache_ttl = 600
```

## Step 2: WHMCS Performance Logger

```php
<?php
// src/Helper/PerformanceLogger.php

namespace WHMCS\Module\Addon\YourModule\Helper;

class PerformanceLogger
{
    private static $marks = [];
    private static $enabled = false;
    private static $logPath;

    public static function enable(string $logPath = null): void
    {
        self::$enabled = true;
        self::$logPath = $logPath ?? dirname(__DIR__, 3) . '/storage/logs/performance.log';
        self::$marks = [];
    }

    public static function mark(string $name, array $context = []): void
    {
        if (!self::$enabled) {
            return;
        }

        self::$marks[$name] = [
            'time' => microtime(true),
            'memory' => memory_get_usage(true),
            'context' => $context
        ];
    }

    public static function measure(string $start, string $end): array
    {
        if (!isset(self::$marks[$start]) || !isset(self::$marks[$end])) {
            return ['error' => 'Missing marks'];
        }

        $startMark = self::$marks[$start];
        $endMark = self::$marks[$end];

        return [
            'duration_ms' => round(($endMark['time'] - $startMark['time']) * 1000, 2),
            'memory_delta' => $endMark['memory'] - $startMark['memory'],
            'start_memory' => $startMark['memory'],
            'end_memory' => $endMark['memory'],
            'start_context' => $startMark['context'],
            'end_context' => $endMark['context']
        ];
    }

    public static function report(): string
    {
        $output = ["=== Performance Report ===", "Time: " . date('Y-m-d H:i:s'), ""];

        $marks = array_keys(self::$marks);
        for ($i = 0; $i < count($marks) - 1; $i++) {
            $measure = self::measure($marks[$i], $marks[$i + 1]);
            $output[] = sprintf(
                "%s -> %s: %s ms | Memory: %s",
                $marks[$i],
                $marks[$i + 1],
                $measure['duration_ms'],
                number_format($measure['memory_delta'] / 1024 / 1024, 2) . ' MB'
            );
        }

        $output[] = "";
        $output[] = "Total marks: " . count(self::$marks);
        $output[] = "Peak memory: " . round(memory_get_peak_usage(true) / 1024 / 1024, 2) . ' MB';

        return implode("\n", $output);
    }

    public static function log(): void
    {
        if (self::$logPath) {
            file_put_contents(self::$logPath, self::report() . "\n\n", FILE_APPEND);
        }
    }
}
```

## Step 3: Query Performance Analysis

### Create Query Analyzer

```php
<?php
// src/Helper/QueryAnalyzer.php

namespace WHMCS\Module\Addon\YourModule\Helper;

use WHMCS\Database\Capsule;

class QueryAnalyzer
{
    private static $queries = [];
    private static $enabled = false;

    public static function enable(): void
    {
        self::$enabled = true;
        self::$queries = [];
        Capsule::connection()->enableQueryLog();
    }

    public static function getQueries(): array
    {
        return Capsule::connection()->getQueryLog();
    }

    public static function analyze(): array
    {
        $queries = self::getQueries();
        $analysis = [
            'total_queries' => count($queries),
            'total_time' => 0,
            'slow_queries' => [],
            'duplicate_queries' => [],
            'missing_indexes' => [],
            'query_types' => []
        ];

        $queryHashes = [];
        foreach ($queries as $index => $query) {
            $start = microtime(true);

            // Get query info
            $sql = $query['query'];
            $hash = md5($sql);

            // Categorize by type
            $type = self::getQueryType($sql);
            $analysis['query_types'][$type] = ($analysis['query_types'][$type] ?? 0) + 1;

            // Check for duplicates
            if (isset($queryHashes[$hash])) {
                $analysis['duplicate_queries'][] = [
                    'sql' => $sql,
                    'count' => $queryHashes[$hash]['count'] + 1,
                    'first_index' => $queryHashes[$hash]['index']
                ];
            }
            $queryHashes[$hash] = ['count' => 1, 'index' => $index];

            // Check for missing indexes (SELECT without LIMIT or with LIKE)
            if (preg_match('/^SELECT/i', $sql) && !preg_match('/LIMIT/i', $sql)) {
                $analysis['missing_indexes'][] = $sql;
            }

            $executionTime = (microtime(true) - $start) * 1000;
            $analysis['total_time'] += $executionTime;

            if ($executionTime > 10) {
                $analysis['slow_queries'][] = [
                    'sql' => $sql,
                    'execution_time_ms' => round($executionTime, 2)
                ];
            }
        }

        return $analysis;
    }

    private static function getQueryType(string $sql): string
    {
        $type = strtoupper(preg_replace('/\s+.*$/s', '', trim($sql)));
        return in_array($type, ['SELECT', 'INSERT', 'UPDATE', 'DELETE', 'CREATE', 'DROP', 'ALTER'])
            ? $type
            : 'OTHER';
    }

    public static function report(): string
    {
        $analysis = self::analyze();
        $output = ["=== Query Analysis ===", ""];
        $output[] = "Total Queries: " . $analysis['total_queries'];
        $output[] = "Total Time: " . round($analysis['total_time'], 2) . " ms";
        $output[] = "";
        $output[] = "Query Types:";
        foreach ($analysis['query_types'] as $type => $count) {
            $output[] = "  - $type: $count";
        }

        if (!empty($analysis['slow_queries'])) {
            $output[] = "";
            $output[] = "Slow Queries (>10ms):";
            foreach (array_slice($analysis['slow_queries'], 0, 5) as $query) {
                $output[] = "  - " . substr($query['sql'], 0, 100) . "... ({$query['execution_time_ms']} ms)";
            }
        }

        if (!empty($analysis['duplicate_queries'])) {
            $output[] = "";
            $output[] = "Duplicate Queries:";
            foreach (array_slice($analysis['duplicate_queries'], 0, 3) as $query) {
                $output[] = "  - " . substr($query['sql'], 0, 100) . "... (executed {$query['count']} times)";
            }
        }

        return implode("\n", $output);
    }
}
```

### Usage in Module

```php
<?php
// Example usage
use WHMCS\Module\Addon\YourModule\Helper\PerformanceLogger;
use WHMCS\Module\Addon\YourModule\Helper\QueryAnalyzer;

PerformanceLogger::enable();
QueryAnalyzer::enable();

PerformanceLogger::mark('process_start');

// Your module code with queries
$clients = Capsule::table('tblclients')
    ->where('status', 'Active')
    ->get();

foreach ($clients as $client) {
    // N+1 query problem here
    $services = Capsule::table('tblhosting')
        ->where('userid', $client->id)
        ->get();
}

PerformanceLogger::mark('process_end');

// Output analysis
echo QueryAnalyzer::report();
echo "\n\n";
echo PerformanceLogger::report();

// Log for later analysis
QueryAnalyzer::log();
PerformanceLogger::log();
```

## Step 4: Optimize Database Queries

### Eager Loading (Fix N+1)

```php
<?php
// BAD: N+1 Query
foreach ($clients as $client) {
    $services = Capsule::table('tblhosting')
        ->where('userid', $client->id)
        ->get();
    // This runs a query for EACH client
}

// GOOD: Eager Loading
$clients = Capsule::table('tblclients')
    ->where('status', 'Active')
    ->with(['services']) // Requires relationship defined
    ->get();

// Or manual eager loading
$clientIds = array_column($clients, 'id');
$servicesByClient = Capsule::table('tblhosting')
    ->whereIn('userid', $clientIds)
    ->get()
    ->groupBy('userid');

foreach ($clients as $client) {
    $services = $servicesByClient[$client->id] ?? [];
}
```

### Chunking Large Operations

```php
<?php
// Process large datasets in chunks
$chunkSize = 100;
$processed = 0;

 Capsule::table('tblhosting')
    ->where('domainstatus', 'Active')
    ->chunk($chunkSize, function ($services) use (&$processed) {
        foreach ($services as $service) {
            // Process each service
            processService($service);
            $processed++;
        }

        // Yield to database between chunks
        Capsule::connection()->disconnect();
    });

// Alternative: chunkById for better performance
 Capsule::table('tblhosting')
    ->where('domainstatus', 'Active')
    ->orderBy('id')
    ->chunkById(100, function ($services) use (&$processed) {
        foreach ($services as $service) {
            processService($service);
            $processed++;
        }
    }, 'id');
```

### Add Database Indexes

```php
<?php
// In your module activation
function your_module_activate()
{
    // Add indexes for common queries
    $indexes = [
        // For WHERE userid = ? queries
        "CREATE INDEX idx_userid ON mod_your_table (userid)",

        // For WHERE status = ? AND date > ? queries
        "CREATE INDEX idx_status_date ON mod_your_table (status, created_at)",

        // For WHERE email LIKE ? queries
        "CREATE INDEX idx_email ON mod_your_table (email)"
    ];

    foreach ($indexes as $index) {
        try {
            full_query($index);
        } catch (\Exception $e) {
            // Index might already exist
        }
    }
}
```

## Step 5: Caching Implementation

```php
<?php
// src/Helper/CacheManager.php

namespace WHMCS\Module\Addon\YourModule\Helper;

use WHMCS\Module\Addon\YourModule\Service\ConfigService;

class CacheManager
{
    private $cacheDir;

    public function __construct()
    {
        $this->cacheDir = dirname(__DIR__, 3) . '/storage/module_cache/';
        if (!is_dir($this->cacheDir)) {
            mkdir($this->cacheDir, 0755, true);
        }
    }

    public function get(string $key, int $ttl = 3600): ?array
    {
        $cacheFile = $this->cacheDir . md5($key) . '.cache';

        if (!file_exists($cacheFile)) {
            return null;
        }

        $cache = json_decode(file_get_contents($cacheFile), true);

        if ($cache['expires'] < time()) {
            unlink($cacheFile);
            return null;
        }

        return $cache['data'];
    }

    public function set(string $key, array $data, int $ttl = 3600): void
    {
        $cacheFile = $this->cacheDir . md5($key) . '.cache';
        $cache = [
            'data' => $data,
            'expires' => time() + $ttl,
            'created' => time()
        ];

        file_put_contents($cacheFile, json_encode($cache));
    }

    public function invalidate(string $key): void
    {
        $cacheFile = $this->cacheDir . md5($key) . '.cache';
        if (file_exists($cacheFile)) {
            unlink($cacheFile);
        }
    }

    public function invalidatePattern(string $pattern): void
    {
        $files = glob($this->cacheDir . '*.cache');
        foreach ($files as $file) {
            $key = str_replace(['.cache', $this->cacheDir], '', $file);
            if (preg_match('/' . $pattern . '/', $key)) {
                unlink($file);
            }
        }
    }

    public function clear(): void
    {
        $files = glob($this->cacheDir . '*.cache');
        foreach ($files as $file) {
            unlink($file);
        }
    }
}
```

### Use Caching in Module

```php
<?php
// src/Service/ClientService.php

namespace WHMCS\Module\Addon\YourModule\Service;

use WHMCS\Module\Addon\YourModule\Helper\CacheManager;
use WHMCS\Database\Capsule;

class ClientService
{
    private $cache;

    public function __construct()
    {
        $this->cache = new CacheManager();
    }

    public function getActiveClients(): array
    {
        $cacheKey = 'active_clients';

        // Try cache first
        $cached = $this->cache->get($cacheKey, 300); // 5 min TTL
        if ($cached !== null) {
            return $cached;
        }

        // Query database
        $clients = Capsule::table('tblclients')
            ->where('status', 'Active')
            ->orderBy('id')
            ->get()
            ->toArray();

        // Cache result
        $this->cache->set($cacheKey, $clients);

        return $clients;
    }

    public function invalidateClientCache(int $clientId): void
    {
        $this->cache->invalidate('active_clients');
        $this->cache->invalidate('client_' . $clientId);
    }
}
```

## Step 6: Performance Testing

### Benchmark Script

```php
<?php
// benchmark.php

require_once __DIR__ . '/includes/init.php';

use WHMCS\Module\Addon\YourModule\Helper\PerformanceLogger;

echo "=== WHMCS Module Performance Benchmark ===\n\n";

// Test 1: Database Operations
echo "Test 1: Database Operations\n";
PerformanceLogger::enable();
PerformanceLogger::mark('db_start');

for ($i = 0; $i < 100; $i++) {
    Capsule::table('tblclients')
        ->where('id', ($i % 50) + 1)
        ->first();
}

PerformanceLogger::mark('db_end');
echo PerformanceLogger::report();
echo "\n";

// Test 2: Module Service
echo "Test 2: Module Service\n";
PerformanceLogger::enable();
PerformanceLogger::mark('service_start');

$service = new \WHMCS\Module\Addon\YourModule\Service\ModuleService();
for ($i = 0; $i < 50; $i++) {
    $service->process(['id' => $i, 'data' => 'test']);
}

PerformanceLogger::mark('service_end');
echo PerformanceLogger::report();
```

### Apache Bench for API

```bash
# Test WHMCS API endpoint performance
ab -n 100 -c 10 -g results.tsv "https://yoursite.com/whmcs/api.php?action=GetClients&username=apiuser&password=apipass&responsetype=json"

# Generate HTML report
gnuplot -e "
    set terminal png;
    set output 'benchmark.png';
    set title 'Response Time Distribution';
    plot 'results.tsv' using 4 with lines;
"
```

## Step 7: Blackfire Integration

### Install Blackfire PHP Agent

```bash
# Install Blackfire PHP agent
wget -O - https://package.blackfire.io/gpg.key | sudo apt-key add -
echo "deb http://packages.blackfire.io/debian any main" | sudo tee /etc/apt/sources.list.d/blackfire.list
sudo apt-get update
sudo apt-get install blackfire-agent blackfire-php

# Configure agent
sudo blackfire-agent --register --server-id=YOUR_SERVER_ID --server-token=YOUR_SERVER_TOKEN

# Restart PHP-FPM
sudo systemctl restart php8.1-fpm
```

### Profile a Request

```bash
# Install Blackfire CLI
curl -Lsf https://packages.blackfire.io/blackfire-cli.linux_amd64.tgz | sudo tar zxvf - -C /usr/local/bin

# Profile a WHMCS page
blackfire curl https://yoursite.com/whmcs/clientarea.php

# Profile API endpoint
blackfire curl https://yoursite.com/whmcs/api.php?action=GetClients

# Compare profiles
blackfire comparison run \
    --baseline=previous-profile-id \
    current-profile-id
```

## Verification Checklist

- [ ] MySQL slow query log enabled
- [ ] PHP opcache configured
- [ ] Performance logger implemented
- [ ] Query analyzer created
- [ ] N+1 queries identified and fixed
- [ ] Indexes added for frequent queries
- [ ] Caching implemented
- [ ] Large operations chunked
- [ ] Performance benchmark script created
- [ ] Profiling tool installed (Blackfire/XHProf)
- [ ] Performance targets defined and met

## Performance Targets

| Metric | Target |
|--------|--------|
| Page Load Time | < 2s |
| API Response | < 500ms |
| Database Query | < 100ms |
| Memory Usage | < 128MB |
| Queries per Page | < 20 |
