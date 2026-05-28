# WHMCS Queue Worker Skill
# Version: 1.0 | Updated: 2026-05-28

## Purpose

Guide for implementing queue-based task processing.

## When to Use

- Async task processing
- Email batch sending
- External API integration

## Queue Patterns

```php
<?php
class QueueWorker {
    private int $maxJobs = 100;
    private int $timeout = 60;

    public function dispatch(string $job, array $data = []): int {
        return Capsule::table('mod_queue')->insertGetId([
            'job' => $job,
            'payload' => json_encode($data),
            'status' => 'pending',
            'attempts' => 0,
            'created_at' => date('Y-m-d H:i:s'),
        ]);
    }

    public function process(): void {
        $jobs = Capsule::table('mod_queue')
            ->where('status', 'pending')
            ->where('attempts', '<', 3)
            ->orderBy('created_at')
            ->limit($this->maxJobs)
            ->get();

        foreach ($jobs as $job) {
            $this->executeJob($job);
        }
    }

    private function executeJob($job): void {
        Capsule::table('mod_queue')
            ->where('id', $job->id)
            ->update([
                'status' => 'processing',
                'attempts' => $job->attempts + 1,
                'started_at' => date('Y-m-d H:i:s'),
            ]);

        try {
            $payload = json_decode($job->payload, true);
            $handler = $this->resolveHandler($job->job);
            $handler($payload);

            Capsule::table('mod_queue')
                ->where('id', $job->id)
                ->update([
                    'status' => 'completed',
                    'completed_at' => date('Y-m-d H:i:s'),
                ]);

        } catch (\Exception $e) {
            Capsule::table('mod_queue')
                ->where('id', $job->id)
                ->update([
                    'status' => $job->attempts >= 2 ? 'failed' : 'pending',
                    'error' => $e->getMessage(),
                ]);
        }
    }

    private function resolveHandler(string $job): callable {
        return match ($job) {
            'send_email' => fn($p) => $this->sendEmail($p),
            'sync_service' => fn($p) => $this->syncService($p),
            'process_payment' => fn($p) => $this->processPayment($p),
            default => throw new \Exception("Unknown job: $job"),
        };
    }
}
```

### Cron Processing
```php
add_hook('DailyCronJob', 1, function() {
    $worker = new QueueWorker();
    $worker->process();
});
```

---

**Related Skills:**
- whmcs-queue-processing
- whmcs-email-template-builder
- whmcs-cron-automation
