# WHMCS Queue Processing Workflow

## Overview
This workflow implements a robust queue system for WHMCS to handle asynchronous and delayed processing of tasks.

## Prerequisites
- WHMCS installation with database access
- PHP 7.4+ for modern syntax
- Understanding of queue patterns

## Step-by-Step Process

### Step 1: Create Queue Database Schema
```php
<?php
// /includes/hooks/queue_schema.php

use Illuminate\Database\Schema\Blueprint;

function installQueueSchema()
{
    // Create queue jobs table
    if (!Capsule::hasTable('mod_queue_jobs')) {
        Capsule::schema()->create('mod_queue_jobs', function (Blueprint $table) {
            $table->increments('id');
            $table->string('queue', 50)->default('default');
            $table->string('job', 100);
            $table->text('payload');
            $table->unsignedTinyInteger('attempts')->default(0);
            $table->unsignedInteger('max_attempts')->default(3);
            $table->timestamp('available_at')->nullable();
            $table->timestamp('reserved_at')->nullable();
            $table->timestamp('completed_at')->nullable();
            $table->text('error')->nullable();
            $table->timestamps();

            $table->index(['queue', 'status', 'available_at']);
            $table->index('reserved_at');
        });
    }
}

function installQueueTables()
{
    // Jobs table
    Capsule::schema()->create('mod_queue_jobs', function (Blueprint $table) {
        $table->increments('id');
        $table->string('queue', 50)->default('default');
        $table->string('job', 100);
        $table->json('payload');
        $table->unsignedTinyInteger('attempts')->default(0);
        $table->unsignedInteger('max_attempts')->default(3);
        $table->timestamp('available_at')->nullable();
        $table->timestamp('reserved_at')->nullable();
        $table->timestamp('completed_at')->nullable();
        $table->text('error')->nullable();
        $table->timestamps();
    });

    // Failed jobs table
    Capsule::schema()->create('mod_queue_failed_jobs', function (Blueprint $table) {
        $table->increments('id');
        $table->string('queue', 50);
        $table->string('job', 100);
        $table->json('payload');
        $table->text('error');
        $table->timestamp('failed_at');
    });
}
```

### Step 2: Create Queue Manager Class
```php
<?php
// /includes/queue/QueueManager.php

namespace WHMCS\Queue;

class QueueManager
{
    private $connection;
    private $defaultQueue = 'default';
    private $maxRetries = 3;
    private $retryDelay = 60; // seconds

    public function __construct()
    {
        $this->connection = Capsule::connection();
    }

    /**
     * Push a job onto the queue
     */
    public function push(string $job, array $data = [], array $options = [])
    {
        $queue = $options['queue'] ?? $this->defaultQueue;
        $delay = $options['delay'] ?? 0;
        $maxAttempts = $options['max_attempts'] ?? $this->maxRetries;

        $availableAt = $delay > 0
            ? date('Y-m-d H:i:s', time() + $delay)
            : date('Y-m-d H:i:s');

        return Capsule::table('mod_queue_jobs')->insertGetId([
            'queue' => $queue,
            'job' => $job,
            'payload' => json_encode([
                'data' => $data,
                'job' => $job,
                'attempts' => 0
            ]),
            'attempts' => 0,
            'max_attempts' => $maxAttempts,
            'available_at' => $availableAt,
            'created_at' => date('Y-m-d H:i:s')
        ]);
    }

    /**
     * Push a job to run immediately
     */
    public function now(string $job, array $data = [])
    {
        return $this->push($job, $data, ['delay' => 0]);
    }

    /**
     * Push a delayed job
     */
    public function later(int $seconds, string $job, array $data = [])
    {
        return $this->push($job, $data, ['delay' => $seconds]);
    }

    /**
     * Push a job with delay until specific time
     */
    public function laterAt(string $timestamp, string $job, array $data = [])
    {
        $delay = max(0, strtotime($timestamp) - time());
        return $this->push($job, $data, ['delay' => $delay]);
    }

    /**
     * Pop the next job from the queue
     */
    public function pop(string $queue = null)
    {
        $queue = $queue ?? $this->defaultQueue;

        // Find available job
        $job = Capsule::table('mod_queue_jobs')
            ->where('queue', $queue)
            ->where('available_at', '<=', date('Y-m-d H:i:s'))
            ->whereNull('reserved_at')
            ->whereNull('completed_at')
            ->orderBy('available_at', 'ASC')
            ->first();

        if (!$job) {
            return null;
        }

        // Reserve the job
        Capsule::table('mod_queue_jobs')
            ->where('id', $job->id)
            ->update([
                'reserved_at' => date('Y-m-d H:i:s'),
                'attempts' => $job->attempts + 1
            ]);

        return $job;
    }

    /**
     * Complete a job successfully
     */
    public function complete($job)
    {
        Capsule::table('mod_queue_jobs')
            ->where('id', $job->id)
            ->update([
                'completed_at' => date('Y-m-d H:i:s')
            ]);

        logActivity("Queue job completed: {$job->job}");
    }

    /**
     * Fail a job and optionally retry
     */
    public function fail($job, Exception $exception)
    {
        $attempts = $job->attempts;
        $maxAttempts = $job->max_attempts;

        if ($attempts < $maxAttempts) {
            // Schedule retry with backoff
            $delay = $this->retryDelay * pow(2, $attempts - 1);

            Capsule::table('mod_queue_jobs')
                ->where('id', $job->id)
                ->update([
                    'available_at' => date('Y-m-d H:i:s', time() + $delay),
                    'reserved_at' => null,
                    'error' => $exception->getMessage()
                ]);

            logActivity("Queue job retry scheduled: {$job->job} (attempt {$attempts})");
        } else {
            // Move to failed jobs
            Capsule::table('mod_queue_failed_jobs')->insert([
                'queue' => $job->queue,
                'job' => $job->job,
                'payload' => $job->payload,
                'error' => $exception->getMessage(),
                'failed_at' => date('Y-m-d H:i:s')
            ]);

            // Mark original as completed (but failed)
            Capsule::table('mod_queue_jobs')
                ->where('id', $job->id)
                ->update([
                    'completed_at' => date('Y-m-d H:i:s'),
                    'error' => 'Max retries exceeded: ' . $exception->getMessage()
                ]);

            logActivity("Queue job failed permanently: {$job->job}");

            // Notify admin
            $this->notifyFailedJob($job, $exception);
        }
    }

    private function notifyFailedJob($job, Exception $exception)
    {
        $data = json_decode($job->payload, true);

        sendAdminEmail('Queue Job Failed', [
            'job' => $job->job,
            'queue' => $job->queue,
            'attempts' => $job->attempts,
            'error' => $exception->getMessage(),
            'data' => json_encode($data['data'] ?? [])
        ]);
    }
}
```

### Step 3: Create Job Classes
```php
<?php
// /includes/queue/jobs/SendEmailJob.php

namespace WHMCS\Queue\Jobs;

use WHMCS\Queue\QueueManager;

class SendEmailJob
{
    public static function handle($data)
    {
        $to = $data['to'];
        $template = $data['template'];
        $vars = $data['vars'] ?? [];
        $attachments = $data['attachments'] ?? [];

        return sendEmail($to, $template, $vars, $attachments);
    }

    public static function getName(): string
    {
        return 'SendEmailJob';
    }
}

// /includes/queue/jobs/ProvisionServiceJob.php

class ProvisionServiceJob
{
    public static function handle($data)
    {
        $serviceId = $data['service_id'];
        $serverId = $data['server_id'] ?? null;

        // Get service details
        $service = Capsule::table('tblhosting')
            ->where('id', $serviceId)
            ->first();

        if (!$service) {
            throw new Exception("Service not found: {$serviceId}");
        }

        // Get server
        if (!$serverId) {
            $serverId = $service->server;
        }

        $server = Capsule::table('tblservers')
            ->where('id', $serverId)
            ->first();

        if (!$server) {
            throw new Exception("Server not found: {$serverId}");
        }

        // Provision the service
        $result = provisionServiceOnServer($service, $server);

        if (!$result['success']) {
            throw new Exception("Provisioning failed: " . ($result['error'] ?? 'Unknown'));
        }

        // Update service status
        Capsule::table('tblhosting')
            ->where('id', $serviceId)
            ->update([
                'domainstatus' => 'Active',
                'username' => $result['username']
            ]);

        // Send activation email
        $queue = new QueueManager();
        $queue->later(5, 'SendEmailJob', [
            'to' => $service->userid,
            'template' => 'Service Activated',
            'vars' => [
                'service_id' => $serviceId,
                'details' => $result
            ]
        ]);

        return $result;
    }

    public static function getName(): string
    {
        return 'ProvisionServiceJob';
    }
}

// /includes/queue/jobs/SyncExternalJob.php

class SyncExternalJob
{
    public static function handle($data)
    {
        $type = $data['type']; // 'crm', 'accounting', 'support'
        $action = $data['action']; // 'create', 'update', 'delete'
        $entityId = $data['entity_id'];
        $entityType = $data['entity_type']; // 'client', 'invoice', 'service'

        // Route to appropriate sync handler
        $handler = "sync{$entityType}" . ucfirst($action);

        if (method_exists(__CLASS__, $handler)) {
            return self::$handler($entityId);
        }

        throw new Exception("Unknown sync action: {$type}/{$action}/{$entityType}");
    }

    private static function syncClientCreate($clientId)
    {
        $client = getClientsDetails($clientId);

        // Sync to CRM
        $crmResult = syncToCRM('create', 'client', $client);

        // Sync to accounting
        $accountingResult = syncToAccounting('create', 'client', $client);

        return [
            'crm' => $crmResult,
            'accounting' => $accountingResult
        ];
    }

    private static function syncClientUpdate($clientId)
    {
        $client = getClientsDetails($clientId);
        return [
            'crm' => syncToCRM('update', 'client', $client),
            'accounting' => syncToAccounting('update', 'client', $client)
        ];
    }

    public static function getName(): string
    {
        return 'SyncExternalJob';
    }
}
```

### Step 4: Create Queue Worker
```php
<?php
// /includes/queue/QueueWorker.php

namespace WHMCS\Queue;

class QueueWorker
{
    private $queue;
    private $sleepSeconds = 5;
    private $maxIterations = 0;
    private $timeout = 300; // 5 minutes max per job
    private $iterations = 0;

    public function __construct()
    {
        $this->queue = new QueueManager();
    }

    /**
     * Run the queue worker
     */
    public function run(array $options = [])
    {
        $options = array_merge([
            'queue' => 'default',
            'sleep' => 5,
            'max_iterations' => 0,
            'timeout' => 300
        ], $options);

        $this->sleepSeconds = $options['sleep'];
        $this->maxIterations = $options['max_iterations'];
        $this->timeout = $options['timeout'];

        logActivity("Queue worker started");

        while (true) {
            $this->iterations++;

            // Check if should stop
            if ($this->maxIterations > 0 && $this->iterations > $this->maxIterations) {
                logActivity("Queue worker stopped: max iterations reached");
                break;
            }

            // Get next job
            $job = $this->queue->pop($options['queue']);

            if ($job) {
                $this->processJob($job);
            } else {
                // No job available, sleep
                sleep($this->sleepSeconds);
            }
        }
    }

    private function processJob($job)
    {
        $startTime = time();
        $payload = json_decode($job->payload, true);
        $jobClass = $payload['job'];

        logActivity("Processing queue job: {$jobClass}");

        try {
            // Load job handler
            $handler = $this->resolveJob($jobClass);

            // Execute job
            $result = call_user_func([$handler, 'handle'], $payload['data']);

            // Mark as complete
            $this->queue->complete($job);

            logActivity("Queue job completed: {$jobClass}");
        } catch (Exception $e) {
            logActivity("Queue job failed: {$jobClass} - " . $e->getMessage());
            $this->queue->fail($job, $e);
        }
    }

    private function resolveJob($jobClass)
    {
        $classMap = [
            'SendEmailJob' => \WHMCS\Queue\Jobs\SendEmailJob::class,
            'ProvisionServiceJob' => \WHMCS\Queue\Jobs\ProvisionServiceJob::class,
            'SyncExternalJob' => \WHMCS\Queue\Jobs\SyncExternalJob::class,
        ];

        if (isset($classMap[$jobClass])) {
            return new $classMap[$jobClass]();
        }

        if (class_exists($jobClass)) {
            return new $jobClass();
        }

        throw new Exception("Unknown job class: {$jobClass}");
    }
}
```

### Step 5: Set Up Queue Worker Cron
```bash
# Add to crontab
# Run queue worker every minute
* * * * * php -q /path/to/whmcs/includes/queue/worker.php >> /var/log/whmcs-queue.log 2>&1
```

```php
<?php
// /includes/queue/worker.php

require_once __DIR__ . '/../../../init.php';
require_once __DIR__ . '/QueueWorker.php';

use WHMCS\Queue\QueueWorker;

$worker = new QueueWorker();

$worker->run([
    'queue' => 'default',
    'sleep' => 5,
    'max_iterations' => 12, // Run for ~1 minute then restart
    'timeout' => 300
]);
```

### Step 6: Hooks to Queue Jobs
```php
<?php
// /includes/hooks/queue_hooks.php

use WHMCS\Queue\QueueManager;

$queue = new QueueManager();

// Queue emails on service creation
add_hook('AfterServiceCreate', 1, function($vars) {
    $queue->later(1, 'SendEmailJob', [
        'to' => $vars['userid'],
        'template' => 'Service Welcome',
        'vars' => [
            'service_id' => $vars['serviceid'],
            'product_name' => getProductName($vars['packageid'])
        ]
    ]);
});

// Queue provisioning
add_hook('AfterServiceCreate', 1, function($vars) {
    // Queue provisioning for async execution
    $queue->now('ProvisionServiceJob', [
        'service_id' => $vars['serviceid'],
        'server_id' => null // Auto-select
    ]);
});

// Queue external sync
add_hook('ClientAdd', 1, function($vars) {
    $queue->now('SyncExternalJob', [
        'type' => 'crm',
        'action' => 'create',
        'entity_id' => $vars['userid'],
        'entity_type' => 'client'
    ]);
});

add_hook('InvoicePaid', 1, function($vars) {
    $invoice = getInvoice($vars['invoiceid']);

    $queue->later(5, 'SyncExternalJob', [
        'type' => 'accounting',
        'action' => 'create',
        'entity_id' => $vars['invoiceid'],
        'entity_type' => 'invoice'
    ]);

    // Queue receipt email
    $queue->later(2, 'SendEmailJob', [
        'to' => $invoice['userid'],
        'template' => 'Payment Received',
        'vars' => [
            'invoice_id' => $vars['invoiceid'],
            'amount' => $invoice['total']
        ]
    ]);
});
```

### Step 7: Monitor Queue Status
```php
/**
 * Get queue statistics
 */
function getQueueStats()
{
    return [
        'pending' => Capsule::table('mod_queue_jobs')
            ->whereNull('completed_at')
            ->whereNull('reserved_at')
            ->count(),

        'processing' => Capsule::table('mod_queue_jobs')
            ->whereNotNull('reserved_at')
            ->whereNull('completed_at')
            ->count(),

        'completed_today' => Capsule::table('mod_queue_jobs')
            ->whereNotNull('completed_at')
            ->whereDate('completed_at', date('Y-m-d'))
            ->count(),

        'failed' => Capsule::table('mod_queue_failed_jobs')
            ->whereDate('failed_at', '>=', date('Y-m-d'))
            ->count(),

        'by_queue' => Capsule::table('mod_queue_jobs')
            ->select('queue', Capsule::raw('COUNT(*) as count'))
            ->whereNull('completed_at')
            ->groupBy('queue')
            ->get()
    ];
}

/**
 * Display queue status in admin
 */
add_hook('AdminHomepage', 1, function($vars) {
    $stats = getQueueStats();

    return [
        'queueStatus' => [
            'pending' => $stats['pending'],
            'processing' => $stats['processing'],
            'failed_today' => $stats['failed']
        ]
    ];
});
```

## Queue Monitoring Dashboard
```php
// /includes/queue/dashboard.php - Admin dashboard widget

function renderQueueDashboard()
{
    $stats = getQueueStats();

    $html = '<div class="queue-dashboard">';
    $html .= '<h3>Queue Status</h3>';
    $html .= '<ul>';
    $html .= '<li>Pending: ' . $stats['pending'] . '</li>';
    $html .= '<li>Processing: ' . $stats['processing'] . '</li>';
    $html .= '<li>Completed Today: ' . $stats['completed_today'] . '</li>';
    $html .= '<li>Failed Today: ' . $stats['failed'] . '</li>';
    $html .= '</ul>';

    if ($stats['pending'] > 100) {
        $html .= '<div class="alert alert-warning">';
        $html .= 'High queue depth - consider adding more workers';
        $html .= '</div>';
    }

    $html .= '</div>';

    return $html;
}
```

## Queue Processing Best Practices

1. **Idempotency** - Jobs should be safe to retry
2. **Small Payloads** - Store IDs, not full objects
3. **Timeout Limits** - Set reasonable job timeouts
4. **Error Handling** - Always catch and log exceptions
5. **Monitoring** - Track queue depth and latency
6. **Scaling** - Run multiple workers if needed

## Queue Configuration

| Setting | Default | Description |
|---------|---------|-------------|
| Max Retries | 3 | Times to retry failed job |
| Retry Delay | 60s | Initial retry delay |
| Worker Timeout | 300s | Max job execution time |
| Sleep Interval | 5s | Time between job polls |

## Related Workflows
- [WHMCS Cron Automation](./whmcs-cron-automation.md)
- [WHMCS Async Tasks](./whmcs-async-tasks.md)
- [WHMCS Scheduled Tasks](./whmcs-scheduled-tasks.md)