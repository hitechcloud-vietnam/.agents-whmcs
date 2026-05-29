# WHMCS Debugging

Complete guide to debugging WHMCS issues.

## Overview

Identify and resolve issues effectively.

## Debug Mode

### Enable Debug Mode

```php
<?php
/**
 * Debug configuration
 */

// In configuration.php
$debug_mode = true;
$display_errors = true;

// Debug log function
function debug_log(string $message, array $context = []): void
{
    if (!defined('DEBUG_MODE') || !DEBUG_MODE) {
        return;
    }
    
    $logFile = __DIR__ . '/../logs/debug.log';
    
    $entry = [
        'timestamp' => date('Y-m-d H:i:s.u'),
        'message' => $message,
        'context' => $context,
        'memory' => round(memory_get_usage(true) / 1024 / 1024, 2) . 'MB',
        'peak_memory' => round(memory_get_peak_usage(true) / 1024 / 1024, 2) . 'MB',
    ];
    
    file_put_contents($logFile, json_encode($entry) . "\n", FILE_APPEND);
}
```

## Error Handling

### Custom Error Handler

```php
<?php
/**
 * Custom error handler
 */
set_error_handler(function($severity, $message, $file, $line) {
    $error = [
        'type' => 'PHP Error',
        'severity' => $severity,
        'message' => $message,
        'file' => $file,
        'line' => $line,
        'timestamp' => date('Y-m-d H:i:s'),
        'trace' => debug_backtrace(DEBUG_BACKTRACE_IGNORE_ARGS),
    ];
    
    // Log error
    Capsule::table('mod_error_log')->insert([
        'error_type' => 'php_error',
        'error_data' => json_encode($error),
        'created_at' => date('Y-m-d H:i:s'),
    ]);
    
    // Log to file
    error_log("[PHP Error] {$message} in {$file}:{$line}");
    
    // In debug mode, throw exception
    if (defined('DEBUG_MODE') && DEBUG_MODE) {
        throw new ErrorException($message, 0, $severity, $file, $line);
    }
    
    return true;
});

/**
 * Exception handler
 */
set_exception_handler(function(Throwable $e) {
    $error = [
        'type' => get_class($e),
        'message' => $e->getMessage(),
        'code' => $e->getCode(),
        'file' => $e->getFile(),
        'line' => $e->getLine(),
        'trace' => $e->getTrace(),
        'timestamp' => date('Y-m-d H:i:s'),
    ];
    
    // Log error
    Capsule::table('mod_error_log')->insert([
        'error_type' => get_class($e),
        'error_data' => json_encode($error),
        'created_at' => date('Y-m-d H:i:s'),
    ]);
    
    // Log to file
    error_log("[Exception] {$e->getMessage()} in {$e->getFile()}:{$e->getLine()}");
    
    // Show error page
    if (defined('DEBUG_MODE') && DEBUG_MODE) {
        echo "<h1>Exception: {$e->getMessage()}</h1>";
        echo "<pre>" . $e->getTraceAsString() . "</pre>";
    } else {
        echo "An error occurred. Please try again.";
    }
});
```

## Logging Utilities

### Debug Logger

```php
<?php
/**
 * Debug logging utility
 */
class DebugLogger
{
    private string $logFile;
    private bool $enabled;
    
    public function __construct(string $logFile = null)
    {
        $this->logFile = $logFile ?? __DIR__ . '/../logs/debug.log';
        $this->enabled = defined('DEBUG_MODE') && DEBUG_MODE;
        
        if (!is_dir(dirname($this->logFile))) {
            mkdir(dirname($this->logFile), 0755, true);
        }
    }
    
    /**
     * Log debug message
     */
    public function debug(string $message, array $context = []): void
    {
        $this->log('DEBUG', $message, $context);
    }
    
    /**
     * Log info message
     */
    public function info(string $message, array $context = []): void
    {
        $this->log('INFO', $message, $context);
    }
    
    /**
     * Log warning
     */
    public function warning(string $message, array $context = []): void
    {
        $this->log('WARNING', $message, $context);
    }
    
    /**
     * Log error
     */
    public function error(string $message, array $context = []): void
    {
        $this->log('ERROR', $message, $context);
    }
    
    /**
     * Log with level
     */
    private function log(string $level, string $message, array $context): void
    {
        if (!$this->enabled) {
            return;
        }
        
        $entry = [
            'timestamp' => date('Y-m-d H:i:s.u'),
            'level' => $level,
            'message' => $message,
            'context' => $context,
            'memory' => memory_get_usage(true),
            'peak_memory' => memory_get_peak_usage(true),
        ];
        
        $entry['context']['_memory'] = [
            'current' => round(memory_get_usage(true) / 1024 / 1024, 2) . 'MB',
            'peak' => round(memory_get_peak_usage(true) / 1024 / 1024, 2) . 'MB',
        ];
        
        file_put_contents($this->logFile, json_encode($entry) . "\n", FILE_APPEND);
    }
    
    /**
     * Log function entry/exit
     */
    public function trace(string $function, array $params = []): void
    {
        $this->log('TRACE', "ENTER: {$function}", ['params' => $params]);
    }
    
    /**
     * Log function exit
     */
    public function untrace(string $function, $result = null): void
    {
        $this->log('TRACE', "EXIT: {$function}", ['result' => $result]);
    }
    
    /**
     * Dump variable
     */
    public function dump(string $label, $variable): void
    {
        $this->log('DUMP', $label, [
            'type' => gettype($variable),
            'value' => is_scalar($variable) ? $variable : get_class($variable),
        ]);
    }
    
    /**
     * Read recent logs
     */
    public function readRecent(int $lines = 100): array
    {
        if (!file_exists($this->logFile)) {
            return [];
        }
        
        $content = file($this->logFile);
        $recent = array_slice($content, -$lines);
        
        return array_map('json_decode', $recent);
    }
    
    /**
     * Clear logs
     */
    public function clear(): void
    {
        if (file_exists($this->logFile)) {
            unlink($this->logFile);
        }
    }
}
```

## Query Debugging

### Query Logger

```php
<?php
/**
 * Query debugging
 */
class QueryDebugger
{
    private array $queries = [];
    private float $totalTime = 0;
    private bool $enabled;
    
    public function __construct(bool $enabled = false)
    {
        $this->enabled = $enabled;
    }
    
    /**
     * Log query
     */
    public function log(string $query, array $bindings = [], float $time = 0): void
    {
        if (!$this->enabled) {
            return;
        }
        
        $this->queries[] = [
            'query' => $query,
            'bindings' => $bindings,
            'time' => $time,
            'timestamp' => microtime(true),
        ];
        
        $this->totalTime += $time;
    }
    
    /**
     * Get query report
     */
    public function report(): array
    {
        return [
            'total_queries' => count($this->queries),
            'total_time' => round($this->totalTime * 1000, 2),
            'slow_queries' => array_filter($this->queries, fn($q) => $q['time'] > 0.1),
            'queries' => $this->queries,
        ];
    }
    
    /**
     * Print report
     */
    public function printReport(): void
    {
        $report = $this->report();
        
        echo "Query Debug Report\n";
        echo str_repeat('=', 50) . "\n";
        echo "Total Queries: {$report['total_queries']}\n";
        echo "Total Time: {$report['total_time']}ms\n";
        echo "Slow Queries: " . count($report['slow_queries']) . "\n";
        echo str_repeat('=', 50) . "\n\n";
        
        foreach ($report['queries'] as $i => $query) {
            $time = round($query['time'] * 1000, 2);
            $marker = $query['time'] > 0.1 ? ' [SLOW]' : '';
            
            echo "[{$i}] {$time}ms{$marker}\n";
            echo "SQL: {$query['query']}\n";
            
            if (!empty($query['bindings'])) {
                echo "Bindings: " . json_encode($query['bindings']) . "\n";
            }
            
            echo "\n";
        }
    }
}
```

## Performance Profiling

### Profiler

```php
<?php
/**
 * Performance profiler
 */
class Profiler
{
    private array $checkpoints = [];
    private array $memorySnapshots = [];
    
    /**
     * Start profiling
     */
    public function start(): void
    {
        $this->checkpoints = [['start', microtime(true)]];
        $this->memorySnapshots = [['start', memory_get_usage(true)]];
    }
    
    /**
     * Add checkpoint
     */
    public function checkpoint(string $name): void
    {
        $time = microtime(true);
        $memory = memory_get_usage(true);
        
        $lastCheckpoint = end($this->checkpoints);
        $lastMemory = end($this->memorySnapshots);
        
        $this->checkpoints[] = [$name, $time];
        $this->memorySnapshots[] = [$name, $memory];
    }
    
    /**
     * Get profile report
     */
    public function report(): array
    {
        $report = [];
        
        for ($i = 1; $i < count($this->checkpoints); $i++) {
            $current = $this->checkpoints[$i];
            $previous = $this->checkpoints[$i - 1];
            
            $timeDelta = ($current[1] - $previous[1]) * 1000;
            $memoryDelta = ($this->memorySnapshots[$i][1] - $this->memorySnapshots[$i - 1][1]) / 1024;
            
            $report[] = [
                'checkpoint' => $current[0],
                'time_ms' => round($timeDelta, 2),
                'memory_kb' => round($memoryDelta, 2),
            ];
        }
        
        return $report;
    }
    
    /**
     * Print report
     */
    public function printReport(): void
    {
        $report = $this->report();
        
        echo "Performance Profile\n";
        echo str_repeat('=', 60) . "\n";
        echo str_pad("Checkpoint", 30) . str_pad("Time (ms)", 15) . "Memory (KB)\n";
        echo str_repeat('-', 60) . "\n";
        
        foreach ($report as $item) {
            echo str_pad($item['checkpoint'], 30);
            echo str_pad($item['time_ms'], 15);
            echo $item['memory_kb'] . "\n";
        }
        
        echo str_repeat('=', 60) . "\n";
    }
}
```

## Common Issues

### Issue Detection

```php
<?php
/**
 * Detect common issues
 */
function diagnoseWHMCS(): array
{
    $issues = [];
    
    // Check PHP version
    if (version_compare(PHP_VERSION, '7.4', '<')) {
        $issues[] = [
            'severity' => 'error',
            'issue' => 'PHP version too old',
            'recommendation' => 'Upgrade to PHP 7.4 or higher',
        ];
    }
    
    // Check required extensions
    $required = ['pdo', 'pdo_mysql', 'curl', 'gd', 'mbstring'];
    foreach ($required as $ext) {
        if (!extension_loaded($ext)) {
            $issues[] = [
                'severity' => 'error',
                'issue' => "Missing PHP extension: {$ext}",
                'recommendation' => "Install the {$ext} extension",
            ];
        }
    }
    
    // Check disk space
    $freeSpace = disk_free_space(__DIR__);
    if ($freeSpace < 100 * 1024 * 1024) { // 100MB
        $issues[] = [
            'severity' => 'warning',
            'issue' => 'Low disk space',
            'recommendation' => 'Free up disk space',
        ];
    }
    
    // Check database connection
    try {
        Capsule::select('SELECT 1');
    } catch (Exception $e) {
        $issues[] = [
            'severity' => 'error',
            'issue' => 'Database connection failed',
            'recommendation' => 'Check database configuration',
        ];
    }
    
    // Check for maintenance mode
    if (file_exists(__DIR__ . '/maintenance.php')) {
        $issues[] = [
            'severity' => 'info',
            'issue' => 'Maintenance mode is enabled',
            'recommendation' => 'Remove maintenance.php to disable',
        ];
    }
    
    return $issues;
}
```

## Best Practices

1. **Enable debug mode** - In development only
2. **Log everything** - Comprehensive logging
3. **Profile queries** - Monitor slow queries
4. **Check error logs** - Review regularly
5. **Version control** - Track configuration changes
6. **Test in staging** - Reproduce issues safely

## Related Documentation

- [whmcs-advanced-testing.md](whmcs-advanced-testing.md)
- [whmcs-advanced-performance.md](whmcs-advanced-performance.md)
