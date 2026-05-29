# WHMCS Async Task Processing Workflow

## Overview
This workflow implements asynchronous task processing for WHMCS to handle long-running operations without blocking the main request.

## Prerequisites
- WHMCS with custom hooks capability
- PHP 7.4+ with pcntl extension for process management
- Understanding of async programming patterns

## Step-by-Step Process

### Step 1: Understand Async Processing Architecture
```
SYNCHRONOUS:
Request → Processing → Response (blocked)

ASYNCHRONOUS:
Request → Task Queued → Response (immediate)
                     ↓
         Background Worker → Task Execution → Result Storage
```

### Step 2: Create Async Task Manager
```php
<?php
// /includes/async/AsyncTaskManager.php

namespace WHMCS\Async;

class AsyncTaskManager
{
    private $storagePath;
    private $maxConcurrent = 5;

    public function __construct($storagePath = null)
    {
        $this->storagePath = $storagePath ?? __DIR__ . '/tasks';
        $this->ensureStorageDir();
    }

    private function ensureStorageDir()
    {
        if (!is_dir($this->storagePath)) {
            mkdir($this->storagePath, 0755, true);
        }
    }

    /**
     * Queue an async task
     */
    public function dispatch(string $taskType, array $data = [], array $options = []): string
    {
        $taskId = $this->generateTaskId();

        $task = [
            'id' => $taskId,
            'type' => $taskType,
            'data' => $data,
            'status' => 'pending',
            'priority' => $options['priority'] ?? 'normal',
            'created_at' => date('c'),
            'started_at' => null,
            'completed_at' => null,
            'result' => null,
            'error' => null,
            'progress' => 0
        ];

        $this->saveTask($task);

        // Trigger background processing
        $this->triggerBackgroundExecution($taskId);

        return $taskId;
    }

    /**
     * Dispatch multiple tasks
     */
    public function dispatchMany(array $tasks): array
    {
        $taskIds = [];

        foreach ($tasks as $task) {
            $taskIds[] = $this->dispatch(
                $task['type'],
                $task['data'] ?? [],
                $task['options'] ?? []
            );
        }

        return $taskIds;
    }

    /**
     * Get task status
     */
    public function getStatus(string $taskId): ?array
    {
        $task = $this->loadTask($taskId);

        if (!$task) {
            return null;
        }

        return [
            'id' => $task['id'],
            'status' => $task['status'],
            'progress' => $task['progress'],
            'created_at' => $task['created_at'],
            'started_at' => $task['started_at'],
            'completed_at' => $task['completed_at'],
            'result' => $task['result'],
            'error' => $task['error']
        ];
    }

    /**
     * Wait for task completion
     */
    public function wait(string $taskId, int $timeout = 300): ?array
    {
        $start = time();

        while (time() - $start < $timeout) {
            $status = $this->getStatus($taskId);

            if ($status['status'] === 'completed' || $status['status'] === 'failed') {
                return $status;
            }

            usleep(100000); // 100ms
        }

        return $this->getStatus($taskId);
    }

    /**
     * Update task progress
     */
    public function updateProgress(string $taskId, int $progress)
    {
        $task = $this->loadTask($taskId);

        if ($task) {
            $task['progress'] = min(100, max(0, $progress));
            $this->saveTask($task);
        }
    }

    /**
     * Mark task as complete
     */
    public function complete(string $taskId, $result)
    {
        $task = $this->loadTask($taskId);

        if ($task) {
            $task['status'] = 'completed';
            $task['completed_at'] = date('c');
            $task['result'] = $result;
            $task['progress'] = 100;
            $this->saveTask($task);
        }
    }

    /**
     * Mark task as failed
     */
    public function fail(string $taskId, string $error)
    {
        $task = $this->loadTask($taskId);

        if ($task) {
            $task['status'] = 'failed';
            $task['completed_at'] = date('c');
            $task['error'] = $error;
            $this->saveTask($task);
        }
    }

    private function generateTaskId(): string
    {
        return uniqid('task_', true);
    }

    private function saveTask(array $task)
    {
        file_put_contents(
            $this->storagePath . '/' . $task['id'] . '.json',
            json_encode($task, JSON_PRETTY_PRINT)
        );
    }

    private function loadTask(string $taskId): ?array
    {
        $path = $this->storagePath . '/' . $taskId . '.json';

        if (!file_exists($path)) {
            return null;
        }

        return json_decode(file_get_contents($path), true);
    }

    private function triggerBackgroundExecution(string $taskId)
    {
        // Fork process for background execution
        if (function_exists('pcntl_fork')) {
            $pid = pcntl_fork();

            if ($pid === -1) {
                // Fork failed, execute synchronously
                $this->executeTask($taskId);
            } elseif ($pid === 0) {
                // Child process
                $this->executeTask($taskId);
                exit(0);
            }
            // Parent continues immediately
        } else {
            // No forking, execute synchronously but non-blocking
            $this->executeTaskAsync($taskId);
        }
    }

    private function executeTaskAsync(string $taskId)
    {
        // Schedule via cron or background script
        $task = $this->loadTask($taskId);

        if ($task && $task['status'] === 'pending') {
            $task['status'] = 'queued';
            $this->saveTask($task);
        }
    }
}
```

### Step 3: Create Task Handlers
```php
<?php
// /includes/async/handlers/TaskHandlers.php

namespace WHMCS\Async\Handlers;

class TaskHandlers
{
    /**
     * Handle bulk provisioning task
     */
    public static function handleBulkProvisioning(array $data): array
    {
        $serviceIds = $data['service_ids'];
        $results = [
            'success' => 0,
            'failed' => 0,
            'errors' => []
        ];

        $total = count($serviceIds);
        $processed = 0;

        foreach ($serviceIds as $serviceId) {
            try {
                $result = self::provisionService($serviceId);

                if ($result['success']) {
                    $results['success']++;
                } else {
                    $results['failed']++;
                    $results['errors'][] = [
                        'service_id' => $serviceId,
                        'error' => $result['error']
                    ];
                }
            } catch (Exception $e) {
                $results['failed']++;
                $results['errors'][] = [
                    'service_id' => $serviceId,
                    'error' => $e->getMessage()
                ];
            }

            $processed++;
            $progress = round(($processed / $total) * 100);

            // Update progress callback
            if (isset($data['progress_callback'])) {
                call_user_func($data['progress_callback'], $progress);
            }
        }

        return $results;
    }

    /**
     * Handle bulk email task
     */
    public static function handleBulkEmail(array $data): array
    {
        $recipients = $data['recipients'];
        $template = $data['template'];
        $vars = $data['vars'] ?? [];
        $batchSize = $data['batch_size'] ?? 50;

        $results = [
            'sent' => 0,
            'failed' => 0,
            'skipped' => 0
        ];

        $chunks = array_chunk($recipients, $batchSize);

        foreach ($chunks as $batch) {
            foreach ($batch as $recipient) {
                try {
                    // Check if recipient should receive email
                    if (self::shouldSendEmail($recipient, $template)) {
                        sendEmail($recipient['id'], $template, array_merge($vars, $recipient));
                        $results['sent']++;
                    } else {
                        $results['skipped']++;
                    }
                } catch (Exception $e) {
                    $results['failed']++;
                    logActivity("Bulk email failed for {$recipient['id']}: " . $e->getMessage());
                }

                // Rate limiting
                usleep(10000); // 10ms between emails
            }

            // Pause between batches
            sleep(1);
        }

        return $results;
    }

    /**
     * Handle data export task
     */
    public static function handleDataExport(array $data): array
    {
        $exportType = $data['type'];
        $filters = $data['filters'] ?? [];
        $format = $data['format'] ?? 'csv';

        $exportDir = __DIR__ . '/../../exports';
        if (!is_dir($exportDir)) {
            mkdir($exportDir, 0755, true);
        }

        $filename = $exportType . '_' . date('Y-m-d_His') . '.' . $format;
        $fullPath = $exportDir . '/' . $filename;

        // Fetch data based on type
        $data = self::fetchExportData($exportType, $filters);

        // Export to file
        $rows = 0;
        $handle = fopen($fullPath, 'w');

        foreach ($data as $row) {
            fputcsv($handle, (array)$row);
            $rows++;

            if ($rows % 1000 === 0) {
                // Update progress every 1000 rows
            }
        }

        fclose($handle);

        return [
            'file' => $filename,
            'path' => $fullPath,
            'rows' => $rows,
            'size' => filesize($fullPath)
        ];
    }

    /**
     * Handle external sync task
     */
    public static function handleExternalSync(array $data): array
    {
        $syncType = $data['sync_type']; // 'crm', 'accounting', 'support'
        $batchSize = $data['batch_size'] ?? 100;
        $since = $data['since'] ?? null;

        $results = [
            'synced' => 0,
            'created' => 0,
            'updated' => 0,
            'failed' => 0,
            'errors' => []
        ];

        // Fetch pending sync items
        $items = self::getSyncItems($syncType, $since, $batchSize);

        foreach ($items as $item) {
            try {
                $result = self::syncItem($syncType, $item);

                $results['synced']++;

                if ($result['action'] === 'create') {
                    $results['created']++;
                } else {
                    $results['updated']++;
                }
            } catch (Exception $e) {
                $results['failed']++;
                $results['errors'][] = [
                    'item_id' => $item['id'],
                    'error' => $e->getMessage()
                ];
            }
        }

        return $results;
    }

    // Helper methods
    private static function provisionService(int $serviceId): array
    {
        $service = Capsule::table('tblhosting')
            ->where('id', $serviceId)
            ->first();

        if (!$service) {
            return ['success' => false, 'error' => 'Service not found'];
        }

        // Provision logic
        $result = runServerModuleFunction(
            'CreateAccount',
            $service->servertype,
            $service->server,
            ['serviceid' => $serviceId]
        );

        if ($result['success']) {
            Capsule::table('tblhosting')
                ->where('id', $serviceId)
                ->update(['domainstatus' => 'Active']);
        }

        return $result;
    }

    private static function shouldSendEmail(array $recipient, string $template): bool
    {
        // Check email preferences
        return true;
    }

    private static function fetchExportData(string $type, array $filters): \Illuminate\Support\Collection
    {
        return match ($type) {
            'clients' => Capsule::table('tblclients')->get(),
            'services' => Capsule::table('tblhosting')->get(),
            'invoices' => Capsule::table('tblinvoices')->get(),
            'domains' => Capsule::table('tbldomains')->get(),
            default => collect([])
        };
    }

    private static function getSyncItems(string $syncType, ?string $since, int $limit): array
    {
        // Implementation depends on sync type
        return [];
    }

    private static function syncItem(string $syncType, array $item): array
    {
        // Route to appropriate sync handler
        return match ($syncType) {
            'crm' => self::syncToCRM($item),
            'accounting' => self::syncToAccounting($item),
            default => ['action' => 'unknown', 'success' => false]
        };
    }

    private static function syncToCRM(array $item): array
    {
        // CRM sync implementation
        return ['action' => 'updated', 'success' => true];
    }

    private static function syncToAccounting(array $item): array
    {
        // Accounting sync implementation
        return ['action' => 'created', 'success' => true];
    }
}
```

### Step 4: Create Async Task Worker
```php
<?php
// /includes/async/worker.php

require_once __DIR__ . '/../../../init.php';

use WHMCS\Async\AsyncTaskManager;
use WHMCS\Async\Handlers\TaskHandlers;

class AsyncWorker
{
    private $storagePath;
    private $maxConcurrent = 3;
    private $running = true;

    public function __construct($storagePath = null)
    {
        $this->storagePath = $storagePath ?? __DIR__ . '/tasks';
    }

    public function run()
    {
        // Setup signal handlers for graceful shutdown
        if (function_exists('pcntl_signal')) {
            pcntl_signal(SIGTERM, [$this, 'shutdown']);
            pcntl_signal(SIGINT, [$this, 'shutdown']);
        }

        logActivity("Async worker started");

        while ($this->running) {
            // Find pending tasks
            $tasks = $this->getPendingTasks();

            if (empty($tasks)) {
                sleep(1);
                continue;
            }

            // Process tasks
            foreach ($tasks as $task) {
                if (!$this->running) break;

                $this->processTask($task);
            }

            // Check signals
            if (function_exists('pcntl_signal_dispatch')) {
                pcntl_signal_dispatch();
            }
        }

        logActivity("Async worker stopped");
    }

    private function getPendingTasks(): array
    {
        $tasks = [];

        foreach (glob($this->storagePath . '/task_*.json') as $file) {
            $task = json_decode(file_get_contents($file), true);

            if ($task['status'] === 'queued' || $task['status'] === 'pending') {
                $tasks[] = $task;
            }
        }

        return $tasks;
    }

    private function processTask(array $task)
    {
        $taskId = $task['id'];
        $taskType = $task['type'];
        $data = $task['data'];

        logActivity("Processing async task: {$taskType} ({$taskId})");

        // Update status
        $this->updateTaskStatus($taskId, 'running');

        try {
            // Route to handler
            $result = $this->executeTask($taskType, $data);

            // Mark complete
            $this->completeTask($taskId, $result);

            logActivity("Async task completed: {$taskType} ({$taskId})");
        } catch (Exception $e) {
            logActivity("Async task failed: {$taskType} ({$taskId}) - " . $e->getMessage());
            $this->failTask($taskId, $e->getMessage());
        }
    }

    private function executeTask(string $type, array $data)
    {
        $handlerMap = [
            'bulk_provisioning' => [TaskHandlers::class, 'handleBulkProvisioning'],
            'bulk_email' => [TaskHandlers::class, 'handleBulkEmail'],
            'data_export' => [TaskHandlers::class, 'handleDataExport'],
            'external_sync' => [TaskHandlers::class, 'handleExternalSync']
        ];

        if (isset($handlerMap[$type])) {
            return call_user_func($handlerMap[$type], $data);
        }

        throw new Exception("Unknown task type: {$type}");
    }

    private function updateTaskStatus(string $taskId, string $status)
    {
        $file = $this->storagePath . '/' . $taskId . '.json';
        $task = json_decode(file_get_contents($file), true);

        $task['status'] = $status;
        $task['started_at'] = date('c');

        file_put_contents($file, json_encode($task, JSON_PRETTY_PRINT));
    }

    private function completeTask(string $taskId, $result)
    {
        $file = $this->storagePath . '/' . $taskId . '.json';
        $task = json_decode(file_get_contents($file), true);

        $task['status'] = 'completed';
        $task['completed_at'] = date('c');
        $task['result'] = $result;
        $task['progress'] = 100;

        file_put_contents($file, json_encode($task, JSON_PRETTY_PRINT));
    }

    private function failTask(string $taskId, string $error)
    {
        $file = $this->storagePath . '/' . $taskId . '.json';
        $task = json_decode(file_get_contents($file), true);

        $task['status'] = 'failed';
        $task['completed_at'] = date('c');
        $task['error'] = $error;

        file_put_contents($file, json_encode($task, JSON_PRETTY_PRINT));
    }

    public function shutdown()
    {
        $this->running = false;
        logActivity("Async worker shutdown signal received");
    }
}

// Run worker
$worker = new AsyncWorker();
$worker->run();
```

### Step 5: Create Hooks for Async Processing
```php
<?php
// /includes/hooks/async_hooks.php

use WHMCS\Async\AsyncTaskManager;

add_hook('AfterServiceCreate', 1, function($vars) {
    $async = new AsyncTaskManager();

    // Queue provisioning as async task
    $taskId = $async->dispatch('bulk_provisioning', [
        'service_ids' => [$vars['serviceid']],
        'priority' => 'high'
    ]);

    logActivity("Service provisioning queued: {$taskId}");
});

add_hook('ClientAdd', 1, function($vars) {
    $async = new AsyncTaskManager();

    // Queue CRM sync
    $async->dispatch('external_sync', [
        'sync_type' => 'crm',
        'items' => [['type' => 'client', 'id' => $vars['userid']]]
    ]);
});
```

### Step 6: Monitor Async Tasks
```php
/**
 * Get async task statistics
 */
function getAsyncTaskStats()
{
    $storagePath = __DIR__ . '/../async/tasks';
    $stats = [
        'pending' => 0,
        'queued' => 0,
        'running' => 0,
        'completed' => 0,
        'failed' => 0
    ];

    foreach (glob($storagePath . '/task_*.json') as $file) {
        $task = json_decode(file_get_contents($file), true);
        $status = $task['status'];

        if (isset($stats[$status])) {
            $stats[$status]++;
        }
    }

    return $stats;
}
```

## Async Task Best Practices

1. **Idempotency** - Tasks should be safe to re-run
2. **Progress Tracking** - Update progress for long tasks
3. **Error Handling** - Always catch exceptions
4. **Cleanup** - Remove completed tasks after retention period
5. **Resource Limits** - Set concurrent task limits
6. **Monitoring** - Track task durations and failures

## Task Status Lifecycle
```
pending → queued → running → completed
                      ↓
                   failed
```

## Related Workflows
- [WHMCS Queue Processing](./whmcs-queue-processing.md)
- [WHMCS Batch Operations](./whmcs-batch-operations.md)
- [WHMCS Cron Automation](./whmcs-cron-automation.md)