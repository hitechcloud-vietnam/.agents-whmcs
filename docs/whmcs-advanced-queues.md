# WHMCS Advanced Queues

Complete guide to queue-based processing.

## Overview

Implement asynchronous processing with queues.

## Queue System

### Queue Manager

```php
<?php
/**
 * Queue manager
 */
class QueueManager
{
    private RedisCache $redis;
    
    public function __construct(RedisCache $redis)
    {
        $this->redis = $redis;
    }
    
    /**
     * Push job to queue
     */
    public function push(string $queue, string $job, array $data): int
    {
        $jobData = [
            'id' => uniqid('job_'),
            'job' => $job,
            'data' => $data,
            'queued_at' => time(),
            'attempts' => 0,
        ];
        
        return $this->redis->listPush("queue:{$queue}", $jobData);
    }
    
    /**
     * Pop job from queue
     */
    public function pop(string $queue): ?array
    {
        return $this->redis->listPop("queue:{$queue}");
    }
    
    /**
     * Get queue size
     */
    public function size(string $queue): int
    {
        return $this->redis->listSize("queue:{$queue}");
    }
}
```

## Job Processing

### Job Handler

```php
<?php
/**
 * Job processor
 */
class JobProcessor
{
    private array $handlers = [];
    private QueueManager $queue;
    
    public function __construct(QueueManager $queue)
    {
        $this->queue = $queue;
    }
    
    /**
     * Register job handler
     */
    public function handle(string $job, callable $handler): void
    {
        $this->handlers[$job] = $handler;
    }
    
    /**
     * Process a job
     */
    public function process(string $queue = 'default'): ?array
    {
        $job = $this->queue->pop($queue);
        
        if (!$job) {
            return null;
        }
        
        $jobClass = $job['job'];
        
        if (!isset($this->handlers[$jobClass])) {
            throw new Exception("No handler for job: {$jobClass}");
        }
        
        $handler = $this->handlers[$jobClass];
        $result = $handler($job['data']);
        
        return [
            'job' => $job,
            'result' => $result,
        ];
    }
    
    /**
     * Run worker
     */
    public function run(string $queue = 'default', int $maxJobs = 100): void
    {
        for ($i = 0; $i < $maxJobs; $i++) {
            $result = $this->process($queue);
            
            if (!$result) {
                break; // Queue empty
            }
            
            echo "Processed: {$result['job']['id']}\n";
        }
    }
}
```

## Best Practices

1. **Idempotent jobs** - Handle repeated processing
2. **Retry logic** - Handle failures gracefully
3. **Timeout** - Prevent stuck jobs
4. **Monitoring** - Track queue size
5. **Priority** - Support job priorities
6. **Cleanup** - Remove completed jobs

## Related Documentation

- [whmcs-advanced-automation.md](whmcs-advanced-automation.md)
- [whmcs-advanced-caching.md](whmcs-advanced-caching.md)
