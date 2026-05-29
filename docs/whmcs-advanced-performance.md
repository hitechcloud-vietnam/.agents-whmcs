# WHMCS Advanced Performance

Complete guide to performance optimization.

## Overview

Optimize WHMCS for maximum speed and efficiency.

## Query Optimization

### Query Builder Optimization

```php
<?php
/**
 * Optimized query examples
 */
class OptimizedQueries
{
    /**
     * Use indexes efficiently
     */
    public function getActiveServicesWithClient(): array
    {
        // Use SELECT with specific columns instead of *
        return Capsule::table('tblhosting')
            ->select([
                'tblhosting.id',
                'tblhosting.domain',
                'tblhosting.domainstatus',
                'tblhosting.nextduedate',
                'tblclients.email',
                'tblclients.firstname',
                'tblclients.lastname',
            ])
            ->join('tblclients', 'tblhosting.userid', '=', 'tblclients.id')
            ->where('tblhosting.domainstatus', 'Active')
            ->where('tblhosting.nextduedate', '<=', date('Y-m-d'))
            ->orderBy('tblhosting.nextduedate', 'asc')
            ->limit(100)
            ->get()
            ->toArray();
    }
    
    /**
     * Use chunking for large datasets
     */
    public function processAllClients(): void
    {
        Capsule::table('tblclients')
            ->where('status', 'Active')
            ->chunk(100, function($clients) {
                foreach ($clients as $client) {
                    // Process client
                    $this->processClient($client);
                }
            });
    }
    
    /**
     * Use pagination
     */
    public function getClientsPaginated(int $page = 1, int $perPage = 50): array
    {
        $offset = ($page - 1) * $perPage;
        
        $clients = Capsule::table('tblclients')
            ->orderBy('id', 'desc')
            ->offset($offset)
            ->limit($perPage)
            ->get();
        
        $total = Capsule::table('tblclients')->count();
        
        return [
            'data' => $clients->toArray(),
            'pagination' => [
                'current_page' => $page,
                'per_page' => $perPage,
                'total' => $total,
                'last_page' => ceil($total / $perPage),
            ],
        ];
    }
    
    /**
     * Use subqueries efficiently
     */
    public function getClientsWithActiveServices(): array
    {
        return Capsule::table('tblclients')
            ->whereIn('id', function($query) {
                $query->select('userid')
                    ->from('tblhosting')
                    ->where('domainstatus', 'Active');
            })
            ->get()
            ->toArray();
    }
}
```

## Response Time Optimization

### Response Time Monitor

```php
<?php
/**
 * Monitor and optimize response times
 */
class ResponseTimeOptimizer
{
    private array $checkpoints = [];
    
    /**
     * Start timing
     */
    public function start(): void
    {
        $this->checkpoints = ['start' => microtime(true)];
    }
    
    /**
     * Add checkpoint
     */
    public function checkpoint(string $name): void
    {
        $this->checkpoints[$name] = microtime(true);
    }
    
    /**
     * Get timing report
     */
    public function report(): array
    {
        $report = [];
        $previous = null;
        
        foreach ($this->checkpoints as $name => $time) {
            if ($previous === null) {
                $report[$name] = ['total_ms' => 0, 'delta_ms' => 0];
            } else {
                $delta = ($time - $this->checkpoints[$previous]) * 1000;
                $total = ($time - $this->checkpoints['start']) * 1000;
                $report[$name] = [
                    'total_ms' => round($total, 2),
                    'delta_ms' => round($delta, 2),
                ];
            }
            $previous = $name;
        }
        
        return $report;
    }
    
    /**
     * Check if within threshold
     */
    public function isAcceptable(float $thresholdMs): bool
    {
        $total = (end($this->checkpoints) - reset($this->checkpoints)) * 1000;
        return $total < $thresholdMs;
    }
}
```

### Optimization Hook

```php
<?php
/**
 * Performance monitoring hook
 */
add_hook('PreAuthentication', 1, function($vars) {
    $optimizer = new ResponseTimeOptimizer();
    $optimizer->start();
    
    // Store in session for later retrieval
    $_SESSION['performance_start'] = microtime(true);
});
```

## Database Optimization

### Connection Pooling

```php
<?php
/**
 * Database connection pool
 */
class ConnectionPool
{
    private static $pool = [];
    private static $maxConnections = 10;
    
    /**
     * Get connection from pool
     */
    public static function getConnection(): PDO
    {
        $hash = spl_object_hash(PDO::getAvailableDrivers());
        
        if (!isset(self::$pool[$hash])) {
            if (count(self::$pool) >= self::$maxConnections) {
                // Return least recently used
                array_shift(self::$pool);
            }
            
            self::$pool[$hash] = new PDO(
                'mysql:host=' . DB_HOST . ';dbname=' . DB_NAME,
                DB_USERNAME,
                DB_PASSWORD
            );
        }
        
        return self::$pool[$hash];
    }
    
    /**
     * Clear pool
     */
    public static function clearPool(): void
    {
        self::$pool = [];
    }
}
```

### Query Caching

```php
<?php
/**
 * Query result caching
 */
class CachedQueryBuilder
{
    private RedisCache $cache;
    private string $query;
    private array $bindings;
    private int $ttl;
    
    public function __construct(RedisCache $cache)
    {
        $this->cache = $cache;
        $this->ttl = 300; // 5 minutes default
    }
    
    /**
     * Execute cached query
     */
    public function get(string $query, array $bindings = []): array
    {
        $cacheKey = $this->generateCacheKey($query, $bindings);
        
        $cached = $this->cache->get($cacheKey);
        if ($cached !== null) {
            return $cached;
        }
        
        $result = Capsule::select($query, $bindings);
        $this->cache->set($cacheKey, $result, $this->ttl);
        
        return $result;
    }
    
    /**
     * Generate cache key
     */
    private function generateCacheKey(string $query, array $bindings): string
    {
        return 'query:' . md5($query . serialize($bindings));
    }
    
    /**
     * Set TTL
     */
    public function ttl(int $seconds): self
    {
        $this->ttl = $seconds;
        return $this;
    }
    
    /**
     * Invalidate cache
     */
    public function invalidate(string $pattern): int
    {
        return $this->cache->deletePattern("query:{$pattern}*");
    }
}
```

## Asset Optimization

### Asset Minification

```php
<?php
/**
 * Asset optimization
 */
class AssetOptimizer
{
    /**
     * Minify CSS
     */
    public function minifyCSS(string $css): string
    {
        // Remove comments
        $css = preg_replace('/\/\*[\s\S]*?\*\//', '', $css);
        
        // Remove whitespace
        $css = preg_replace('/\s+/', ' ', $css);
        
        // Remove spaces around special characters
        $css = preg_replace('/\s*([{}:;,])\s*/', '$1', $css);
        
        // Remove trailing semicolons
        $css = preg_replace('/;}/', '}', $css);
        
        return trim($css);
    }
    
    /**
     * Minify JavaScript
     */
    public function minifyJS(string $js): string
    {
        // Remove comments
        $js = preg_replace('/\/\/.*$/m', '', $js);
        $js = preg_replace('/\/\*[\s\S]*?\*\//', '', $js);
        
        // Remove whitespace
        $js = preg_replace('/\s+/', ' ', $js);
        
        // Remove spaces around operators
        $js = preg_replace('/\s*([{}()=;,:])\s*/', '$1', $js);
        
        return trim($js);
    }
    
    /**
     * Generate asset hash
     */
    public function generateHash(string $content): string
    {
        return substr(md5($content), 0, 8);
    }
}
```

## Lazy Loading

### Lazy Loading Implementation

```php
<?php
/**
 * Lazy loading for services
 */
class LazyServiceLoader
{
    private RedisCache $cache;
    
    public function __construct(RedisCache $cache)
    {
        $this->cache = $cache;
    }
    
    /**
     * Lazy load service details
     */
    public function loadService(int $serviceId): LazyService
    {
        return new LazyService($serviceId, $this->cache);
    }
}

/**
 * Lazy service proxy
 */
class LazyService
{
    private int $serviceId;
    private ?array $data = null;
    private RedisCache $cache;
    
    public function __construct(int $serviceId, RedisCache $cache)
    {
        $this->serviceId = $serviceId;
        $this->cache = $cache;
    }
    
    /**
     * Get property on demand
     */
    public function __get(string $name)
    {
        if ($this->data === null) {
            $this->load();
        }
        
        return $this->data[$name] ?? null;
    }
    
    /**
     * Load data only when accessed
     */
    private function load(): void
    {
        $cached = $this->cache->get("service:{$this->serviceId}");
        
        if ($cached !== null) {
            $this->data = $cached;
            return;
        }
        
        $service = Capsule::table('tblhosting')
            ->where('id', $this->serviceId)
            ->first();
        
        $this->data = (array)$service;
        $this->cache->set("service:{$this->serviceId}", $this->data, 3600);
    }
}
```

## Async Processing

### Background Jobs

```php
<?php
/**
 * Background job processor
 */
class BackgroundJobProcessor
{
    private RedisCache $queue;
    
    public function __construct(RedisCache $queue)
    {
        $this->queue = $queue;
    }
    
    /**
     * Queue a job
     */
    public function dispatch(string $job, array $data, int $delay = 0): void
    {
        $jobData = [
            'job' => $job,
            'data' => $data,
            'queued_at' => time(),
            'execute_at' => time() + $delay,
        ];
        
        $this->queue->set(
            'job:' . uniqid(),
            $jobData,
            86400
        );
    }
    
    /**
     * Process jobs
     */
    public function process(): int
    {
        $processed = 0;
        
        // Process pending jobs
        $jobs = $this->queue->getPendingJobs();
        
        foreach ($jobs as $jobId => $job) {
            if ($job['execute_at'] <= time()) {
                $this->executeJob($job);
                $this->queue->completeJob($jobId);
                $processed++;
            }
        }
        
        return $processed;
    }
    
    /**
     * Execute job
     */
    private function executeJob(array $job): void
    {
        switch ($job['job']) {
            case 'SendEmail':
                $this->executeEmailJob($job['data']);
                break;
            case 'ProcessOrder':
                $this->executeOrderJob($job['data']);
                break;
            case 'SyncData':
                $this->executeSyncJob($job['data']);
                break;
        }
    }
    
    private function executeEmailJob(array $data): void
    {
        // Email sending logic
    }
    
    private function executeOrderJob(array $data): void
    {
        // Order processing logic
    }
    
    private function executeSyncJob(array $data): void
    {
        // Data sync logic
    }
}
```

## Best Practices

1. **Profile first** - Identify actual bottlenecks
2. **Cache aggressively** - Cache expensive operations
3. **Use indexes** - Ensure proper database indexing
4. **Lazy load** - Load data only when needed
5. **Async operations** - Process heavy tasks in background
6. **Monitor trends** - Track performance over time

## Related Documentation

- [whmcs-advanced-caching.md](whmcs-advanced-caching.md)
- [whmcs-advanced-database.md](whmcs-advanced-database.md)
