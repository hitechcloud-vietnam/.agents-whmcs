# WHMCS Cron Development Skill
# Version: 1.0 | Updated: 2026-05-29

## Purpose

Guide for developing custom cron jobs and automated tasks within the WHMCS system, including scheduling patterns, queue processing, and monitoring.

## When to Use

- Creating scheduled automation tasks
- Building background job processors
- Implementing periodic data synchronization
- Setting up maintenance tasks
- Creating custom billing cycles

## Cron System Overview

### 1. WHMCS Cron Configuration

```php
<?php
// includes/crons/custom_tasks.php

if (!defined("WHMCS")) { die("Direct access denied"); }

use WHMCS\Database\Capsule;

/**
 * Custom Cron Configuration
 * Schedule: Every 5 minutes, hourly, daily, weekly, monthly
 *
 * Add to crontab:
 * /usr/bin/php -q /path/to/whmcs/crons/crons.php CustomTask
 */
```

### 2. Cron Task Structure

```php
<?php
// modules/addons/myaddon/cron/ProcessTasks.php

namespace MyAddon\Cron;

class ProcessTasks {
    private int $batchSize = 100;
    private int $maxRuntime = 55; // Leave buffer before cron timeout

    public function execute(): void {
        $startTime = time();
        $processed = 0;

        logActivity('[MyAddon Cron] Starting task processing');

        while ($processed < $this->batchSize) {
            // Check if we're approaching timeout
            if ((time() - $startTime) > $this->maxRuntime) {
                logActivity("[MyAddon Cron] Timeout reached, processed {$processed} items");
                break;
            }

            $task = $this->fetchNextTask();
            if (!$task) {
                break; // No more tasks
            }

            try {
                $this->processTask($task);
                $this->markCompleted($task->id);
                $processed++;
            } catch (\Exception $e) {
                $this->markFailed($task->id, $e->getMessage());
                logActivity("[MyAddon Cron] Task {$task->id} failed: " . $e->getMessage());
            }
        }

        logActivity("[MyAddon Cron] Completed. Processed: {$processed} tasks");
    }

    private function fetchNextTask(): ?object {
        return Capsule::table('mod_myaddon_tasks')
            ->where('status', 'pending')
            ->where('attempts', '<', 3)
            ->orderBy('priority', 'ASC')
            ->orderBy('created_at', 'ASC')
            ->first();
    }

    private function processTask(object $task): void {
        $payload = json_decode($task->payload, true);

        switch ($task->type) {
            case 'send_email':
                $this->sendEmailTask($payload);
                break;
            case 'sync_data':
                $this->syncDataTask($payload);
                break;
            case 'generate_report':
                $this->generateReportTask($payload);
                break;
            default:
                throw new \Exception("Unknown task type: {$task->type}");
        }
    }

    private function markCompleted(int $taskId): void {
        Capsule::table('mod_myaddon_tasks')
            ->where('id', $taskId)
            ->update([
                'status' => 'completed',
                'completed_at' => date('Y-m-d H:i:s'),
            ]);
    }

    private function markFailed(int $taskId, string $error): void {
        Capsule::table('mod_myaddon_tasks')
            ->where('id', $taskId)
            ->update([
                'status' => 'failed',
                'error_message' => $error,
                'attempts' => Capsule::raw('attempts + 1'),
            ]);
    }
}
```

### 3. Hook-Based Cron Jobs

```php
<?php
// hooks.php - Using Daily/Hourly cron hooks

add_hook('DailyCronJob', 1, function($vars) {
    $this->processDailyBilling();
    $this->syncDomainStatuses();
    $this->generateUsageReports();
    $this->cleanupOldData();
});

add_hook('HourlyCronJob', 1, function($vars) {
    $this->processPendingTasks();
    $this->checkServiceExpirations();
});

add_hook('PreCronJob', 1, function($vars) {
    // Run before WHMCS cron tasks
    $this->prepareCronEnvironment();
});

add_hook('PostCronJob', 1, function($vars) {
    // Run after WHMCS cron tasks
    $this->notifyCronCompletion();
});

// Daily tasks implementation
private function processDailyBilling(): void {
    logActivity('[Cron] Starting daily billing processing');

    // Get overdue invoices
    $overdueInvoices = Capsule::table('tblinvoices')
        ->where('status', 'Unpaid')
        ->where('duedate', '<', date('Y-m-d'))
        ->get();

    foreach ($overdueInvoices as $invoice) {
        $daysOverdue = (strtotime(date('Y-m-d')) - strtotime($invoice->duedate)) / 86400;

        if ($daysOverdue == 1) {
            // First reminder
            $this->sendOverdueReminder($invoice, 'first');
        } elseif ($daysOverdue == 7) {
            // Second reminder
            $this->sendOverdueReminder($invoice, 'second');
        } elseif ($daysOverdue >= 14) {
            // Suspension warning
            $this->sendSuspensionWarning($invoice);
        }
    }

    logActivity('[Cron] Daily billing processing complete');
}

private function syncDomainStatuses(): void {
    // Sync domain statuses with registrars
    $domains = Capsule::table('tbldomains')
        ->where('status', 'Active')
        ->where('expirydate', '<=', date('Y-m-d', strtotime('+30 days')))
        ->get();

    foreach ($domains as $domain) {
        // Check registrar for current status
        $registrar = \WHMCS\Domains::getRegistrar($domain->registrar);
        $domainData = $registrar->sync($domain->id);

        if ($domainData['status'] === 'Active') {
            Capsule::table('tbldomains')
                ->where('id', $domain->id)
                ->update([
                    'expirydate' => $domainData['expiry'],
                ]);
        }
    }
}
```

## Advanced Cron Patterns

### 1. Queue-Based Processing

```php
<?php
// modules/addons/myaddon/cron/QueueWorker.php

namespace MyAddon\Cron;

use WHMCS\Database\Capsule;

class QueueWorker {
    private array $handlers = [];
    private int $maxJobs = 100;
    private int $memoryLimit = 128; // MB

    public function __construct() {
        // Register job handlers
        $this->handlers = [
            'process_order' => ProcessOrderJob::class,
            'send_notification' => SendNotificationJob::class,
            'sync_inventory' => SyncInventoryJob::class,
            'generate_invoice' => GenerateInvoiceJob::class,
        ];
    }

    public function run(): void {
        $this->checkMemory();
        $processed = 0;

        logActivity('[QueueWorker] Starting queue processing');

        while ($processed < $this->maxJobs) {
            $this->checkMemory();

            $job = $this->reserveJob();
            if (!$job) {
                break; // Queue empty
            }

            $this->processJob($job);
            $processed++;
        }

        logActivity("[QueueWorker] Processed {$processed} jobs");
    }

    private function reserveJob(): ?object {
        // Atomic job reservation with row lock
        Capsule::connection()->transaction(function () {
            $job = Capsule::table('mod_queue')
                ->where('status', 'pending')
                ->where('available_at', '<=', date('Y-m-d H:i:s'))
                ->orderBy('priority', 'ASC')
                ->orderBy('created_at', 'ASC')
                ->lockForUpdate()
                ->first();

            if ($job) {
                Capsule::table('mod_queue')
                    ->where('id', $job->id)
                    ->update([
                        'status' => 'processing',
                        'started_at' => date('Y-m-d H:i:s'),
                    ]);
            }

            return $job;
        });

        return $job ?? null;
    }

    private function processJob(object $job): void {
        $handler = $this->handlers[$job->type] ?? null;

        if (!$handler) {
            $this->failJob($job->id, "Unknown job type: {$job->type}");
            return;
        }

        $processor = new $handler();
        $payload = json_decode($job->payload, true);

        try {
            $processor->handle($payload);
            $this->completeJob($job->id);
        } catch (\Exception $e) {
            $this->handleJobFailure($job, $e);
        }
    }

    private function completeJob(int $jobId): void {
        Capsule::table('mod_queue')
            ->where('id', $jobId)
            ->update([
                'status' => 'completed',
                'completed_at' => date('Y-m-d H:i:s'),
            ]);
    }

    private function handleJobFailure(object $job, \Exception $e): void {
        $attempts = $job->attempts + 1;

        if ($attempts >= 5) {
            Capsule::table('mod_queue')
                ->where('id', $job->id)
                ->update([
                    'status' => 'failed',
                    'error' => $e->getMessage(),
                    'attempts' => $attempts,
                    'failed_at' => date('Y-m-d H:i:s'),
                ]);
        } else {
            // Exponential backoff
            $delay = pow(2, $attempts) * 60;
            Capsule::table('mod_queue')
                ->where('id', $job->id)
                ->update([
                    'status' => 'pending',
                    'attempts' => $attempts,
                    'available_at' => date('Y-m-d H:i:s', time() + $delay),
                    'error' => $e->getMessage(),
                ]);
        }
    }

    private function checkMemory(): void {
        $memoryUsage = memory_get_usage(true) / 1024 / 1024;

        if ($memoryUsage > $this->memoryLimit) {
            logActivity("[QueueWorker] Memory limit reached: {$memoryUsage}MB");
            exit(0);
        }
    }

    // Dispatch jobs
    public static function dispatch(string $type, array $payload, int $priority = 10): void {
        Capsule::table('mod_queue')->insert([
            'type' => $type,
            'payload' => json_encode($payload),
            'priority' => $priority,
            'status' => 'pending',
            'available_at' => date('Y-m-d H:i:s'),
            'created_at' => date('Y-m-d H:i:s'),
            'attempts' => 0,
        ]);
    }
}
```

### 2. Scheduled Task Manager

```php
<?php
// modules/addons/myaddon/cron/Scheduler.php

namespace MyAddon\Cron;

use WHMCS\Database\Capsule;

class Scheduler {
    private array $schedules = [];

    public function __construct() {
        $this->loadSchedules();
    }

    private function loadSchedules(): void {
        $this->schedules = [
            'every_minute' => ['interval' => 60, 'callback' => 'runMinuteTasks'],
            'every_5_minutes' => ['interval' => 300, 'callback' => 'runFiveMinuteTasks'],
            'every_hour' => ['interval' => 3600, 'callback' => 'runHourlyTasks'],
            'every_day' => ['interval' => 86400, 'callback' => 'runDailyTasks'],
            'every_week' => ['interval' => 604800, 'callback' => 'runWeeklyTasks'],
            'every_month' => ['interval' => 2592000, 'callback' => 'runMonthlyTasks'],
        ];
    }

    public function shouldRun(string $schedule): bool {
        $config = $this->schedules[$schedule] ?? null;
        if (!$config) {
            return false;
        }

        $lastRun = Capsule::table('mod_scheduler_log')
            ->where('schedule', $schedule)
            ->orderBy('started_at', 'DESC')
            ->first();

        if (!$lastRun) {
            return true;
        }

        $nextRun = strtotime($lastRun->started_at) + $config['interval'];
        return time() >= $nextRun;
    }

    public function run(string $schedule): void {
        $config = $this->schedules[$schedule] ?? null;
        if (!$config) {
            return;
        }

        $logId = Capsule::table('mod_scheduler_log')->insertGetId([
            'schedule' => $schedule,
            'started_at' => date('Y-m-d H:i:s'),
            'status' => 'running',
        ]);

        try {
            $callback = $config['callback'];
            $this->$callback();

            Capsule::table('mod_scheduler_log')
                ->where('id', $logId)
                ->update([
                    'status' => 'completed',
                    'completed_at' => date('Y-m-d H:i:s'),
                ]);
        } catch (\Exception $e) {
            Capsule::table('mod_scheduler_log')
                ->where('id', $logId)
                ->update([
                    'status' => 'failed',
                    'error' => $e->getMessage(),
                    'completed_at' => date('Y-m-d H:i:s'),
                ]);

            throw $e;
        }
    }

    // Schedule callbacks
    private function runMinuteTasks(): void {
        // Check for critical alerts
        $this->checkServerDowntimes();
        $this->checkPendingApprovals();
    }

    private function runFiveMinuteTasks(): void {
        // Process pending orders
        $this->processPendingOrders();
        // Check for expiring services
        $this->checkServiceExpirations();
    }

    private function runHourlyTasks(): void {
        // Clean up temp files
        $this->cleanupTempFiles();
        // Generate hourly reports
        $this->generateHourlyStats();
    }

    private function runDailyTasks(): void {
        // Process billing
        $this->processDailyBilling();
        // Sync domains
        $this->syncDomainStatuses();
        // Send daily reports
        $this->sendDailyReport();
    }

    private function runWeeklyTasks(): void {
        // Generate weekly summary
        $this->generateWeeklySummary();
        // Clean old logs
        $this->cleanupOldLogs();
        // Backup database
        $this->performWeeklyBackup();
    }

    private function runMonthlyTasks(): void {
        // Generate monthly reports
        $this->generateMonthlyReport();
        // Archive old data
        $this->archiveOldData();
        // Review client activities
        $this->reviewClientActivity();
    }
}
```

### 3. Cron Monitoring & Alerts

```php
<?php
// modules/addons/myaddon/cron/CronMonitor.php

namespace MyAddon\Cron;

use WHMCS\Database\Capsule;

class CronMonitor {
    private array $alertThresholds = [
        'max_execution_time' => 300, // 5 minutes
        'max_memory_mb' => 256,
        'max_queue_size' => 1000,
        'failed_jobs_threshold' => 10,
    ];

    public function checkHealth(): array {
        $health = [
            'status' => 'healthy',
            'checks' => [],
            'alerts' => [],
        ];

        // Check last run time
        $lastRun = $this->checkLastCronRun();
        if ($lastRun['is_stale']) {
            $health['alerts'][] = [
                'type' => 'cron_stale',
                'message' => "Cron not run for {$lastRun['minutes_ago']} minutes",
                'severity' => 'critical',
            ];
            $health['status'] = 'unhealthy';
        }

        // Check queue health
        $queueHealth = $this->checkQueueHealth();
        if (!$queueHealth['healthy']) {
            $health['alerts'][] = $queueHealth['alert'];
            $health['status'] = 'unhealthy';
        }

        // Check failed jobs
        $failedJobs = $this->checkFailedJobs();
        if ($failedJobs['count'] > $this->alertThresholds['failed_jobs_threshold']) {
            $health['alerts'][] = [
                'type' => 'failed_jobs',
                'message' => "{$failedJobs['count']} failed jobs in queue",
                'severity' => 'warning',
            ];
        }

        // Check resources
        $health['checks'] = [
            'last_cron_run' => $lastRun,
            'queue_size' => $this->getQueueSize(),
            'failed_jobs' => $failedJobs,
            'memory_usage' => memory_get_usage(true) / 1024 / 1024,
        ];

        return $health;
    }

    private function checkLastCronRun(): array {
        $lastCron = Capsule::table('tblactivitylog')
            ->where('description', 'like', '%Cron Job%')
            ->orderBy('id', 'DESC')
            ->first();

        if (!$lastCron) {
            return [
                'is_stale' => true,
                'minutes_ago' => 'unknown',
                'last_run' => null,
            ];
        }

        $minutesAgo = (time() - strtotime($lastCron->date)) / 60;

        return [
            'is_stale' => $minutesAgo > 10, // Stale if not run in 10 minutes
            'minutes_ago' => round($minutesAgo),
            'last_run' => $lastCron->date,
        ];
    }

    private function checkQueueHealth(): array {
        $pending = Capsule::table('mod_queue')
            ->where('status', 'pending')
            ->count();

        if ($pending > $this->alertThresholds['max_queue_size']) {
            return [
                'healthy' => false,
                'alert' => [
                    'type' => 'queue_overflow',
                    'message' => "Queue has {$pending} pending jobs",
                    'severity' => 'critical',
                ],
            ];
        }

        return ['healthy' => true];
    }

    private function checkFailedJobs(): array {
        return [
            'count' => Capsule::table('mod_queue')
                ->where('status', 'failed')
                ->count(),
            'recent' => Capsule::table('mod_queue')
                ->where('status', 'failed')
                ->where('failed_at', '>=', date('Y-m-d H:i:s', strtotime('-24 hours')))
                ->count(),
        ];
    }

    private function getQueueSize(): int {
        return Capsule::table('mod_queue')
            ->whereIn('status', ['pending', 'processing'])
            ->count();
    }

    public function sendAlert(array $alert): void {
        // Log alert
        logActivity("[CronMonitor] Alert: {$alert['type']} - {$alert['message']}");

        // Send notification
        Capsule::table('mod_alerts')->insert([
            'type' => $alert['type'],
            'message' => $alert['message'],
            'severity' => $alert['severity'],
            'created_at' => date('Y-m-d H:i:s'),
        ]);
    }
}
```

## Cron Table Schema

```php
<?php
// Migration for cron tables

use WHMCS\Database\Capsule;

Capsule::schema()->create('mod_queue', function($t) {
    $t->increments('id');
    $t->string('type', 50);
    $t->longText('payload');
    $t->integer('priority')->default(10);
    $t->string('status', 20)->default('pending');
    $t->timestamp('available_at')->nullable();
    $t->timestamp('started_at')->nullable();
    $t->timestamp('completed_at')->nullable();
    $t->timestamp('failed_at')->nullable();
    $t->text('error')->nullable();
    $t->integer('attempts')->default(0);
    $t->timestamps();

    $t->index(['status', 'available_at']);
    $t->index(['type', 'status']);
});

Capsule::schema()->create('mod_scheduler_log', function($t) {
    $t->increments('id');
    $t->string('schedule', 50);
    $t->timestamp('started_at');
    $t->timestamp('completed_at')->nullable();
    $t->string('status', 20);
    $t->text('error')->nullable();
    $t->timestamps();

    $t->index('schedule');
    $t->index('started_at');
});
```

## Crontab Configuration

```bash
# WHMCS Cron - Run every 5 minutes
*/5 * * * * /usr/bin/php -q /var/www/html/whmcs/crons/crons.php >/dev/null 2>&1

# Custom cron job - Every minute
* * * * * /usr/bin/php -q /var/www/html/whmcs/modules/addons/myaddon/cron/run.php >/dev/null 2>&1

# Queue worker - Every minute
* * * * * /usr/bin/php -q /var/www/html/whmcs/modules/addons/myaddon/cron/queue_worker.php >/dev/null 2>&1
```

## Checklist

- [ ] Cron table schema created
- [ ] Batch size configured appropriately
- [ ] Timeout handling implemented
- [ ] Error handling with retry logic
- [ ] Logging for all operations
- [ ] Memory usage monitoring
- [ ] Alert system for failures
- [ ] Crontab properly configured

---

**Related Skills:**
- whmcs-cron-automation
- whmcs-queue-processing
- whmcs-logging
- whmcs-error-handling
- whmcs-monitoring

**Reference:**
- WHMCS Cron System: https://developers.whmcs.com/cron-jobs/