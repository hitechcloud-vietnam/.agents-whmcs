# WHMCS Logging Patterns Skill
# Version: 1.0 | Updated: 2026-05-29

## Purpose

Guide for implementing comprehensive logging patterns in WHMCS modules and customizations, including activity logging, transaction logging, and debug logging.

## When to Use

- Implementing logging for modules
- Debugging issues in WHMCS
- Auditing system activities
- Monitoring module performance
- Compliance and security logging

## Logging Overview

### 1. WHMCS Core Logging Functions

```php
<?php
// Activity Log - For user/system actions
logActivity($message, $clientId = null);

// Transaction Log - For payment transactions
logTransaction($gateway, $data, $status, $transactionId = null);

// Module Log - For provisioning module debugging
logModuleCall($moduleName, $action, $request, $response, $data, $processed);
```

### 2. Basic Logging Patterns

```php
<?php
// Simple activity logging
logActivity("Custom module action completed");

// With user context
logActivity("User updated their profile", $userId);

// Transaction logging
logTransaction('MyGateway', [
    'amount' => 29.99,
    'currency' => 'USD',
    'customer_email' => 'customer@example.com',
], 'Success', 'TXN-12345');

// Module function logging
logModuleCall(
    'MyModule',
    'CreateAccount',
    $params,
    $result,
    '',
    true
);
```

## Advanced Logging Patterns

### 1. Structured Logger Class

```php
<?php
// includes/classes/Logger.php

namespace MyModule\Logger;

use WHMCS\Database\Capsule;

class Logger {
    private string $module;
    private array $config;

    public function __construct(string $module, array $config = []) {
        $this->module = $module;
        $this->config = array_merge([
            'log_to_db' => true,
            'log_to_file' => false,
            'log_level' => 'info',
            'log_file_path' => __DIR__ . '/../../logs/' . $module . '.log',
        ], $config);
    }

    public function info(string $message, array $context = []): void {
        $this->log('info', $message, $context);
    }

    public function debug(string $message, array $context = []): void {
        if ($this->config['log_level'] !== 'debug') {
            return;
        }
        $this->log('debug', $message, $context);
    }

    public function warning(string $message, array $context = []): void {
        $this->log('warning', $message, $context);
    }

    public function error(string $message, array $context = []): void {
        $this->log('error', $message, $context);
    }

    public function critical(string $message, array $context = []): void {
        $this->log('critical', $message, $context);
    }

    private function log(string $level, string $message, array $context): void {
        $entry = [
            'timestamp' => date('Y-m-d H:i:s'),
            'level' => strtoupper($level),
            'module' => $this->module,
            'message' => $message,
            'context' => $context,
            'ip' => $_SERVER['REMOTE_ADDR'] ?? 'unknown',
            'user_id' => $this->getCurrentUserId(),
        ];

        // Remove sensitive data
        $entry = $this->sanitizeEntry($entry);

        if ($this->config['log_to_db']) {
            $this->logToDatabase($entry);
        }

        if ($this->config['log_to_file']) {
            $this->logToFile($entry);
        }

        // Always log to WHMCS activity log for critical items
        if (in_array($level, ['error', 'critical'])) {
            logActivity("[{$this->module}] {$level}: {$message}");
        }
    }

    private function logToDatabase(array $entry): void {
        try {
            Capsule::table('mod_' . $this->module . '_logs')->insert([
                'level' => $entry['level'],
                'message' => $entry['message'],
                'context' => json_encode($entry['context']),
                'ip_address' => $entry['ip'],
                'user_id' => $entry['user_id'],
                'created_at' => $entry['timestamp'],
            ]);
        } catch (\Exception $e) {
            // Fallback to file logging if DB fails
            $this->logToFile(['error' => 'Database logging failed', 'original' => $entry]);
        }
    }

    private function logToFile(array $entry): void {
        $directory = dirname($this->config['log_file_path']);
        if (!is_dir($directory)) {
            mkdir($directory, 0755, true);
        }

        $line = json_encode($entry) . PHP_EOL;
        file_put_contents($this->config['log_file_path'], $line, FILE_APPEND | LOCK_EX);
    }

    private function sanitizeEntry(array $entry): array {
        $sensitiveKeys = [
            'password', 'passwd', 'secret', 'token', 'api_key', 'apikey',
            'credit_card', 'card_number', 'cvv', 'ssn', 'tax_id',
        ];

        array_walk_recursive($entry['context'], function (&$value, $key) use ($sensitiveKeys) {
            foreach ($sensitiveKeys as $sensitiveKey) {
                if (stripos($key, $sensitiveKey) !== false) {
                    $value = '***REDACTED***';
                    break;
                }
            }
        });

        return $entry;
    }

    private function getCurrentUserId(): ?int {
        if (isset($_SESSION['uid'])) {
            return (int) $_SESSION['uid'];
        }
        if (isset($_SESSION['adminid'])) {
            return -1 * (int) $_SESSION['adminid']; // Negative for admin
        }
        return null;
    }
}
```

### 2. API Call Logging

```php
<?php
// includes/classes/ApiLogger.php

namespace MyModule\Logger;

use WHMCS\Database\Capsule;

class ApiLogger {
    private string $tableName = 'mod_api_logs';

    public function logRequest(array $request): int {
        return Capsule::table($this->tableName)->insertGetId([
            'method' => $request['method'] ?? 'UNKNOWN',
            'endpoint' => $request['endpoint'] ?? '',
            'request_body' => $this->sanitize(json_encode($request)),
            'request_headers' => $this->sanitize(json_encode($request['headers'] ?? [])),
            'ip_address' => $_SERVER['REMOTE_ADDR'] ?? '',
            'user_agent' => $_SERVER['HTTP_USER_AGENT'] ?? '',
            'created_at' => date('Y-m-d H:i:s'),
            'status' => 'pending',
        ]);
    }

    public function logResponse(int $logId, array $response, int $httpCode, float $duration): void {
        Capsule::table($this->tableName)
            ->where('id', $logId)
            ->update([
                'response_body' => $this->sanitize(json_encode($response)),
                'http_code' => $httpCode,
                'duration_ms' => $duration,
                'status' => $httpCode >= 400 ? 'error' : 'success',
                'completed_at' => date('Y-m-d H:i:s'),
            ]);
    }

    public function logError(int $logId, string $error): void {
        Capsule::table($this->tableName)
            ->where('id', $logId)
            ->update([
                'error' => $error,
                'status' => 'error',
                'completed_at' => date('Y-m-d H:i:s'),
            ]);
    }

    public function getRecentErrors(int $limit = 100): array {
        return Capsule::table($this->tableName)
            ->where('status', 'error')
            ->orderBy('created_at', 'DESC')
            ->limit($limit)
            ->get()
            ->toArray();
    }

    public function getSlowRequests(int $thresholdMs = 1000, int $limit = 50): array {
        return Capsule::table($this->tableName)
            ->where('duration_ms', '>', $thresholdMs)
            ->orderBy('duration_ms', 'DESC')
            ->limit($limit)
            ->get()
            ->toArray();
    }

    private function sanitize(string $data): string {
        // Remove sensitive patterns
        return preg_replace(
            '/("password":"[^"]*"|"api_key":"[^"]*"|"token":"[^"]*")/i',
            '"***REDACTED***"',
            $data
        );
    }
}

// Usage in API client
class ApiClient {
    private ApiLogger $logger;

    public function request(string $method, string $endpoint, array $data = []): array {
        $startTime = microtime(true);

        $logId = $this->logger->logRequest([
            'method' => $method,
            'endpoint' => $endpoint,
            'data' => $data,
        ]);

        try {
            $ch = curl_init();
            // ... make request ...
            $response = curl_exec($ch);
            $httpCode = curl_getinfo($ch, CURLINFO_HTTP_CODE);
            curl_close($ch);

            $duration = (microtime(true) - $startTime) * 1000;
            $result = json_decode($response, true);

            $this->logger->logResponse($logId, $result, $httpCode, $duration);

            return $result;

        } catch (\Exception $e) {
            $this->logger->logError($logId, $e->getMessage());
            throw $e;
        }
    }
}
```

### 3. Audit Logger for Compliance

```php
<?php
// includes/classes/AuditLogger.php

namespace MyModule\Logger;

use WHMCS\Database\Capsule;

class AuditLogger {
    private string $tableName = 'mod_audit_log';

    public function log(string $action, array $details, ?int $userId = null): void {
        $userId = $userId ?? $this->getCurrentUserId();

        Capsule::table($this->tableName)->insert([
            'user_id' => $userId ?? 0,
            'user_type' => $this->getUserType(),
            'action' => $action,
            'entity_type' => $details['entity_type'] ?? null,
            'entity_id' => $details['entity_id'] ?? null,
            'old_values' => isset($details['old_values']) ? json_encode($details['old_values']) : null,
            'new_values' => isset($details['new_values']) ? json_encode($details['new_values']) : null,
            'ip_address' => $_SERVER['REMOTE_ADDR'] ?? '',
            'user_agent' => $_SERVER['HTTP_USER_AGENT'] ?? '',
            'created_at' => date('Y-m-d H:i:s'),
        ]);
    }

    public function logAccess(string $entityType, int $entityId, string $action = 'view'): void {
        $this->log('access', [
            'entity_type' => $entityType,
            'entity_id' => $entityId,
            'access_type' => $action,
        ]);
    }

    public function logChange(string $entityType, int $entityId, array $oldValues, array $newValues): void {
        $this->log('update', [
            'entity_type' => $entityType,
            'entity_id' => $entityId,
            'old_values' => $oldValues,
            'new_values' => $newValues,
        ]);
    }

    public function logDeletion(string $entityType, int $entityId, array $deletedData): void {
        $this->log('delete', [
            'entity_type' => $entityType,
            'entity_id' => $entityId,
            'old_values' => $deletedData,
        ]);
    }

    public function logLogin(int $userId, bool $success, string $reason = ''): void {
        $this->log($success ? 'login_success' : 'login_failed', [
            'reason' => $reason,
        ], $success ? $userId : null);
    }

    public function logPermissionCheck(int $userId, string $permission, bool $granted): void {
        $this->log('permission_check', [
            'permission' => $permission,
            'granted' => $granted,
        ], $userId);
    }

    public function getAuditTrail(string $entityType, int $entityId, int $limit = 100): array {
        return Capsule::table($this->tableName)
            ->where('entity_type', $entityType)
            ->where('entity_id', $entityId)
            ->orderBy('created_at', 'DESC')
            ->limit($limit)
            ->get()
            ->toArray();
    }

    public function getUserActivity(int $userId, \DateTime $from, \DateTime $to): array {
        return Capsule::table($this->tableName)
            ->where('user_id', $userId)
            ->whereBetween('created_at', [$from->format('Y-m-d H:i:s'), $to->format('Y-m-d H:i:s')])
            ->orderBy('created_at', 'DESC')
            ->get()
            ->toArray();
    }

    private function getCurrentUserId(): ?int {
        if (isset($_SESSION['uid'])) {
            return (int) $_SESSION['uid'];
        }
        if (isset($_SESSION['adminid'])) {
            return (int) $_SESSION['adminid'];
        }
        return null;
    }

    private function getUserType(): string {
        if (isset($_SESSION['adminid'])) {
            return 'admin';
        }
        if (isset($_SESSION['uid'])) {
            return 'client';
        }
        return 'system';
    }
}
```

### 4. Performance Logger

```php
<?php
// includes/classes/PerformanceLogger.php

namespace MyModule\Logger;

use WHMCS\Database\Capsule;

class PerformanceLogger {
    private string $tableName = 'mod_performance_logs';
    private array $markers = [];

    public function start(string $label): void {
        $this->markers[$label] = microtime(true);
    }

    public function end(string $label): float {
        if (!isset($this->markers[$label])) {
            throw new \Exception("No start marker found for: {$label}");
        }

        $duration = (microtime(true) - $this->markers[$label]) * 1000;
        unset($this->markers[$label]);

        return $duration;
    }

    public function logOperation(string $operation, float $durationMs, array $metadata = []): void {
        Capsule::table($this->tableName)->insert([
            'operation' => $operation,
            'duration_ms' => $durationMs,
            'memory_usage' => memory_get_usage(true),
            'peak_memory' => memory_get_peak_usage(true),
            'metadata' => json_encode($metadata),
            'created_at' => date('Y-m-d H:i:s'),
        ]);
    }

    public function measure(string $operation, callable $callback, array $metadata = []): mixed {
        $startTime = microtime(true);
        $startMemory = memory_get_usage(true);

        try {
            $result = $callback();

            $duration = (microtime(true) - $startTime) * 1000;
            $memoryUsed = memory_get_usage(true) - $startMemory;

            $this->logOperation($operation, $duration, array_merge($metadata, [
                'memory_delta' => $memoryUsed,
                'success' => true,
            ]));

            return $result;

        } catch (\Exception $e) {
            $duration = (microtime(true) - $startTime) * 1000;

            $this->logOperation($operation, $duration, array_merge($metadata, [
                'error' => $e->getMessage(),
                'success' => false,
            ]));

            throw $e;
        }
    }

    public function getSlowOperations(int $thresholdMs = 100, int $limit = 50): array {
        return Capsule::table($this->tableName)
            ->where('duration_ms', '>', $thresholdMs)
            ->orderBy('duration_ms', 'DESC')
            ->limit($limit)
            ->get()
            ->toArray();
    }

    public function getOperationStats(string $operation, int $hours = 24): array {
        $since = date('Y-m-d H:i:s', strtotime("-{$hours} hours"));

        $stats = Capsule::table($this->tableName)
            ->where('operation', $operation)
            ->where('created_at', '>=', $since)
            ->selectRaw('
                COUNT(*) as count,
                AVG(duration_ms) as avg_duration,
                MIN(duration_ms) as min_duration,
                MAX(duration_ms) as max_duration,
                SUM(duration_ms) as total_duration
            ')
            ->first();

        return (array) $stats;
    }
}
```

## Log Table Schema

```php
<?php
// Migration for logging tables

use WHMCS\Database\Capsule;

// Module logs table
Capsule::schema()->create('mod_mymodule_logs', function($t) {
    $t->increments('id');
    $t->string('level', 20);
    $t->text('message');
    $t->longText('context');
    $t->string('ip_address', 45);
    $t->integer('user_id')->nullable();
    $t->timestamp('created_at');

    $t->index('level');
    $t->index('created_at');
    $t->index('user_id');
});

// API logs table
Capsule::schema()->create('mod_api_logs', function($t) {
    $t->increments('id');
    $t->string('method', 10);
    $t->string('endpoint', 255);
    $t->longText('request_body');
    $t->longText('request_headers');
    $t->longText('response_body');
    $t->string('http_code', 3);
    $t->string('duration_ms', 10);
    $t->text('error')->nullable();
    $t->string('status', 20);
    $t->string('ip_address', 45);
    $t->string('user_agent', 500);
    $t->timestamp('created_at');
    $t->timestamp('completed_at')->nullable();

    $t->index('status');
    $t->index('http_code');
    $t->index('created_at');
});

// Audit log table
Capsule::schema()->create('mod_audit_log', function($t) {
    $t->increments('id');
    $t->integer('user_id');
    $t->string('user_type', 20);
    $t->string('action', 50);
    $t->string('entity_type', 50)->nullable();
    $t->integer('entity_id')->nullable();
    $t->longText('old_values')->nullable();
    $t->longText('new_values')->nullable();
    $t->string('ip_address', 45);
    $t->string('user_agent', 500);
    $t->timestamp('created_at');

    $t->index('user_id');
    $t->index(['entity_type', 'entity_id']);
    $t->index('action');
    $t->index('created_at');
});

// Performance logs table
Capsule::schema()->create('mod_performance_logs', function($t) {
    $t->increments('id');
    $t->string('operation', 100);
    $t->decimal('duration_ms', 10, 2);
    $t->bigInteger('memory_usage');
    $t->bigInteger('peak_memory');
    $t->longText('metadata')->nullable();
    $t->timestamp('created_at');

    $t->index('operation');
    $t->index('duration_ms');
    $t->index('created_at');
});
```

## Logging Best Practices

```php
<?php
// DO: Log meaningful messages
logActivity("Invoice #{$invoiceId} marked as paid via PayPal");
logActivity("Client {$clientId} password changed via admin panel");

// DON'T: Log sensitive information
logActivity("Password changed to: " . $newPassword); // NEVER
logActivity("API Key: " . $apiKey); // NEVER

// DO: Include context
logActivity("Service suspension failed for order #{$orderId}: " . $e->getMessage());

// DO: Use appropriate log levels
logActivity("Module activated successfully"); // Standard
logActivity("CRITICAL: Payment gateway timeout, retrying..."); // Critical

// DO: Log both success and failure
logActivity("Email sent to {$email}");
logActivity("Failed to send email to {$email}: " . $e->getMessage());

// DO: Log API calls in modules
logModuleCall(
    'MyModule',
    'ProvisionServer',
    $params,
    ['server_id' => 123],
    '',
    true
);
```

## Log Rotation & Cleanup

```php
<?php
// includes/hooks/LogRotation.php

add_hook('DailyCronJob', 50, function($vars) {
    rotateLogs();
    cleanupOldLogs();
});

function rotateLogs(): void {
    $logFiles = glob(__DIR__ . '/../../logs/*.log');

    foreach ($logFiles as $file) {
        $size = filesize($file);
        $maxSize = 10 * 1024 * 1024; // 10MB

        if ($size > $maxSize) {
            $rotatedName = $file . '.' . date('Y-m-d-His');
            rename($file, $rotatedName);

            // Compress old log
            if (function_exists('gzencode')) {
                $content = file_get_contents($rotatedName);
                file_put_contents($rotatedName . '.gz', gzencode($content));
                unlink($rotatedName);
            }
        }
    }
}

function cleanupOldLogs(): void {
    $retentionDays = 90;

    use WHMCS\Database\Capsule;

    // Clean database logs
    Capsule::table('mod_mymodule_logs')
        ->where('created_at', '<', date('Y-m-d H:i:s', strtotime("-{$retentionDays} days")))
        ->delete();

    // Clean old compressed logs
    $oldLogs = glob(__DIR__ . '/../../logs/*.log.gz');
    foreach ($oldLogs as $file) {
        if (filemtime($file) < strtotime("-{$retentionDays} days")) {
            unlink($file);
        }
    }
}
```

## Checklist

- [ ] Logger class implemented with proper levels
- [ ] Sensitive data redaction in place
- [ ] Log rotation configured
- [ ] Log retention policy set
- [ ] Performance logging for critical operations
- [ ] Audit logging for compliance
- [ ] Error logging with stack traces
- [ ] Log queries indexed properly

---

**Related Skills:**
- whmcs-logging
- whmcs-error-handling
- whmcs-monitoring
- whmcs-debugging
- whmcs-security-hardening

**Reference:**
- WHMCS Logging: https://developers.whmcs.com/advanced/logging/