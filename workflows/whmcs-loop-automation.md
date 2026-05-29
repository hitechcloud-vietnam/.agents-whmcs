# WHMCS Loop-Based Automation Workflow

## Overview
This workflow implements loop-based automation for processing collections of items in WHMCS.

## Prerequisites
- WHMCS with custom hook capability
- PHP 7.4+ for modern syntax
- Understanding of array processing

## Step-by-Step Process

### Step 1: Understand Loop Patterns
```
LOOP PATTERNS:
1. Collection Loop: Process each item in a set
2. Batch Loop: Process items in chunks
3. Nested Loop: Process multi-level data
4. Parallel Loop: Process items concurrently
5. Conditional Loop: Loop until condition met
```

### Step 2: Create Basic Collection Loops
```php
<?php
// /includes/hooks/loop_automation.php

/**
 * Loop-Based Automation Handlers
 */

// ============================================================================
// BATCH PROCESSING SERVICES
// ============================================================================

add_hook('DailyCronJob', 1, function($vars) {
    // Get all active services needing health check
    $services = Capsule::table('tblhosting')
        ->where('domainstatus', 'Active')
        ->where('servertype', '!=', '')
        ->get();

    $processed = 0;
    $failed = 0;

    foreach ($services as $service) {
        try {
            // Perform health check
            $healthStatus = checkServiceHealth($service);

            // Update service record
            Capsule::table('tblhosting')
                ->where('id', $service->id)
                ->update([
                    'notes' => json_encode([
                        'last_health_check' => date('Y-m-d H:i:s'),
                        'status' => $healthStatus['status'],
                        'response_time' => $healthStatus['response_time'] ?? null
                    ])
                ]);

            // Handle unhealthy services
            if ($healthStatus['status'] !== 'healthy') {
                handleUnhealthyService($service, $healthStatus);
            }

            $processed++;
        } catch (Exception $e) {
            logActivity("Health check failed for service {$service->id}: " . $e->getMessage());
            $failed++;
        }
    }

    return "Health checks: {$processed} processed, {$failed} failed";
});
```

### Step 3: Implement Chunked/Batch Processing
```php
/**
 * Process large datasets in chunks to avoid memory issues
 */
function processServicesInChunks($callback, $chunkSize = 100)
{
    $totalProcessed = 0;
    $offset = 0;

    do {
        // Fetch chunk of services
        $services = Capsule::table('tblhosting')
            ->where('domainstatus', 'Active')
            ->offset($offset)
            ->limit($chunkSize)
            ->get();

        if ($services->isEmpty()) {
            break;
        }

        // Process chunk
        foreach ($services as $service) {
            $callback($service);
            $totalProcessed++;
        }

        $offset += $chunkSize;

        // Free memory
        gc_collect_cycles();

    } while (count($services) === $chunkSize);

    return $totalProcessed;
}

// Usage in cron job
add_hook('DailyCronJob', 1, function($vars) {
    $results = [
        'checked' => 0,
        'updated' => 0,
        'errors' => 0
    ];

    $processed = processServicesInChunks(function($service) use (&$results) {
        $results['checked']++;

        // Example: Update expiration status
        if (shouldUpdateExpiration($service)) {
            updateServiceExpiration($service);
            $results['updated']++;
        }
    }, 50); // Process 50 at a time

    return "Processed {$processed} services. Updated: {$results['updated']}";
});
```

### Step 4: Create Nested Loops
```php
// Nested loop: Process services and their addons
add_hook('DailyCronJob', 1, function($vars) {
    // Get all active clients
    $clients = Capsule::table('tblclients')
        ->where('status', 'Active')
        ->get();

    $summary = [
        'clients_processed' => 0,
        'services_processed' => 0,
        'addons_processed' => 0,
        'actions_taken' => 0
    ];

    foreach ($clients as $client) {
        $summary['clients_processed']++;

        // Get client's services
        $services = Capsule::table('tblhosting')
            ->where('userid', $client->id)
            ->where('domainstatus', 'Active')
            ->get();

        foreach ($services as $service) {
            $summary['services_processed']++;

            // Get service addons
            $addons = Capsule::table('tblhostingaddons')
                ->where('hostingid', $service->id)
                ->where('status', 'Active')
                ->get();

            foreach ($addons as $addon) {
                $summary['addons_processed']++;

                // Process each addon
                $action = processAddon($addon, $service, $client);

                if ($action) {
                    $summary['actions_taken']++;
                }
            }

            // Process main service
            $action = processMainService($service, $client);

            if ($action) {
                $summary['actions_taken']++;
            }
        }

        // Batch save to reduce DB writes
        if ($summary['services_processed'] % 100 === 0) {
            Capsule::connection()->flushQueryLog();
        }
    }

    return "Loop complete: " . json_encode($summary);
});
```

### Step 5: Implement Parallel Processing
```php
/**
 * Process items in parallel using curl_multi
 */
function processInParallel(array $items, callable $callback, int $maxConcurrent = 10)
{
    $results = [];
    $chunks = array_chunk($items, $maxConcurrent);

    foreach ($chunks as $chunk) {
        $multiHandle = curl_multi_init();
        $handles = [];

        // Initialize handles
        foreach ($chunk as $index => $item) {
            $handles[$index] = createCurlHandle($item, $callback);
            curl_multi_add_handle($multiHandle, $handles[$index]);
        }

        // Execute in parallel
        $running = null;
        do {
            curl_multi_exec($multiHandle, $running);
            curl_multi_select($multiHandle);
        } while ($running > 0);

        // Collect results
        foreach ($handles as $index => $handle) {
            $results[$index] = curl_multi_getcontent($handle);
            curl_multi_remove_handle($multiHandle, $handle);
            curl_close($handle);
        }

        curl_multi_close($multiHandle);
    }

    return $results;
}

// Usage: Bulk provisioning check
add_hook('DailyCronJob', 1, function($vars) {
    $services = Capsule::table('tblhosting')
        ->where('domainstatus', 'Pending')
        ->where('servertype', '!=', '')
        ->limit(100)
        ->get();

    $results = processInParallel($services->toArray(), function($service) {
        return checkProvisioningStatus($service);
    }, 20);

    // Process results
    foreach ($results as $index => $result) {
        $service = $services[$index];

        if ($result['status'] === 'ready') {
            activateService($service->id);
        } elseif ($result['status'] === 'failed') {
            logProvisioningFailure($service->id, $result['error']);
        }
    }
});
```

### Step 6: Create Conditional Loop (While-Style)
```php
/**
 * Process queue until empty
 */
function processQueueUntilEmpty($queueName)
{
    $processed = 0;
    $maxIterations = 1000; // Safety limit
    $iteration = 0;

    while ($iteration < $maxIterations) {
        // Get next item from queue
        $item = Capsule::table('mod_automation_queue')
            ->where('queue_name', $queueName)
            ->where('status', 'pending')
            ->orderBy('priority', 'ASC')
            ->orderBy('created_at', 'ASC')
            ->first();

        if (!$item) {
            break; // Queue is empty
        }

        // Mark as processing
        Capsule::table('mod_automation_queue')
            ->where('id', $item->id)
            ->update(['status' => 'processing']);

        try {
            // Process item
            $result = processQueueItem($item);

            // Mark as completed
            Capsule::table('mod_automation_queue')
                ->where('id', $item->id)
                ->update([
                    'status' => 'completed',
                    'completed_at' => date('Y-m-d H:i:s'),
                    'result' => json_encode($result)
                ]);

            $processed++;
        } catch (Exception $e) {
            // Mark as failed with retry
            $retryCount = $item->retry_count ?? 0;

            if ($retryCount < 3) {
                Capsule::table('mod_automation_queue')
                    ->where('id', $item->id)
                    ->update([
                        'status' => 'pending',
                        'retry_count' => $retryCount + 1,
                        'last_error' => $e->getMessage()
                    ]);
            } else {
                Capsule::table('mod_automation_queue')
                    ->where('id', $item->id)
                    ->update([
                        'status' => 'failed',
                        'last_error' => $e->getMessage()
                    ]);
            }
        }

        $iteration++;
    }

    return $processed;
}
```

### Step 7: Implement Recursive Loop
```php
/**
 * Recursive: Process hierarchical data (e.g., parent-child services)
 */
function processServiceHierarchy($parentId, callable $callback, $depth = 0, $maxDepth = 10)
{
    // Get child services
    $children = Capsule::table('tblhosting')
        ->where('parent_service_id', $parentId)
        ->where('domainstatus', 'Active')
        ->get();

    foreach ($children as $child) {
        // Process this child
        $callback($child, $depth + 1);

        // Recurse into grandchildren
        if ($depth < $maxDepth) {
            processServiceHierarchy($child->id, $callback, $depth + 1, $maxDepth);
        }
    }
}

// Usage: Cascading suspend for parent-child services
add_hook('ServiceSuspended', 1, function($vars) {
    $parentServiceId = $vars['serviceid'];

    processServiceHierarchy($parentServiceId, function($service, $depth) {
        // Suspend child service
        localApi('SuspendAccount', [
            'serviceid' => $service->id,
            'suspendreason' => "Parent service suspended"
        ]);

        logActivity("Suspended child service {$service->id} (depth: {$depth})");
    });
});
```

### Step 8: Optimize Loop Performance
```php
// Performance optimization techniques

// 1. Use database indexes for loop queries
function ensureLoopIndexes()
{
    $indexes = [
        'tblhosting' => ['domainstatus', 'userid', 'nextduedate'],
        'tblinvoices' => ['status', 'userid', 'duedate'],
        'tblclients' => ['status', 'groupid']
    ];

    foreach ($indexes as $table => $columns) {
        foreach ($columns as $column) {
            if (!hasIndex($table, $column)) {
                Capsule::statement("ALTER TABLE {$table} ADD INDEX idx_{$column}({$column})");
            }
        }
    }
}

// 2. Use chunked queries with cursor
function efficientLargeLoop($limit = 10000)
{
    $count = 0;

    foreach (Capsule::table('tblhosting')
        ->where('domainstatus', 'Active')
        ->cursor() as $service) {

        processService($service);
        $count++;

        // Yield to prevent blocking
        if ($count % 100 === 0) {
            usleep(1000); // 1ms pause
        }
    }

    return $count;
}

// 3. Batch database operations
function batchUpdateServices(array $serviceIds, array $updates)
{
    foreach (array_chunk($serviceIds, 500) as $chunk) {
        Capsule::table('tblhosting')
            ->whereIn('id', $chunk)
            ->update($updates);
    }
}
```

### Step 9: Implement Loop Monitoring
```php
/**
 * Monitor loop execution and alert on issues
 */
class LoopMonitor
{
    private $loopName;
    private $startTime;
    private $itemCount = 0;
    private $errorCount = 0;

    public function __construct($loopName)
    {
        $this->loopName = $loopName;
        $this->startTime = microtime(true);
    }

    public function tick()
    {
        $this->itemCount++;

        // Warn every 1000 items
        if ($this->itemCount % 1000 === 0) {
            $elapsed = microtime(true) - $this->startTime;
            $rate = $this->itemCount / $elapsed;

            logActivity("Loop {$this->loopName}: {$this->itemCount} items, {$rate} items/sec");
        }
    }

    public function error(Exception $e)
    {
        $this->errorCount++;

        // Alert on high error rate
        if ($this->errorCount > 10 && ($this->errorCount / $this->itemCount) > 0.1) {
            sendAdminAlert("High error rate in {$this->loopName}: {$this->errorCount}/{$this->itemCount}");
        }
    }

    public function complete()
    {
        $elapsed = microtime(true) - $this->startTime;

        Capsule::table('mod_loop_log')->insert([
            'loop_name' => $this->loopName,
            'items_processed' => $this->itemCount,
            'errors' => $this->errorCount,
            'elapsed_seconds' => round($elapsed, 2),
            'completed_at' => date('Y-m-d H:i:s')
        ]);
    }
}
```

## Loop Pattern Selection Guide

| Pattern | Use Case | Performance |
|---------|----------|-------------|
| Basic foreach | Small datasets (<1000) | Good |
| Chunked | Large datasets (>10000) | Excellent |
| Nested | Hierarchical data | Varies |
| Parallel | I/O-bound operations | Excellent |
| Queue-based | Distributed processing | Excellent |
| Recursive | Tree structures | Good |

## Best Practices

1. **Set safety limits** - Prevent infinite loops
2. **Use chunking** - For large datasets
3. **Monitor memory** - Clear references
4. **Log progress** - Track execution
5. **Handle failures** - Implement retries
6. **Use indexes** - Speed up queries

## Related Workflows
- [WHMCS Batch Operations](./whmcs-batch-operations.md)
- [WHMCS Queue Processing](./whmcs-queue-processing.md)
- [WHMCS Async Tasks](./whmcs-async-tasks.md)