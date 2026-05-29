# WHMCS Timeout Handling Workflow

## Overview
This workflow implements comprehensive timeout handling for WHMCS operations.

## Prerequisites
- WHMCS with custom hooks capability
- PHP configuration access
- Understanding of timeout patterns

## Step-by-Step Process

### Step 1: Configure PHP Timeout Settings
```php
<?php
// /includes/timeout/TimeoutConfig.php

class TimeoutConfig
{
    // Set in php.ini or .htaccess:
    // max_execution_time = 300
    // max_input_time = 300
    // default_socket_timeout = 60

    public static function configure(): array
    {
        return [
            'php' => [
                'max_execution_time' => 300,
                'max_input_time' => 300,
                'memory_limit' => '256M'
            ],
            'whmcs' => [
                'API Timeout' => 60,
                'Module Timeout' => 300,
                'Cron Timeout' => 600,
                'Webhook Timeout' => 30
            ]
        ];
    }

    public static function setTimeout(string $type): bool
    {
        $timeouts = [
            'short' => 30,
            'medium' => 60,
            'long' => 300,
            'extended' => 600
        ];

        if (isset($timeouts[$type])) {
            set_time_limit($timeouts[$type]);
            return true;
        }

        return false;
    }
}
```

### Step 2: Create Timeout Wrapper
```php
<?php
// /includes/timeout/TimeoutWrapper.php

namespace WHMCS\Timeout;

class TimeoutWrapper
{
    private $timeout;
    private $startTime;

    public function __construct(int $timeout = 60)
    {
        $this->timeout = $timeout;
    }

    /**
     * Execute with timeout enforcement
     */
    public function execute(callable $operation, int $timeout = null): Result
    {
        $timeout = $timeout ?? $this->timeout;
        $this->startTime = microtime(true);

        $pcntlEnabled = function_exists('pcntl_signal') && function_exists('pcntl_async_signals');

        if ($pcntlEnabled) {
            return $this->executeWithSignal($operation, $timeout);
        }

        return $this->executeWithMonitoring($operation, $timeout);
    }

    private function executeWithSignal(callable $operation, int $timeout): Result
    {
        // Set alarm signal for timeout
        pcntl_signal(SIGALRM, function() {
            throw new TimeoutException("Operation timed out");
        });

        pcntl_alarm($timeout);

        try {
            $result = $operation();
            pcntl_alarm(0); // Cancel alarm

            return new Result(true, $result, microtime(true) - $this->startTime);
        } catch (TimeoutException $e) {
            pcntl_alarm(0);
            return new Result(false, null, microtime(true) - $this->startTime, $e->getMessage());
        } catch (Exception $e) {
            pcntl_alarm(0);
            throw $e;
        }
    }

    private function executeWithMonitoring(callable $operation, int $timeout): Result
    {
        $startTime = microtime(true);

        while (true) {
            // Check elapsed time
            $elapsed = microtime(true) - $startTime;

            if ($elapsed >= $timeout) {
                return new Result(false, null, $elapsed, "Operation timed out after {$timeout}s");
            }

            // Check if operation is still running (via flag file or queue)
            if ($this->isOperationCancelled()) {
                return new Result(false, null, $elapsed, "Operation cancelled");
            }

            // Execute a chunk of work
            $result = $this->executeChunk($operation);

            if ($result->isComplete()) {
                return new Result(true, $result->getValue(), microtime(true) - $this->startTime);
            }

            // Yield to allow timeout check
            usleep(10000);
        }
    }

    private function executeChunk(callable $operation)
    {
        try {
            return new ChunkResult(true, $operation());
        } catch (Exception $e) {
            return new ChunkResult(false, null, $e);
        }
    }

    private function isOperationCancelled(): bool
    {
        return false; // Override in extended class
    }
}

class Result
{
    private bool $success;
    private $value;
    private float $duration;
    private ?string $error;

    public function __construct(bool $success, $value, float $duration, ?string $error = null)
    {
        $this->success = $success;
        $this->value = $value;
        $this->duration = $duration;
        $this->error = $error;
    }

    public function isSuccess(): bool { return $this->success; }
    public function getValue() { return $this->value; }
    public function getDuration(): float { return $this->duration; }
    public function getError(): ?string { return $this->error; }
}

class ChunkResult
{
    private bool $complete;
    private $value;
    private ?Exception $error;

    public function __construct(bool $complete, $value, ?Exception $error = null)
    {
        $this->complete = $complete;
        $this->value = $value;
        $this->error = $error;
    }

    public function isComplete(): bool { return $this->complete; }
    public function getValue() { return $this->value; }
    public function getError(): ?Exception { return $this->error; }
}

class TimeoutException extends Exception {}
```

### Step 3: Implement Long-Running Task Timeout
```php
<?php
// /includes/timeout/LongRunningTaskHandler.php

namespace WHMCS\Timeout;

class LongRunningTaskHandler
{
    private $timeout = 600; // 10 minutes
    private $checkpointInterval = 30; // seconds

    /**
     * Execute long-running task with progress checkpoints
     */
    public function execute(string $taskId, callable $task, array $options = []): array
    {
        $timeout = $options['timeout'] ?? $this->timeout;
        $taskWrapper = new TimeoutWrapper($timeout);

        // Initialize task state
        $state = $this->initializeTask($taskId, $options);

        return $taskWrapper->execute(function() use ($taskId, $task, $state, $options) {
            $processed = 0;
            $total = $state['total'] ?? 0;

            // Create checkpoint monitor
            $monitor = new TaskProgressMonitor($taskId);
            $monitor->setTimeout($this->checkpointInterval);

            foreach ($this->getTaskItems($taskId) as $item) {
                // Check timeout before processing
                if ($monitor->shouldStop()) {
                    throw new TimeoutException("Checkpoint timeout at item {$processed}");
                }

                // Process item
                $result = $task($item, $state);

                // Update progress
                $processed++;
                $monitor->checkpoint($processed, $total);

                // Save intermediate state
                if ($processed % 100 === 0) {
                    $this->saveProgress($taskId, $processed, $state);
                }
            }

            return [
                'processed' => $processed,
                'state' => $state
            ];
        });
    }

    /**
     * Resume interrupted task
     */
    public function resume(string $taskId, callable $task): array
    {
        $state = $this->loadProgress($taskId);
        $processed = $state['processed'] ?? 0;

        logActivity("Resuming task {$taskId} from position {$processed}");

        return $this->execute($taskId, $task, ['resume_from' => $processed]);
    }

    private function initializeTask(string $taskId, array $options): array
    {
        return [
            'task_id' => $taskId,
            'started_at' => date('Y-m-d H:i:s'),
            'status' => 'running',
            'processed' => 0
        ];
    }

    private function getTaskItems(string $taskId): array
    {
        // Override in implementation
        return [];
    }

    private function saveProgress(string $taskId, int $processed, array $state)
    {
        Capsule::table('mod_long_running_tasks')
            ->where('task_id', $taskId)
            ->update([
                'processed' => $processed,
                'state' => json_encode($state),
                'last_checkpoint' => date('Y-m-d H:i:s')
            ]);
    }

    private function loadProgress(string $taskId): array
    {
        $record = Capsule::table('mod_long_running_tasks')
            ->where('task_id', $taskId)
            ->first();

        return $record ? json_decode($record->state, true) : [];
    }
}

class TaskProgressMonitor
{
    private $taskId;
    private $lastCheckpoint;
    private $timeout;

    public function __construct(string $taskId)
    {
        $this->taskId = $taskId;
    }

    public function setTimeout(int $seconds)
    {
        $this->timeout = $seconds;
    }

    public function checkpoint(int $processed, int $total)
    {
        Capsule::table('mod_task_checkpoints')->insert([
            'task_id' => $this->taskId,
            'processed' => $processed,
            'total' => $total,
            'checkpoint_at' => date('Y-m-d H:i:s')
        ]);

        $this->lastCheckpoint = time();
    }

    public function shouldStop(): bool
    {
        if (!$this->lastCheckpoint) {
            return false;
        }

        return (time() - $this->lastCheckpoint) > $this->timeout;
    }
}
```

### Step 4: Create API Timeout Handler
```php
<?php
// /includes/timeout/ApiTimeoutHandler.php

namespace WHMCS\Timeout;

class ApiTimeoutHandler
{
    private $defaultTimeout = 30;
    private $slowThreshold = 5;

    /**
     * Call external API with timeout
     */
    public function call(string $url, array $options = []): array
    {
        $timeout = $options['timeout'] ?? $this->defaultTimeout;
        $startTime = microtime(true);

        $ch = curl_init();

        curl_setopt_array($ch, [
            CURLOPT_URL => $url,
            CURLOPT_RETURNTRANSFER => true,
            CURLOPT_TIMEOUT => $timeout,
            CURLOPT_CONNECTTIMEOUT => $options['connect_timeout'] ?? 10,
            CURLOPT_NOSIGNAL => 1 // Required for timeout to work
        ]);

        if (isset($options['method'])) {
            curl_setopt($ch, CURLOPT_CUSTOMREQUEST, $options['method']);
        }

        if (isset($options['body'])) {
            curl_setopt($ch, CURLOPT_POSTFIELDS, json_encode($options['body']));
            curl_setopt($ch, CURLOPT_HTTPHEADER, ['Content-Type: application/json']);
        }

        $response = curl_exec($ch);
        $httpCode = curl_getinfo($ch, CURLINFO_HTTP_CODE);
        $error = curl_error($ch);
        $duration = microtime(true) - $startTime;

        curl_close($ch);

        // Log slow API calls
        if ($duration > $this->slowThreshold) {
            logActivity("Slow API call: {$url} took {$duration}s");
        }

        if ($error) {
            if (strpos($error, 'timed out') !== false) {
                throw new ApiTimeoutException("API timeout after {$timeout}s: {$url}");
            }
            throw new ApiException("API error: {$error}");
        }

        return [
            'success' => $httpCode >= 200 && $httpCode < 300,
            'http_code' => $httpCode,
            'response' => json_decode($response, true),
            'duration' => $duration
        ];
    }

    /**
     * Batch call with individual timeouts
     */
    public function batchCall(array $calls, int $totalTimeout = 60): array
    {
        $multiHandle = curl_multi_init();
        $handles = [];
        $results = [];

        $startTime = microtime(true);

        foreach ($calls as $index => $call) {
            $ch = curl_init();

            curl_setopt_array($ch, [
                CURLOPT_URL => $call['url'],
                CURLOPT_RETURNTRANSFER => true,
                CURLOPT_TIMEOUT => $call['timeout'] ?? $this->defaultTimeout,
                CURLOPT_NOSIGNAL => 1
            ]);

            $handles[$index] = $ch;
            curl_multi_add_handle($multiHandle, $ch);
        }

        // Execute all requests
        do {
            $status = curl_multi_exec($multiHandle, $running);

            if ($running) {
                curl_multi_select($multiHandle, 0.1);
            }

            // Check total timeout
            if ((microtime(true) - $startTime) > $totalTimeout) {
                foreach ($handles as $ch) {
                    curl_multi_remove_handle($multiHandle, $ch);
                    curl_close($ch);
                }
                curl_multi_close($multiHandle);
                throw new ApiTimeoutException("Batch timeout after {$totalTimeout}s");
            }
        } while ($status === CURLM_CALL_MULTI_PERFORM || $running);

        // Collect results
        foreach ($handles as $index => $ch) {
            $results[$index] = [
                'response' => curl_multi_getcontent($ch),
                'http_code' => curl_getinfo($ch, CURLINFO_HTTP_CODE),
                'error' => curl_error($ch)
            ];
            curl_multi_remove_handle($multiHandle, $ch);
            curl_close($ch);
        }

        curl_multi_close($multiHandle);

        return $results;
    }
}

class ApiTimeoutException extends Exception {}
class ApiException extends Exception {}
```

### Step 5: Implement Module Timeout Hook
```php
<?php
// /includes/hooks/timeout_hooks.php

use WHMCS\Timeout\TimeoutWrapper;
use WHMCS\Timeout\ApiTimeoutHandler;

add_hook('PreModuleCreate', 1, function($vars) {
    // Set extended timeout for provisioning
    set_time_limit(300);
});

add_hook('AfterModuleCreate', 1, function($vars) {
    // Reset to normal timeout
    set_time_limit(60);
});

// External API calls with timeout
add_hook('InvoicePaid', 1, function($vars) {
    $api = new ApiTimeoutHandler();

    try {
        $result = $api->call('https://api.example.com/webhook', [
            'method' => 'POST',
            'body' => ['invoice_id' => $vars['invoiceid']],
            'timeout' => 10 // Short timeout for webhooks
        ]);
    } catch (ApiTimeoutException $e) {
        // Queue for retry
        queueWebhookRetry($vars['invoiceid']);
    }
});
```

### Step 6: Monitor Timeout Statistics
```php
<?php
// /includes/timeout/TimeoutMonitor.php

class TimeoutMonitor
{
    public static function recordTimeout(string $type, string $operation, float $duration)
    {
        Capsule::table('mod_timeout_log')->insert([
            'type' => $type,
            'operation' => $operation,
            'duration' => $duration,
            'created_at' => date('Y-m-d H:i:s')
        ]);
    }

    public static function getStats(int $days = 7): array
    {
        return Capsule::select("
            SELECT
                type,
                operation,
                COUNT(*) as count,
                AVG(duration) as avg_duration,
                MAX(duration) as max_duration
            FROM mod_timeout_log
            WHERE created_at >= DATE_SUB(NOW(), INTERVAL ? DAY)
            GROUP BY type, operation
        ", [$days]);
    }
}

// Log timeouts
add_hook('TimeoutOccurred', 1, function($vars) {
    TimeoutMonitor::recordTimeout(
        $vars['type'],
        $vars['operation'],
        $vars['duration']
    );
});
```

## Timeout Configuration Guide

| Operation | Recommended Timeout | Reason |
|-----------|-------------------|--------|
| Simple queries | 5-10s | Quick response expected |
| API calls | 10-30s | Depends on service |
| Module calls | 60-300s | Server provisioning can be slow |
| Cron jobs | 5-10 min | Batch operations |
| Exports | 5-10 min | Large data sets |
| Webhooks | 5-30s | Should be fast |

## Best Practices

1. **Set reasonable timeouts** - Not too short (false failures), not too long (blocking)
2. **Use async for long tasks** - Don't block web requests
3. **Implement checkpoints** - Save progress for resumable tasks
4. **Monitor timeout rates** - High timeout = performance issue
5. **Use circuit breakers** - Stop calling failing services
6. **Log slow operations** - Identify optimization opportunities

## Related Workflows
- [WHMCS Retry Logic](./whmcs-retry-logic.md)
- [WHMCS Async Tasks](./whmcs-async-tasks.md)
- [WHMCS Queue Processing](./whmcs-queue-processing.md)