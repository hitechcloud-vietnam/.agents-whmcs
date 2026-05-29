# WHMCS Logging Integration

Complete guide for logging and monitoring integrations.

## Overview

Implement comprehensive logging for audit trails and monitoring.

## Custom Logging

### Log Manager

```php
<?php
/**
 * Custom log manager
 */
class LogManager
{
    private string $logDir;
    private string $logFile;
    private int $maxFileSize;
    private int $maxFiles;
    
    public function __construct(string $logDir = null, string $prefix = 'whmcs')
    {
        $this->logDir = $logDir ?? __DIR__ . '/../logs';
        $this->logFile = $this->logDir . '/' . $prefix . '_' . date('Y-m-d') . '.log';
        $this->maxFileSize = 10 * 1024 * 1024; // 10MB
        $this->maxFiles = 30;
        
        if (!is_dir($this->logDir)) {
            mkdir($this->logDir, 0755, true);
        }
    }
    
    /**
     * Write log entry
     */
    public function log(string $message, string $level = 'INFO', array $context = []): void
    {
        $this->ensureRotation();
        
        $entry = $this->formatEntry($message, $level, $context);
        
        file_put_contents($this->logFile, $entry . "\n", FILE_APPEND);
    }
    
    /**
     * Format log entry
     */
    private function formatEntry(string $message, string $level, array $context): string
    {
        $entry = [
            'timestamp' => date('Y-m-d H:i:s'),
            'level' => $level,
            'message' => $message,
        ];
        
        if (!empty($context)) {
            $entry['context'] = $context;
        }
        
        // Add request info if available
        if (isset($_SERVER['REQUEST_URI'])) {
            $entry['request'] = [
                'uri' => $_SERVER['REQUEST_URI'],
                'method' => $_SERVER['REQUEST_METHOD'] ?? 'CLI',
                'ip' => $_SERVER['REMOTE_ADDR'] ?? 'CLI',
            ];
        }
        
        return json_encode($entry);
    }
    
    /**
     * Log levels
     */
    public function debug(string $message, array $context = []): void
    {
        $this->log($message, 'DEBUG', $context);
    }
    
    public function info(string $message, array $context = []): void
    {
        $this->log($message, 'INFO', $context);
    }
    
    public function warning(string $message, array $context = []): void
    {
        $this->log($message, 'WARNING', $context);
    }
    
    public function error(string $message, array $context = []): void
    {
        $this->log($message, 'ERROR', $context);
    }
    
    public function critical(string $message, array $context = []): void
    {
        $this->log($message, 'CRITICAL', $context);
    }
    
    /**
     * Rotate logs if needed
     */
    private function ensureRotation(): void
    {
        if (file_exists($this->logFile) && filesize($this->logFile) >= $this->maxFileSize) {
            rename($this->logFile, $this->logFile . '.' . time());
        }
    }
    
    /**
     * Clean old logs
     */
    public function cleanup(): int
    {
        $files = glob($this->logDir . '/*.log*');
        $cutoff = strtotime("-{$this->maxFiles} days");
        
        $deleted = 0;
        foreach ($files as $file) {
            if (filemtime($file) < $cutoff) {
                unlink($file);
                $deleted++;
            }
        }
        
        return $deleted;
    }
}
```

## Audit Logging

### Audit Trail

```php
<?php
/**
 * Audit log for tracking changes
 */
class AuditLogger
{
    /**
     * Log an audit event
     */
    public static function log(string $action, string $entity, int $entityId, array $data = []): int
    {
        return Capsule::table('mod_audit_log')->insertGetId([
            'action' => $action,
            'entity_type' => $entity,
            'entity_id' => $entityId,
            'user_id' => $_SESSION['uid'] ?? 0,
            'user_ip' => $_SERVER['REMOTE_ADDR'] ?? '',
            'data' => json_encode($data),
            'created_at' => date('Y-m-d H:i:s'),
        ]);
    }
    
    /**
     * Get audit trail for entity
     */
    public static function getTrail(string $entity, int $entityId, int $limit = 50): array
    {
        return Capsule::table('mod_audit_log')
            ->where('entity_type', $entity)
            ->where('entity_id', $entityId)
            ->orderBy('created_at', 'desc')
            ->limit($limit)
            ->get()
            ->toArray();
    }
}

/**
 * Create audit table
 */
function createAuditTable(): void
{
    if (!Capsule::schema()->hasTable('mod_audit_log')) {
        Capsule::schema()->create('mod_audit_log', function($table) {
            $table->increments('id');
            $table->string('action', 50);
            $table->string('entity_type', 50);
            $table->integer('entity_id')->unsigned();
            $table->integer('user_id')->unsigned()->nullable();
            $table->string('user_ip', 45)->nullable();
            $table->json('data')->nullable();
            $table->timestamp('created_at')->useCurrent();
            
            $table->index(['entity_type', 'entity_id']);
            $table->index(['user_id']);
            $table->index(['created_at']);
        });
    }
}
```

## Activity Tracking

```php
<?php
/**
 * Track user activity
 */
class ActivityTracker
{
    /**
     * Track page view
     */
    public static function trackPageView(string $page, string $userId = null): void
    {
        Capsule::table('mod_activity_log')->insert([
            'activity_type' => 'page_view',
            'activity_data' => json_encode(['page' => $page]),
            'user_id' => $userId ?? $_SESSION['uid'] ?? 0,
            'user_ip' => $_SERVER['REMOTE_ADDR'] ?? '',
            'user_agent' => $_SERVER['HTTP_USER_AGENT'] ?? '',
            'created_at' => date('Y-m-d H:i:s'),
        ]);
    }
    
    /**
     * Track API call
     */
    public static function trackApiCall(string $endpoint, string $method, int $responseCode, float $duration): void
    {
        Capsule::table('mod_activity_log')->insert([
            'activity_type' => 'api_call',
            'activity_data' => json_encode([
                'endpoint' => $endpoint,
                'method' => $method,
                'response_code' => $responseCode,
                'duration_ms' => round($duration * 1000, 2),
            ]),
            'created_at' => date('Y-m-d H:i:s'),
        ]);
    }
    
    /**
     * Get popular pages
     */
    public static function getPopularPages(int $days = 7): array
    {
        $cutoff = date('Y-m-d H:i:s', strtotime("-{$days} days"));
        
        return Capsule::table('mod_activity_log')
            ->selectRaw('JSON_EXTRACT(activity_data, "$.page") as page, COUNT(*) as views')
            ->where('activity_type', 'page_view')
            ->where('created_at', '>', $cutoff)
            ->groupBy('page')
            ->orderBy('views', 'desc')
            ->limit(20)
            ->get();
    }
}
```

## Log Aggregation

### ELK Stack Integration

```php
<?php
/**
 * Send logs to Elasticsearch
 */
class ElasticsearchLogger
{
    private string $host;
    private string $index;
    private string $apiKey;
    
    public function __construct(string $host, string $index, string $apiKey)
    {
        $this->host = rtrim($host, '/');
        $this->index = $index;
        $this->apiKey = $apiKey;
    }
    
    /**
     * Send log document
     */
    public function send(array $document): bool
    {
        $document['@timestamp'] = date('c');
        
        $ch = curl_init("{$this->host}/{$this->index}/_doc");
        curl_setopt_array($ch, [
            CURLOPT_POST => true,
            CURLOPT_POSTFIELDS => json_encode($document),
            CURLOPT_RETURNTRANSFER => true,
            CURLOPT_HTTPHEADER => [
                'Content-Type: application/json',
                'Authorization: ApiKey ' . $this->apiKey,
            ],
        ]);
        
        curl_exec($ch);
        $httpCode = curl_getinfo($ch, CURLINFO_HTTP_CODE);
        curl_close($ch);
        
        return $httpCode >= 200 && $httpCode < 300;
    }
    
    /**
     * Bulk send logs
     */
    public function bulkSend(array $documents): bool
    {
        $body = '';
        foreach ($documents as $doc) {
            $body .= json_encode(['index' => ['_index' => $this->index]]) . "\n";
            $body .= json_encode(array_merge($doc, ['@timestamp' => date('c')])) . "\n";
        }
        
        $ch = curl_init("{$this->host}/_bulk");
        curl_setopt_array($ch, [
            CURLOPT_POST => true,
            CURLOPT_POSTFIELDS => $body,
            CURLOPT_RETURNTRANSFER => true,
            CURLOPT_HTTPHEADER => [
                'Content-Type: application/x-ndjson',
                'Authorization: ApiKey ' . $this->apiKey,
            ],
        ]);
        
        $response = json_decode(curl_exec($ch), true);
        curl_close($ch);
        
        return !($response['errors'] ?? false);
    }
}
```

## Real-time Monitoring

### Log Monitoring Hook

```php
<?php
/**
 * Monitor for suspicious activity
 */
add_hook('AfterModuleCreate', 1, function($vars) {
    $log = new LogManager();
    
    $log->info('Service created', [
        'service_id' => $vars['serviceid'],
        'domain' => $vars['params']['domain'],
        'user_id' => $vars['params']['userid'],
    ]);
    
    // Check for suspicious patterns
    $recentCreations = Capsule::table('mod_activity_log')
        ->where('user_id', $vars['params']['userid'])
        ->where('activity_type', 'module_create')
        ->where('created_at', '>', date('Y-m-d H:i:s', strtotime('-1 hour')))
        ->count();
    
    if ($recentCreations > 10) {
        $log->warning('High volume of service creations', [
            'user_id' => $vars['params']['userid'],
            'count' => $recentCreations,
        ]);
        
        // Alert admin
        sendAdminNotification([
            'subject' => 'Suspicious Activity Detected',
            'message' => "User #{$vars['params']['userid']} created {$recentCreations} services in the last hour.",
        ]);
    }
});
```

## Log Analysis

```php
<?php
/**
 * Generate log report
 */
function generateLogReport(int $days = 7): array
{
    $cutoff = date('Y-m-d H:i:s', strtotime("-{$days} days"));
    
    return [
        'errors' => Capsule::table('mod_activity_log')
            ->where('activity_type', 'error')
            ->where('created_at', '>', $cutoff)
            ->count(),
        'warnings' => Capsule::table('mod_activity_log')
            ->where('activity_type', 'warning')
            ->where('created_at', '>', $cutoff)
            ->count(),
        'page_views' => Capsule::table('mod_activity_log')
            ->where('activity_type', 'page_view')
            ->where('created_at', '>', $cutoff)
            ->count(),
        'api_calls' => Capsule::table('mod_activity_log')
            ->where('activity_type', 'api_call')
            ->where('created_at', '>', $cutoff)
            ->count(),
        'top_pages' => ActivityTracker::getPopularPages($days),
    ];
}
```

## Best Practices

1. **Structured logging** - Use JSON format for easy parsing
2. **Log rotation** - Prevent disk space issues
3. **Log levels** - Use appropriate severity levels
4. **Sensitive data** - Never log passwords or tokens
5. **Performance** - Use async logging for high traffic
6. **Centralized logging** - Aggregate logs for analysis

## Related Documentation

- [whmcs-integration-api.md](whmcs-integration-api.md)
- [whmcs-integration-monitoring.md](whmcs-integration-monitoring.md)
