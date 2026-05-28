# WHMCS Queue Processing Skill
# Version: 1.0 | Updated: 2026-05-28

## Purpose

Guide for implementing queue-based processing in WHMCS modules.

## When to Use

- Async task processing
- Rate-limited API calls
- Background job handling

## Queue Patterns

### Queue Setup

```php
function setupQueue(): void {
    Capsule::schema()->create('mod_{module}_queue', function($t) {
        $t->increments('id');
        $t->string('task', 100);
        $t->text('payload');
        $t->string('status', 20)->default('pending');
        $t->integer('attempts')->default(0);
        $t->integer('max_attempts')->default(3);
        $t->text('error')->nullable();
        $t->timestamp('scheduled_at')->nullable();
        $t->timestamp('started_at')->nullable();
        $t->timestamp('completed_at')->nullable();
        $t->timestamp('created_at')->useCurrent();
        $t->index(['status', 'scheduled_at']);
    });
}
```

### Queue Operations

```php
class QueueManager {
    public function enqueue(string $task, array $payload, ?int $delay = null): int {
        return Capsule::table('mod_{module}_queue')->insertGetId([
            'task' => $task,
            'payload' => json_encode($payload),
            'status' => 'pending',
            'scheduled_at' => $delay ? date('Y-m-d H:i:s', time() + $delay) : null,
            'created_at' => date('Y-m-d H:i:s'),
        ]);
    }

    public function processQueue(int $limit = 10): array {
        $jobs = Capsule::table('mod_{module}_queue')
            ->where('status', 'pending')
            ->where(function($q) {
                $q->whereNull('scheduled_at')
                  ->orWhere('scheduled_at', '<=', date('Y-m-d H:i:s'));
            })
            ->limit($limit)
            ->get();

        $results = ['processed' => 0, 'failed' => 0];

        foreach ($jobs as $job) {
            $this->processJob($job);
            $results[$job->status === 'completed' ? 'processed' : 'failed']++;
        }

        return $results;
    }

    private function processJob(object $job): void {
        Capsule::table('mod_{module}_queue')
            ->where('id', $job->id)
            ->update(['started_at' => date('Y-m-d H:i:s')]);

        try {
            $payload = json_decode($job->payload, true);
            $handler = $this->getHandler($job->task);

            $handler($payload);

            Capsule::table('mod_{module}_queue')
                ->where('id', $job->id)
                ->update([
                    'status' => 'completed',
                    'completed_at' => date('Y-m-d H:i:s'),
                ]);
        } catch (\Exception $e) {
            $attempts = $job->attempts + 1;

            if ($attempts >= $job->max_attempts) {
                Capsule::table('mod_{module}_queue')
                    ->where('id', $job->id)
                    ->update([
                        'status' => 'failed',
                        'error' => $e->getMessage(),
                        'attempts' => $attempts,
                    ]);
            } else {
                // Retry with exponential backoff
                Capsule::table('mod_{module}_queue')
                    ->where('id', $job->id)
                    ->update([
                        'attempts' => $attempts,
                        'scheduled_at' => date('Y-m-d H:i:s', time() + pow(2, $attempts) * 60),
                        'error' => $e->getMessage(),
                    ]);
            }
        }
    }

    private function getHandler(string $task): callable {
        $handlers = [
            'sync_server' => function($p) { /* sync logic */ },
            'send_notification' => function($p) { /* notification logic */ },
            'process_payment' => function($p) { /* payment logic */ },
        ];

        return $handlers[$task] ?? function() { throw new \Exception('Unknown task'); };
    }
}
```

### Queue Worker Hook

```php
add_hook('HourlyCronJob', 1, function($vars) {
    $queue = new QueueManager();
    $results = $queue->processQueue(50);

    if ($results['failed'] > 0) {
        logActivity('Queue processing: ' . $results['processed'] . ' ok, ' . $results['failed'] . ' failed');
    }
});
```

## Checklist

- [ ] Queue table setup
- [ ] Enqueue function
- [ ] Process function
- [ ] Retry logic
- [ ] Scheduled tasks
- [ ] Worker hook

---

**Related Skills:**
- whmcs-cron-automation
- whmcs-error-handling
- whmcs-performance-optimization