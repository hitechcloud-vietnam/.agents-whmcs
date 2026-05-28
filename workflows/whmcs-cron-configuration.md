# WHMCS Cron Configuration Workflow

## Purpose

Complete guide to configuring and optimizing WHMCS cron jobs for proper system operation. Covers cron scheduling, automation tasks, performance considerations, and troubleshooting.

## Prerequisites

- WHMCS installation
- Command-line access (SSH) or hosting control panel
- Basic understanding of cron syntax
- PHP CLI knowledge

## Workflow Steps

### Step 1: Understanding WHMCS Cron System

The WHMCS cron system handles automated tasks:

```php
/**
 * WHMCS Cron Jobs Overview
 *
 * Main cron command: php -q /path/to/whmcs/crons/crons.php
 *
 * Available cron tasks:
 * - Module Hook Calls (configurable intervals)
 * - Database Cleanup
 * - Temp Folder Cleanup
 * - Update Currency Exchange Rates
 * - Process Domain Transfer Sync
 * - Verify SSL Certificates
 * - Generate Invoices
 * - Send Invoice Reminders
 * - Process Suspended Services
 * - Process Overdue Services
 * - Process Cancellations
 * - Clear Hook Cache
 * - Affiliate Commission Processing
 * - Enforcement of automatic suspensions/terminations
 */

// Default cron configuration in configuration.php
// $cron_config = [];

 /**
  * Cron configuration options:
  *
  * Disable certain cron commands:
  * $cron_config = [
  *     'Disable' => [
  *         'InvoiceSupport',
  *         'ModuleSaveCLI'
  *     ]
  * ];
  *
  * Custom cron schedule (recommended):
  * $cron_config = [
  *     'Schedule' => [
  *         'daily' => '0 0 * * *',      // Midnight daily
  *         'hourly' => '0 * * * *',     // Every hour
  *         '5min' => '*/5 * * * *',     // Every 5 minutes
  *     ]
  * ];
 */
```

### Step 2: Setting Up the Primary Cron Job

Configure the main WHMCS cron with optimal scheduling:

```bash
# ===========================================
# WHMCS Main Cron Configuration
# ===========================================

# Primary cron - every 5 minutes (required for real-time operations)
*/5 * * * * /usr/bin/php -q /var/www/html/whmcs/crons/crons.php

# Alternative: Run specific cron groups
# */5 * * * * /usr/bin/php -q /var/www/html/whmcs/crons/crons.php RunTask=InvoiceSupport
# */5 * * * * /usr/bin/php -q /var/www/html/whmcs/crons/crons.php RunTask=ModuleQueue
# */5 * * * * /usr/bin/php -q /var/www/html/whmcs/crons/crons.php RunTask=DatabaseCleanup
```

```php
// Advanced cron configuration in configuration.php

// Custom cron settings
$cron_config = [
    'Schedule' => [
        'CreateInvoices' => '0 6 * * *',        // 6 AM daily
        'GenerateRenewalInvoices' => '0 6 1 * *', // 6 AM on 1st of month
        'ProcessInvoices' => '*/15 * * * *',    // Every 15 minutes
        'TicketEscalations' => '*/10 * * * *', // Every 10 minutes
        'DatabaseCleanup' => '0 3 * * *',       // 3 AM daily
    ],
    'Disable' => [
        'OldLogCleanup', // If not needed
    ],
    'Options' => [
        'DebugMode' => false,
        'SingleProcess' => true, // Prevent overlapping runs
    ],
];

// Cron mode: CLI or apache (CLI recommended)
$cron_mode = 'cli';
```

### Step 3: Creating Additional Cron Scripts

Create custom cron scripts for specialized automation:

```php
// crons/custom/cleanup_expired_sessions.php

#!/usr/bin/php
<?php
/**
 * Custom Cron: Cleanup Expired Sessions
 *
 * Cleans up expired session data to free disk space.
 * Run daily at off-peak hours.
 */

define('ROOTDIR', dirname(__DIR__, 4));

require_once ROOTDIR . '/init.php';
require_once ROOTDIR . '/vendor/autoload.php';

// Configuration
const SESSION_LIFETIME = 86400; // 24 hours
const DRY_RUN = false;

// Get expired sessions
$expiredTime = date('Y-m-d H:i:s', time() - SESSION_LIFETIME);

$expiredSessions = Capsule::table('tblsessions')
    ->where('created_at', '<', $expiredTime)
    ->count();

if ($expiredSessions > 0) {
    if (!DRY_RUN) {
        Capsule::table('tblsessions')
            ->where('created_at', '<', $expiredTime)
            ->delete();

        logActivity("Cleaned up {$expiredSessions} expired sessions");
    } else {
        echo "Would delete {$expiredSessions} expired sessions\n";
    }
}

echo "Session cleanup completed. Expired: {$expiredSessions}\n";
```

```php
// crons/custom/sync_external_provisioning.php

#!/usr/bin/php
<?php
/**
 * Custom Cron: External Provisioning Sync
 *
 * Syncs service status with external provisioning API.
 * Should run every 5-15 minutes.
 */

define('ROOTDIR', dirname(__DIR__, 4));
require_once ROOTDIR . '/init.php';

class ExternalProvisioningSync
{
    private $apiEndpoint;
    private $apiKey;
    private $batchSize = 50;
    private $syncLog = [];

    public function __construct()
    {
        $config = DI::make('config');
        $this->apiEndpoint = $config->get('external_provisioning_url');
        $this->apiKey = decrypt($config->get('external_provisioning_key'));
    }

    /**
     * Run full sync process
     */
    public function run(): void
    {
        $startTime = microtime(true);

        // Get services needing sync
        $services = $this->getServicesToSync();

        foreach ($services as $service) {
            try {
                $this->syncService($service);
            } catch (\Exception $e) {
                $this->logSyncError($service, $e->getMessage());
            }
        }

        $duration = round(microtime(true) - $startTime, 2);
        $totalSynced = count($this->syncLog['synced'] ?? []);
        $totalFailed = count($this->syncLog['failed'] ?? []);

        logActivity(sprintf(
            "External sync completed: %d synced, %d failed, %s seconds",
            $totalSynced, $totalFailed, $duration
        ));
    }

    /**
     * Get services pending sync
     */
    private function getServicesToSync(): array
    {
        return Capsule::table('tblhosting')
            ->join('tblservers', 'tblhosting.server', '=', 'tblservers.id')
            ->join('tblproducts', 'tblhosting.packageid', '=', 'tblproducts.id')
            ->where('tblproducts.servertype', 'external_provider')
            ->where(function ($query) {
                $query->whereNull('tblhosting.subscription_id')
                      ->orWhere('tblhosting.domainstatus', 'Waiting')
                      ->orWhere('tblhosting.sync_status', 'pending');
            })
            ->select('tblhosting.*', 'tblservers.serverusername')
            ->limit($this->batchSize)
            ->get()
            ->toArray();
    }

    /**
     * Sync individual service
     */
    private function syncService(stdClass $service): void
    {
        $response = $this->apiCall('GET', "/v1/services/{$service->id}");

        $syncStatus = $response['status'] ?? 'unknown';
        $nextBilling = $response['next_billing'] ?? null;

        // Update WHMCS service status
        if ($syncStatus === 'active' && $service->domainstatus === 'Waiting') {
            $newStatus = 'Active';
        } elseif ($syncStatus === 'suspended') {
            $newStatus = 'Suspended';
        } elseif ($syncStatus === 'terminated') {
            $newStatus = 'Terminated';
        } else {
            $newStatus = $service->domainstatus;
        }

        Capsule::table('tblhosting')
            ->where('id', $service->id)
            ->update([
                'domainstatus' => $newStatus,
                'nextduedate' => $nextBilling ?? $service->nextduedate,
                'sync_status' => 'synced',
                'last_sync' => date('Y-m-d H:i:s'),
            ]);

        $this->syncLog['synced'][] = $service->id;
    }

    /**
     * Make API call to external service
     */
    private function apiCall(string $method, string $endpoint, array $data = []): array
    {
        $url = rtrim($this->apiEndpoint, '/') . $endpoint;

        $ch = curl_init();
        curl_setopt_array($ch, [
            CURLOPT_URL => $url,
            CURLOPT_RETURNTRANSFER => true,
            CURLOPT_CUSTOMREQUEST => $method,
            CURLOPT_TIMEOUT => 30,
            CURLOPT_HTTPHEADER => [
                'Authorization: Bearer ' . $this->apiKey,
                'Content-Type: application/json',
            ],
        ]);

        if (!empty($data)) {
            curl_setopt($ch, CURLOPT_POSTFIELDS, json_encode($data));
        }

        $response = curl_exec($ch);
        curl_close($ch);

        return json_decode($response, true) ?? [];
    }

    private function logSyncError(stdClass $service, string $error): void
    {
        logActivity("Sync error for service {$service->id}: {$error}");

        Capsule::table('tblhosting')
            ->where('id', $service->id)
            ->update([
                'sync_status' => 'failed',
                'sync_error' => $error,
            ]);

        $this->syncLog['failed'][] = $service->id;
    }
}

// Run the sync
$sync = new ExternalProvisioningSync();
$sync->run();
```

### Step 4: Advanced Cron Task Scheduling

Implement sophisticated cron scheduling:

```php
// crons/advanced/rate_limited_cron.php

#!/usr/bin/php
<?php
/**
 * Rate-Limited Cron Task Execution
 *
 * Ensures cron tasks respect API rate limits and prevents overlaps.
 */

define('ROOTDIR', dirname(__DIR__, 4));
require_once ROOTDIR . '/init.php';

class RateLimitedCronExecutor
{
    private $lockFile;
    private $maxExecutionTime = 300; // 5 minutes
    private $cooldownFile;

    public function __construct(string $taskName)
    {
        $this->lockFile = sys_get_temp_dir() . '/whmcs_cron_' . $taskName . '.lock';
        $this->cooldownFile = sys_get_temp_dir() . '/whmcs_cron_' . $taskName . '_cooldown';
    }

    /**
     * Check if task should run (lock + cooldown)
     */
    public function canRun(): bool
    {
        // Check for existing lock
        if (file_exists($this->lockFile)) {
            $lockData = json_decode(file_get_contents($this->lockFile), true);
            $lockTime = $lockData['time'] ?? 0;

            // Check if lock is stale (> max execution time)
            if (time() - $lockTime < $this->maxExecutionTime) {
                return false;
            }

            // Lock is stale, remove it
            @unlink($this->lockFile);
        }

        // Check cooldown
        if (file_exists($this->cooldownFile)) {
            $cooldownEnd = (int) file_get_contents($this->cooldownFile);
            if (time() < $cooldownEnd) {
                return false;
            }
        }

        return true;
    }

    /**
     * Acquire lock for execution
     */
    public function acquireLock(): bool
    {
        if (!$this->canRun()) {
            return false;
        }

        $lockData = [
            'pid' => getmypid(),
            'time' => time(),
            'host' => gethostname(),
        ];

        return file_put_contents($this->lockFile, json_encode($lockData)) !== false;
    }

    /**
     * Release lock after execution
     */
    public function releaseLock(): void
    {
        @unlink($this->lockFile);
    }

    /**
     * Set cooldown period
     */
    public function setCooldown(int $seconds): void
    {
        file_put_contents($this->cooldownFile, time() + $seconds);
    }

    /**
     * Execute task with lock protection
     */
    public function execute(callable $task): array
    {
        $startTime = microtime(true);

        if (!$this->acquireLock()) {
            return [
                'success' => false,
                'message' => 'Task currently running or in cooldown',
            ];
        }

        try {
            $result = $task();

            $this->release Lock();

            return [
                'success' => true,
                'duration' => round(microtime(true) - $startTime, 2),
                'result' => $result,
            ];
        } catch (\Exception $e) {
            $this->releaseLock();

            return [
                'success' => false,
                'error' => $e->getMessage(),
                'duration' => round(microtime(true) - $startTime, 2),
            ];
        }
    }
}

/**
 * Example: Rate-limited service sync
 */
if (php_sapi_name() === 'cli') {
    $executor = new RateLimitedCronExecutor('external_sync');
    $result = $executor->execute(function() {
        // Your sync logic here
        $sync = new ExternalProvisioningSync();
        $sync->run();

        // Set 5-minute cooldown after successful run
        $executor->setCooldown(300);

        return 'Sync completed';
    });

    echo json_encode($result, JSON_PRETTY_PRINT) . "\n";
}
```

```php
// crons/advanced/parallel_tasks.php

#!/usr/bin/php
<?php
/**
 * Parallel Cron Task Execution
 *
 * Executes multiple cron tasks in parallel using pcntl_fork.
 * Suitable for multi-threaded hosting environments.
 */

define('ROOTDIR', dirname(__DIR__, 4));
require_once ROOTDIR . '/init.php';

class ParallelCronExecutor
{
    private $maxChildren = 5;
    private $tasks = [];
    private $results = [];

    /**
     * Add task to queue
     */
    public function addTask(string $name, callable $task): self
    {
        $this->tasks[$name] = $task;
        return $this;
    }

    /**
     * Execute all tasks in parallel
     */
    public function execute(): array
    {
        $pids = [];
        $startTime = microtime(true);

        foreach ($this->tasks as $name => $task) {
            // Check if we've reached max children
            while (count($pids) >= $this->maxChildren) {
                $this->waitForChild($pids);
            }

            $pid = pcntl_fork();

            if ($pid === -1) {
                // Fork failed, execute synchronously
                $this->results[$name] = $this->executeTask($task);
            } elseif ($pid === 0) {
                // Child process
                $result = $this->executeTask($task);
                exit($result['success'] ? 0 : 1);
            } else {
                // Parent process
                $pids[$pid] = $name;
            }
        }

        // Wait for remaining children
        while (!empty($pids)) {
            $this->waitForChild($pids);
        }

        return [
            'total_duration' => round(microtime(true) - $startTime, 2),
            'results' => $this->results,
        ];
    }

    /**
     * Wait for a child process
     */
    private function waitForChild(array &$pids): void
    {
        $pid = pcntl_waitpid(-1, $status, WNOHANG);

        if ($pid > 0) {
            $name = $pids[$pid] ?? 'unknown';
            $this->results[$name] = [
                'success' => pcntl_wifexited($status) && pcntl_wexitstatus($status) === 0,
                'exit_code' => pcntl_wexitstatus($status),
            ];
            unset($pids[$pid]);
        }
    }

    /**
     * Execute single task
     */
    private function executeTask(callable $task): array
    {
        $startTime = microtime(true);

        try {
            $result = $task();

            return [
                'success' => true,
                'duration' => round(microtime(true) - $startTime, 2),
                'result' => $result,
            ];
        } catch (\Exception $e) {
            return [
                'success' => false,
                'duration' => round(microtime(true) - $startTime, 2),
                'error' => $e->getMessage(),
            ];
        }
    }
}

/**
 * Example usage
 */
if (php_sapi_name() === 'cli' && function_exists('pcntl_fork')) {
    $executor = new ParallelCronExecutor();
    $executor->maxChildren = 3;

    $executor->addTask('cleanup_sessions', function() {
        // Session cleanup logic
        return Capsule::table('tblsessions')
            ->where('created_at', '<', date('Y-m-d H:i:s', time() - 86400));
    });

    $executor->addTask('sync_expiring', function() {
        // Sync expiring services
        return Capsule::table('tblhosting')
            ->where('nextduedate', '<', date('Y-m-d', strtotime('+7 days')));
    });

    $executor->addTask('process_affiliates', function() {
        // Process affiliate commissions
        return Affiliate::processAllCommissions();
    });

    $results = $executor->execute();

    echo json_encode($results, JSON_PRETTY_PRINT) . "\n";
} else {
    // Fallback to sequential execution
    logActivity("Parallel cron not available, using sequential execution");
}
```

### Step 5: Cron Monitoring and Alerts

Implement comprehensive cron monitoring:

```php
// crons/monitoring/cron_health_check.php

#!/usr/bin/php
<?php
/**
 * Cron Health Check and Alerting
 *
 * Monitors cron execution health and sends alerts on failures.
 */

define('ROOTDIR', dirname(__DIR__, 4));
require_once ROOTDIR . '/init.php';

class CronHealthMonitor
{
    private $alertThresholds = [
        'max_execution_time' => 300,    // 5 minutes
        'max_queue_age' => 900,          // 15 minutes
        'max_failed_tasks' => 10,       // 10 failed tasks
        'min_success_rate' => 0.95,     // 95% success rate
    ];

    private $logTable = 'mod_cron_execution_log';
    private $alertHook;

    public function __construct(callable $alertHook = null)
    {
        $this->alertHook = $alertHook;
        $this->ensureLogTableExists();
    }

    /**
     * Log cron execution
     */
    public function logExecution(string $taskName, float $duration, bool $success, string $error = ''): void
    {
        Capsule::table($this->logTable)->insert([
            'task_name' => $taskName,
            'duration' => $duration,
            'success' => $success,
            'error_message' => $error,
            'executed_at' => date('Y-m-d H:i:s'),
            'memory_usage' => memory_get_usage(true),
        ]);
    }

    /**
     * Run health check
     */
    public function checkHealth(): array
    {
        $issues = [];

        // Check for long-running tasks
        $longRunning = Capsule::table($this->logTable)
            ->where('success', 1)
            ->where('duration', '>', $this->alertThresholds['max_execution_time'])
            ->where('executed_at', '>=', date('Y-m-d H:i:s', time() - 3600))
            ->count();

        if ($longRunning > 0) {
            $issues[] = [
                'severity' => 'warning',
                'type' => 'slow_tasks',
                'message' => "{$longRunning} tasks exceeded maximum execution time",
            ];
        }

        // Check queue age
        $staleQueue = $this->getStaleQueueEntries();
        if (!empty($staleQueue)) {
            $issues[] = [
                'severity' => 'critical',
                'type' => 'stale_queue',
                'message' => count($staleQueue) . ' queue entries older than 15 minutes',
                'details' => $staleQueue,
            ];
        }

        // Check success rate
        $successRate = $this->calculateSuccessRate();
        if ($successRate < $this->alertThresholds['min_success_rate']) {
            $issues[] = [
                'severity' => 'warning',
                'type' => 'low_success_rate',
                'message' => sprintf("Success rate (%.1f%%) below threshold (%.1f%%)", $successRate * 100, $this->alertThresholds['min_success_rate'] * 100),
            ];
        }

        // Check failed tasks
        $recentFailures = $this->getRecentFailures();
        if (count($recentFailures) > $this->alertThresholds['max_failed_tasks']) {
            $issues[] = [
                'severity' => 'critical',
                'type' => 'excessive_failures',
                'message' => count($recentFailures) . ' failed tasks in last hour',
                'details' => $recentFailures,
            ];
        }

        // Send alerts if needed
        if (!empty($issues)) {
            $this->sendAlerts($issues);
        }

        return [
            'healthy' => empty($issues),
            'issues' => $issues,
            'checked_at' => date('Y-m-d H:i:s'),
        ];
    }

    /**
     * Get recent task executions
     */
    public function getRecentExecutions(int $limit = 100): array
    {
        return Capsule::table($this->logTable)
            ->where('executed_at', '>=', date('Y-m-d H:i:s', time() - 86400))
            ->orderBy('executed_at', 'desc')
            ->limit($limit)
            ->get()
            ->toArray();
    }

    private function getStaleQueueEntries(): array
    {
        return Capsule::table('tblautomationqueue')
            ->where('created_at', '<', date('Y-m-d H:i:s', time() - $this->alertThresholds['max_queue_age']))
            ->where('status', 'pending')
            ->get()
            ->toArray();
    }

    private function calculateSuccessRate(): float
    {
        $stats = Capsule::table($this->logTable)
            ->where('executed_at', '>=', date('Y-m-d H:i:s', time() - 86400))
            ->selectRaw('SUM(CASE WHEN success = 1 THEN 1 ELSE 0 END) as success_count, COUNT(*) as total_count')
            ->first();

        if ($stats->total_count == 0) {
            return 1.0;
        }

        return $stats->success_count / $stats->total_count;
    }

    private function getRecentFailures(): array
    {
        return Capsule::table($this->logTable)
            ->where('success', 0)
            ->where('executed_at', '>=', date('Y-m-d H:i:s', time() - 3600))
            ->orderBy('executed_at', 'desc')
            ->get()
            ->toArray();
    }

    private function sendAlerts(array $issues): void
    {
        if ($this->alertHook) {
            ($this->alertHook)($issues);
        }

        // Log to WHMCS activity log
        foreach ($issues as $issue) {
            logActivity("Cron Health Alert [{$issue['severity']}]: {$issue['message']}");
        }
    }

    private function ensureLogTableExists(): void
    {
        if (!Capsule::schema()->hasTable($this->logTable)) {
            Capsule::schema()->create($this->logTable, function($table) {
                $table->increments('id');
                $table->string('task_name', 100);
                $table->decimal('duration', 10, 2);
                $table->boolean('success');
                $table->text('error_message')->nullable();
                $table->timestamp('executed_at')->useCurrent();
                $table->integer('memory_usage')->nullable();
                $table->index(['executed_at', 'task_name']);
            });
        }
    }
}

/**
 * Monitor wrapper for cron execution
 */
class CronMonitoredExecutor
{
    private $monitor;

    public function __construct()
    {
        $this->monitor = new CronHealthMonitor();
    }

    /**
     * Execute task with monitoring
     */
    public function execute(string $taskName, callable $task): void
    {
        $startTime = microtime(true);

        $success = false;
        $error = '';

        try {
            $task();
            $success = true;
        } catch (\Exception $e) {
            $error = $e->getMessage();
        }

        $duration = microtime(true) - $startTime;
        $this->monitor->logExecution($taskName, $duration, $success, $error);
    }
}
```

## Cron Schedule Best Practices

```
Recommended Cron Schedule:

*/5 * * * *     - Main WHMCS cron (Invoice processing, module queue, etc.)
0 * * * *       - Hourly task checks
0 6 * * *       - Daily tasks (Invoice generation, backups)
0 3 * * *       - Nightly cleanup (off-peak hours)
0 0 1 * *       - Monthly tasks (Month-end processing)

Task Grouping:
- Critical (every 5 min): Payment processing, provisioning sync
- Important (hourly): Email queue, ticket escalations
- Standard (daily): Invoice generation, SSL checks
- Cleanup (nightly): Database cleanup, log rotation
```

## Verification Checklist

```
Cron Configuration:
□ Main cron running every 5 minutes
□ PHP CLI binary path correct
□ Cron user has proper permissions
□ Timeout settings adequate
□ Output redirection configured

Task Scheduling:
□ Invoice generation at off-peak hours
□ Cleanup tasks at night
□ Sync tasks within acceptable intervals
□ No task overlaps in schedule

Monitoring:
□ Execution logging enabled
□ Failure alerting configured
□ Health checks running
□ Dashboard metrics available

Performance:
□ Execution times within limits
□ Batch sizes appropriate
□ Indexes on log tables
□ Old records cleaned up periodically
```

## Common Pitfalls to Avoid

1. **Missing main cron** - Critical for WHMCS operation
2. **Too frequent execution** - Can cause database load
3. **Not checking overlaps** - Multiple cron instances can cause issues
4. **Insufficient timeout** - Long-running tasks may be killed
5. **Ignoring failures** - Track and alert on errors
6. **Large batch sizes** - Can cause memory issues
7. **No cleanup** - Logs grow unbounded
8. **Wrong user permissions** - Cron may fail to write files
9. **Peak hour scheduling** - Run heavy tasks off-peak
10. **Missing logging** - Cannot troubleshoot without logs

## WHMCS ClassDocs References

- [logActivity()](https://developers.whmcs.com/advanced/logging/) - Activity logging
- [Capsule](https://developers.whmcs.com/pdo-wrapper/) - Database operations
- [DI facade](https://developers.whmcs.com/advanced/dependency-injection/) - Dependency injection
- [Cron configuration](https://developers.whmcs.com/advanced/cron-job-configuration/) - Cron docs
