# Background Job Processing

Background job processing enables handling time-consuming tasks asynchronously. This guide covers patterns for implementing reliable job queues in WHMCS.

## Queue Architecture

### Job Interface

```php
<?php
/**
 * Job interface for background processing
 */
interface JobInterface
{
    /**
     * Execute the job
     */
    public function handle(): void;

    /**
     * Get unique job identifier
     */
    public function getJobId(): string;

    /**
     * Get job priority (lower = higher priority)
     */
    public function getPriority(): int;

    /**
     * Maximum retry attempts
     */
    public function getMaxAttempts(): int;

    /**
     * Retry delay in seconds
     */
    public function getRetryDelay(): int;

    /**
     * Handle job failure
     */
    public function failed(\Throwable $exception): void;
}
```

### Job Base Class

```php
<?php
/**
 * Base job class
 */
abstract class AbstractJob implements JobInterface
{
    protected int $attempts = 0;
    protected ?string $jobId = null;
    protected array $payload = [];

    public function __construct(array $payload = [])
    {
        $this->payload = $payload;
        $this->jobId = $this->generateJobId();
    }

    public function handle(): void
    {
        // Override in subclass
    }

    public function getJobId(): string
    {
        return $this->jobId;
    }

    public function getPriority(): int
    {
        return 0;
    }

    public function getMaxAttempts(): int
    {
        return 3;
    }

    public function getRetryDelay(): int
    {
        return 60;
    }

    public function failed(\Throwable $exception): void
    {
        logActivity("Job {$this->jobId} failed: " . $exception->getMessage());
    }

    protected function incrementAttempts(): void
    {
        $this->attempts++;
    }

    protected function getAttempt(): int
    {
        return $this->attempts;
    }

    protected function generateJobId(): string
    {
        return uniqid('job_', true) . '_' . bin2hex(random_bytes(8));
    }

    public function toArray(): array
    {
        return [
            'job_id' => $this->jobId,
            'class' => get_class($this),
            'payload' => $this->payload,
            'priority' => $this->getPriority(),
            'max_attempts' => $this->getMaxAttempts(),
            'retry_delay' => $this->getRetryDelay(),
            'created_at' => date('Y-m-d H:i:s'),
        ];
    }

    public static function fromArray(array $data): self
    {
        $class = $data['class'];
        return new $class($data['payload'] ?? []);
    }
}
```

## Queue Implementation

### Database Queue

```php
<?php
/**
 * Database-backed job queue
 */
class DatabaseQueue
{
    private string $table = 'mod_job_queue';
    private int $maxRetries = 3;

    /**
     * Push a job to the queue
     */
    public function push(JobInterface $job): string
    {
        $data = $job->toArray();
        $data['status'] = 'pending';
        $data['attempts'] = 0;
        $data['available_at'] = date('Y-m-d H:i:s');

        $id = Capsule::table($this->table)->insertGetId($data);

        return $job->getJobId();
    }

    /**
     * Push a delayed job
     */
    public function later(JobInterface $job, int $delaySeconds): string
    {
        $data = $job->toArray();
        $data['status'] = 'pending';
        $data['attempts'] = 0;
        $data['available_at'] = date('Y-m-d H:i:s', time() + $delaySeconds);

        Capsule::table($this->table)->insert($data);

        return $job->getJobId();
    }

    /**
     * Pop the next available job
     */
    public function pop(?string $queue = null): ?JobInterface
    {
        $job = Capsule::table($this->table)
            ->where('status', 'pending')
            ->where('available_at', '<=', date('Y-m-d H:i:s'))
            ->orderBy('priority', 'asc')
            ->orderBy('created_at', 'asc')
            ->first();

        if (!$job) {
            return null;
        }

        // Mark as processing
        Capsule::table($this->table)
            ->where('id', $job->id)
            ->update([
                'status' => 'processing',
                'started_at' => date('Y-m-d H:i:s'),
                'attempts' => $job->attempts + 1,
            ]);

        $instance = AbstractJob::fromArray((array) $job);
        $instance->attempts = $job->attempts;

        return $instance;
    }

    /**
     * Mark job as completed
     */
    public function complete(string $jobId): void
    {
        Capsule::table($this->table)
            ->where('job_id', $jobId)
            ->update([
                'status' => 'completed',
                'completed_at' => date('Y-m-d H:i:s'),
            ]);
    }

    /**
     * Mark job as failed and potentially retry
     */
    public function fail(string $jobId, \Throwable $exception): void
    {
        $job = Capsule::table($this->table)
            ->where('job_id', $jobId)
            ->first();

        if (!$job) {
            return;
        }

        if ($job->attempts < $job->max_attempts) {
            // Retry the job
            Capsule::table($this->table)
                ->where('job_id', $jobId)
                ->update([
                    'status' => 'pending',
                    'available_at' => date('Y-m-d H:i:s', time() + $job->retry_delay),
                    'last_error' => $exception->getMessage(),
                ]);
        } else {
            // Mark as permanently failed
            Capsule::table($this->table)
                ->where('job_id', $jobId)
                ->update([
                    'status' => 'failed',
                    'failed_at' => date('Y-m-d H:i:s'),
                    'last_error' => $exception->getMessage(),
                    'trace' => $exception->getTraceAsString(),
                ]);
        }
    }

    /**
     * Get queue statistics
     */
    public function stats(): array
    {
        return [
            'pending' => Capsule::table($this->table)->where('status', 'pending')->count(),
            'processing' => Capsule::table($this->table)->where('status', 'processing')->count(),
            'completed' => Capsule::table($this->table)->where('status', 'completed')->count(),
            'failed' => Capsule::table($this->table)->where('status', 'failed')->count(),
        ];
    }
}
```

## Job Types

### Email Job

```php
<?php
/**
 * Email sending job
 */
class SendEmailJob extends AbstractJob
{
    private string $to;
    private string $subject;
    private string $body;
    private array $attachments = [];

    public function __construct(
        string $to,
        string $subject,
        string $body,
        array $attachments = []
    ) {
        parent::__construct([]);
        $this->to = $to;
        $this->subject = $subject;
        $this->body = $body;
        $this->attachments = $attachments;
    }

    public function handle(): void
    {
        $mail = new PHPMailer(true);

        try {
            $mail->isSMTP();
            $mail->Host = App::get_config('SMTPHost');
            $mail->SMTPAuth = true;
            $mail->Username = App::get_config('SMTPUsername');
            $mail->Password = App::get_config('SMTPPassword');
            $mail->SMTPSecure = PHPMailer::ENCRYPTION_STARTTLS;
            $mail->Port = App::get_config('SMTPPort');

            $mail->setFrom(App::get_config('EmailFrom'), App::get_config('EmailFromName'));
            $mail->addAddress($this->to);
            $mail->Subject = $this->subject;
            $mail->Body = $this->body;

            foreach ($this->attachments as $attachment) {
                $mail->addAttachment($attachment['path'], $attachment['name']);
            }

            $mail->send();
        } finally {
            $mail->smtpClose();
        }
    }

    public function getPriority(): int
    {
        return 10; // Lower priority for emails
    }

    public function getMaxAttempts(): int
    {
        return 5;
    }

    public function getRetryDelay(): int
    {
        return 30;
    }
}
```

### Data Processing Job

```php
<?php
/**
 * Bulk data processing job
 */
class BulkDataProcessingJob extends AbstractJob
{
    private array $itemIds;
    private string $operation;

    public function __construct(array $itemIds, string $operation)
    {
        $this->itemIds = $itemIds;
        $this->operation = $operation;
    }

    public function handle(): void
    {
        $processed = 0;
        $failed = [];

        foreach ($this->itemIds as $itemId) {
            try {
                $this->processItem($itemId);
                $processed++;
            } catch (\Throwable $e) {
                $failed[] = [
                    'item_id' => $itemId,
                    'error' => $e->getMessage(),
                ];
            }

            // Batch progress tracking
            if ($processed % 100 === 0) {
                $this->updateProgress($processed, count($this->itemIds));
            }
        }

        // Store results
        if (!empty($failed)) {
            Capsule::table('mod_processing_failures')->insert($failed);
        }
    }

    private function processItem(int $itemId): void
    {
        // Process based on operation type
        switch ($this->operation) {
            case 'sync':
                $this->syncItem($itemId);
                break;
            case 'export':
                $this->exportItem($itemId);
                break;
            case 'cleanup':
                $this->cleanupItem($itemId);
                break;
            default:
                throw new InvalidArgumentException("Unknown operation: {$this->operation}");
        }
    }

    private function syncItem(int $itemId): void
    {
        // Sync implementation
    }

    private function exportItem(int $itemId): void
    {
        // Export implementation
    }

    private function cleanupItem(int $itemId): void
    {
        // Cleanup implementation
    }

    private function updateProgress(int $current, int $total): void
    {
        Capsule::table('mod_job_queue')
            ->where('job_id', $this->jobId)
            ->update([
                'progress' => round($current / $total * 100, 2),
            ]);
    }

    public function getMaxAttempts(): int
    {
        return 1; // Don't retry individual items
    }
}
```

## Queue Worker

### Job Processor

```php
<?php
/**
 * Queue worker processor
 */
class QueueWorker
{
    private DatabaseQueue $queue;
    private int $maxTime = 3600; // 1 hour
    private int $maxJobs = 100;
    private bool $shouldStop = false;

    public function __construct(DatabaseQueue $queue)
    {
        $this->queue = $queue;
    }

    /**
     * Run the worker
     */
    public function run(): void
    {
        $startTime = time();
        $jobsProcessed = 0;

        while (!$this->shouldStop && $this->canContinue($startTime, $jobsProcessed)) {
            $job = $this->queue->pop();

            if ($job === null) {
                // No jobs, sleep briefly
                usleep(100000); // 100ms
                continue;
            }

            $this->processJob($job);
            $jobsProcessed++;
        }

        logActivity("Queue worker stopped. Processed {$jobsProcessed} jobs.");
    }

    /**
     * Process a single job
     */
    private function processJob(JobInterface $job): void
    {
        $jobId = $job->getJobId();

        logActivity("Processing job: {$jobId}");

        try {
            $job->handle();
            $this->queue->complete($jobId);
            logActivity("Job completed: {$jobId}");
        } catch (\Throwable $e) {
            $this->queue->fail($jobId, $e);
            $job->failed($e);
            logActivity("Job failed: {$jobId} - " . $e->getMessage());
        }
    }

    private function canContinue(int $startTime, int $jobsProcessed): bool
    {
        return (time() - $startTime) < $this->maxTime
            && $jobsProcessed < $this->maxJobs;
    }

    /**
     * Stop the worker gracefully
     */
    public function stop(): void
    {
        $this->shouldStop = true;
    }
}
```

### Cron Job Runner

```php
<?php
/**
 * Cron-based job runner
 */
add_hook('DailyCronJob', 1, function ($vars) {
    $queue = new DatabaseQueue();
    $worker = new QueueWorker($queue);

    logActivity('Starting queue worker from cron');

    // Run for max 5 minutes during cron
    $worker->setMaxTime(300);
    $worker->run();

    logActivity('Queue worker completed');
});

// Alternative: dedicated cron command
function runQueueWorker_cli($args)
{
    $queue = new DatabaseQueue();
    $worker = new QueueWorker($queue);

    echo "Starting queue worker...\n";
    $worker->run();
    echo "Queue worker finished.\n";
}
```

## Job Scheduling

### Scheduled Jobs

```php
<?php
/**
 * Job scheduler
 */
class JobScheduler
{
    private array $schedules = [];

    /**
     * Schedule a recurring job
     */
    public function schedule(string $expression, JobInterface $job): void
    {
        $this->schedules[] = [
            'expression' => $expression,
            'job' => $job,
            'next_run' => $this->calculateNextRun($expression),
        ];
    }

    /**
     * Run due jobs
     */
    public function runDueJobs(): array
    {
        $now = new DateTime();
        $results = [];

        foreach ($this->schedules as $index => $schedule) {
            if ($schedule['next_run'] <= $now) {
                try {
                    $schedule['job']->handle();
                    $results[] = [
                        'job' => get_class($schedule['job']),
                        'status' => 'success',
                    ];
                } catch (\Throwable $e) {
                    $results[] = [
                        'job' => get_class($schedule['job']),
                        'status' => 'failed',
                        'error' => $e->getMessage(),
                    ];
                }

                // Calculate next run
                $this->schedules[$index]['next_run'] = $this->calculateNextRun(
                    $schedule['expression']
                );
            }
        }

        return $results;
    }

    private function calculateNextRun(string $expression): DateTime
    {
        // Parse cron expression
        $parts = explode(' ', trim($expression));

        if (count($parts) < 5) {
            throw new InvalidArgumentException('Invalid cron expression');
        }

        $now = new DateTime();
        $next = clone $now;

        // Simplified cron parsing
        // minute hour day month weekday
        $next->modify('+1 minute');

        // In production, use a proper cron library
        return $next;
    }
}

// Schedule examples
$scheduler = new JobScheduler();

// Run every minute
$scheduler->schedule('* * * * *', new SyncDataJob());

// Run every hour
$scheduler->schedule('0 * * * *', new CleanupJob());

// Run daily at midnight
$scheduler->schedule('0 0 * * *', new DailyReportJob());

// Cron hook to run scheduled jobs
add_hook('MinuteCronJob', 1, function ($vars) {
    $scheduler = App::make(JobScheduler::class);
    $scheduler->runDueJobs();
});
```

## Job Monitoring

### Job Status Tracker

```php
<?php
/**
 * Job status monitoring
 */
class JobMonitor
{
    /**
     * Get job status
     */
    public static function getStatus(string $jobId): ?array
    {
        $job = Capsule::table('mod_job_queue')
            ->where('job_id', $jobId)
            ->first();

        if (!$job) {
            return null;
        }

        return [
            'job_id' => $job->job_id,
            'status' => $job->status,
            'attempts' => $job->attempts,
            'progress' => $job->progress ?? 0,
            'created_at' => $job->created_at,
            'started_at' => $job->started_at,
            'completed_at' => $job->completed_at,
            'last_error' => $job->last_error,
        ];
    }

    /**
     * Get failed jobs
     */
    public static function getFailedJobs(int $limit = 50): array
    {
        return Capsule::table('mod_job_queue')
            ->where('status', 'failed')
            ->orderBy('failed_at', 'desc')
            ->limit($limit)
            ->get();
    }

    /**
     * Retry a failed job
     */
    public static function retry(string $jobId): bool
    {
        $job = Capsule::table('mod_job_queue')
            ->where('job_id', $jobId)
            ->where('status', 'failed')
            ->first();

        if (!$job) {
            return false;
        }

        Capsule::table('mod_job_queue')
            ->where('job_id', $jobId)
            ->update([
                'status' => 'pending',
                'attempts' => 0,
                'available_at' => date('Y-m-d H:i:s'),
                'last_error' => null,
            ]);

        return true;
    }
}
```

## Best Practices

1. **Idempotent jobs** - Jobs should be safe to run multiple times
2. **Handle failures gracefully** - Implement retry logic with backoff
3. **Track job progress** - Provide feedback for long-running jobs
4. **Use appropriate priorities** - Critical jobs should run first
5. **Monitor queue depth** - Alert when queue backs up
6. **Time out stuck jobs** - Prevent zombie jobs
7. **Log job execution** - Maintain audit trail
8. **Clean up old jobs** - Prune completed/failed jobs regularly

## Related Patterns

- [Queue Processing](./queue-processing.md) - Queue management
- [Cron Events Reference](./cron-events-reference.md) - Scheduled execution
- [Event Sourcing](./event-sourcing.md) - Event-driven job triggers