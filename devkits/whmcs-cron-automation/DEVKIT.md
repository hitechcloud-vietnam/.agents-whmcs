# WHMCS Cron Automation DevKit
# Version: 1.0 | Updated: 2026-05-28

## DevKit Structure

```
devkits/whmcs-cron-automation/
├── cron-automation.php     # Main module
├── lib/
│   ├── CronScheduler.php    # Task scheduling
│   ├── TaskRunner.php       # Task execution
│   └── TaskRegistry.php     # Task registration
├── tasks/
│   ├── CleanupTask.php      # Cleanup tasks
│   ├── ReportTask.php       # Report generation
│   ├── SyncTask.php         # Data synchronization
│   └── NotificationTask.php # Notification sending
└── templates/
    └── cron-config.tpl      # Admin configuration
```

## Main Cron Automation Module

```php
<?php
/**
 * WHMCS Cron Automation Module
 * DevKit Template
 * 
 * Advanced cron job automation with task scheduling
 */

if (!defined("WHMCS")) {
    die("Direct access denied");
}

use WHMCS\Database\Capsule;

/**
 * Config function
 */
function {module}_config(): array {
    return [
        'name' => '{Cron Automation}',
        'description' => 'Advanced cron job automation module',
        'version' => '1.0',
        'author' => '{Author}',
    ];
}

/**
 * Activate
 */
function {module}_activate(): array {
    Capsule::schema()->create('mod_{module}_schedules', function($t) {
        $t->increments('id');
        $t->string('task_name');
        $t->string('task_class');
        $t->string('schedule'); // cron expression
        $t->text('config');
        $t->boolean('is_active');
        $t->timestamp('last_run')->nullable();
        $t->timestamp('next_run')->nullable();
        $t->integer('max_runtime')->default(300);
        $t->integer('retry_count')->default(0);
        $t->string('status')->default('idle');
    });
    
    Capsule::schema()->create('mod_{module}_executions', function($t) {
        $t->increments('id');
        $t->integer('schedule_id');
        $t->string('status'); // running, completed, failed, timeout
        $t->text('output');
        $t->text('error');
        $t->float('duration');
        $t->timestamp('started_at');
        $t->timestamp('completed_at')->nullable();
    });
    
    Capsule::schema()->create('mod_{module}_logs', function($t) {
        $t->increments('id');
        $t->string('level'); // debug, info, warning, error
        $t->string('task');
        $t->text('message');
        $t->timestamp('created_at');
    });
    
    // Register default tasks
    {module}_registerDefaultTasks();
    
    return ['status' => 'success', 'description' => 'Cron Automation activated'];
}

/**
 * Deactivate
 */
function {module}_deactivate(): array {
    Capsule::schema()->dropIfExists('mod_{module}_schedules');
    Capsule::schema()->dropIfExists('mod_{module}_executions');
    Capsule::schema()->dropIfExists('mod_{module}_logs');
    
    return ['status' => 'success'];
}

/**
 * Register default tasks
 */
function {module}_registerDefaultTasks(): void {
    $tasks = [
        [
            'name' => 'Cleanup Expired Sessions',
            'class' => 'CleanupExpiredSessionsTask',
            'schedule' => '0 */6 * * *', // Every 6 hours
            'config' => json_encode(['max_age' => 86400]),
        ],
        [
            'name' => 'Send Invoice Reminders',
            'class' => 'InvoiceReminderTask',
            'schedule' => '0 9 * * *', // Daily at 9 AM
            'config' => json_encode(['days_before' => [3, 7, 14]]),
        ],
        [
            'name' => 'Sync Domain Status',
            'class' => 'DomainSyncTask',
            'schedule' => '0 */4 * * *', // Every 4 hours
            'config' => json_encode([]),
        ],
        [
            'name' => 'Generate Daily Report',
            'class' => 'DailyReportTask',
            'schedule' => '0 0 * * *', // Daily at midnight
            'config' => json_encode(['recipients' => []]),
        ],
        [
            'name' => 'Cleanup Temp Files',
            'class' => 'CleanupTempFilesTask',
            'schedule' => '0 3 * * *', // Daily at 3 AM
            'config' => json_encode(['max_age' => 172800]),
        ],
    ];
    
    foreach ($tasks as $task) {
        Capsule::table('mod_{module}_schedules')->insert([
            'task_name' => $task['name'],
            'task_class' => $task['class'],
            'schedule' => $task['schedule'],
            'config' => $task['config'],
            'is_active' => 1,
            'next_run' => {module}_calculateNextRun($task['schedule']),
        ]);
    }
}

/**
 * Calculate next run time from cron expression
 */
function {module}_calculateNextRun(string $schedule): string {
    // Simple implementation - use cron library in production
    $parts = preg_split('/\s+/', $schedule);
    
    if (count($parts) < 5) {
        return date('Y-m-d H:i:s', strtotime('+1 hour'));
    }
    
    // Basic cron parsing
    $now = time();
    $next = $now;
    
    // Just advance by hour for simple schedules
    $next = strtotime('+1 hour', $next);
    
    return date('Y-m-d H:i:s', $next);
}

/**
 * Run cron tasks (called by WHMCS cron)
 */
function {module}_cron(): void {
    $runner = new \CronAutomation\TaskRunner();
    $executed = $runner->runDueTasks();
    
    logActivity("{Module}: Executed {$executed} tasks");
}

/**
 * Output function (Admin Interface)
 */
function {module}_output(array $vars): void {
    $action = $_REQUEST['action'] ?? 'dashboard';
    
    if ($_SERVER['REQUEST_METHOD'] === 'POST') {
        check_token('WHMCS.admin.default');
    }
    
    switch ($action) {
        case 'schedules':
            {module}_manageSchedules();
            break;
        case 'executions':
            {module}_viewExecutions();
            break;
        case 'logs':
            {module}_viewLogs();
            break;
        case 'run':
            {module}_runTask();
            break;
        default:
            {module}_showDashboard();
    }
}
```

## Task Runner

```php
<?php
/**
 * Cron Task Runner
 */

namespace CronAutomation;

use WHMCS\Database\Capsule;

class TaskRunner {
    
    private int $maxConcurrent = 3;
    private array $runningTasks = [];
    
    /**
     * Run all tasks that are due
     */
    public function runDueTasks(): int {
        $dueTasks = Capsule::table('mod_{module}_schedules')
            ->where('is_active', 1)
            ->where(function($query) {
                $query->whereNull('next_run')
                      ->orWhere('next_run', '<=', date('Y-m-d H:i:s'));
            })
            ->where('status', '!=', 'running')
            ->get();
        
        $executed = 0;
        
        foreach ($dueTasks as $schedule) {
            if (count($this->runningTasks) >= $this->maxConcurrent) {
                break; // Don't start more than max concurrent
            }
            
            if ($this->runTask($schedule)) {
                $executed++;
            }
        }
        
        return $executed;
    }
    
    /**
     * Run a specific task
     */
    public function runTask($schedule): bool {
        $taskClass = $schedule->task_class;
        
        // Check if task class exists
        if (!class_exists($taskClass)) {
            $taskClass = "\\CronAutomation\\Tasks\\{$taskClass}";
        }
        
        if (!class_exists($taskClass)) {
            $this->log('error', $schedule->task_name, "Task class not found: {$taskClass}");
            return false;
        }
        
        // Update status to running
        Capsule::table('mod_{module}_schedules')
            ->where('id', $schedule->id)
            ->update([
                'status' => 'running',
                'last_run' => date('Y-m-d H:i:s'),
            ]);
        
        // Create execution record
        $executionId = Capsule::table('mod_{module}_executions')->insertGetId([
            'schedule_id' => $schedule->id,
            'status' => 'running',
            'started_at' => date('Y-m-d H:i:s'),
        ]);
        
        $startTime = microtime(true);
        $output = '';
        $error = '';
        $status = 'completed';
        
        try {
            $task = new $taskClass();
            $config = json_decode($schedule->config, true) ?? [];
            
            $result = $task->execute($config);
            
            $output = is_array($result) ? json_encode($result) : (string) $result;
            
            // Reset retry count on success
            Capsule::table('mod_{module}_schedules')
                ->where('id', $schedule->id)
                ->update([
                    'status' => 'idle',
                    'retry_count' => 0,
                    'next_run' => {module}_calculateNextRun($schedule->schedule),
                ]);
            
            $this->log('info', $schedule->task_name, "Task completed successfully");
            
        } catch (\Exception $e) {
            $error = $e->getMessage();
            $status = 'failed';
            
            // Increment retry count
            Capsule::table('mod_{module}_schedules')
                ->where('id', $schedule->id)
                ->update([
                    'status' => 'idle',
                    'retry_count' => $schedule->retry_count + 1,
                ]);
            
            $this->log('error', $schedule->task_name, "Task failed: {$error}");
        }
        
        // Update execution record
        $duration = microtime(true) - $startTime;
        
        Capsule::table('mod_{module}_executions')
            ->where('id', $executionId)
            ->update([
                'status' => $status,
                'output' => substr($output, 0, 65535),
                'error' => substr($error, 0, 65535),
                'duration' => round($duration, 3),
                'completed_at' => date('Y-m-d H:i:s'),
            ]);
        
        return $status === 'completed';
    }
    
    /**
     * Force run a task
     */
    public function forceRun(int $scheduleId): bool {
        $schedule = Capsule::table('mod_{module}_schedules')
            ->where('id', $scheduleId)
            ->first();
        
        if (!$schedule) {
            return false;
        }
        
        return $this->runTask($schedule);
    }
    
    /**
     * Get running tasks
     */
    public function getRunningTasks(): array {
        return Capsule::table('mod_{module}_executions')
            ->where('status', 'running')
            ->get()
            ->toArray();
    }
    
    /**
     * Log message
     */
    private function log(string $level, string $task, string $message): void {
        Capsule::table('mod_{module}_logs')->insert([
            'level' => $level,
            'task' => $task,
            'message' => $message,
            'created_at' => date('Y-m-d H:i:s'),
        ]);
    }
}
```

## Task Registry

```php
<?php
/**
 * Task Registry
 * Manages available cron tasks
 */

namespace CronAutomation;

use WHMCS\Database\Capsule;

class TaskRegistry {
    
    private static array $tasks = [];
    
    /**
     * Register a task
     */
    public static function register(string $name, string $class, string $description = ''): void {
        self::$tasks[$name] = [
            'class' => $class,
            'description' => $description,
        ];
    }
    
    /**
     * Get all registered tasks
     */
    public static function all(): array {
        return self::$tasks;
    }
    
    /**
     * Get task by name
     */
    public static function get(string $name): ?array {
        return self::$tasks[$name] ?? null;
    }
    
    /**
     * Check if task exists
     */
    public static function exists(string $name): bool {
        return isset(self::$tasks[$name]);
    }
    
    /**
     * Initialize default tasks
     */
    public static function initDefaults(): void {
        self::register(
            'cleanup_expired_sessions',
            'CleanupExpiredSessionsTask',
            'Clean up expired user sessions'
        );
        
        self::register(
            'invoice_reminder',
            'InvoiceReminderTask',
            'Send invoice payment reminders'
        );
        
        self::register(
            'domain_sync',
            'DomainSyncTask',
            'Synchronize domain status with registrars'
        );
        
        self::register(
            'daily_report',
            'DailyReportTask',
            'Generate daily activity report'
        );
        
        self::register(
            'cleanup_temp_files',
            'CleanupTempFilesTask',
            'Clean up temporary files'
        );
        
        self::register(
            'backup_database',
            'BackupDatabaseTask',
            'Create database backup'
        );
        
        self::register(
            'sync_products',
            'SyncProductsTask',
            'Synchronize product catalog'
        );
    }
}

/**
 * Base Task Class
 */
abstract class BaseTask {
    
    protected array $config = [];
    
    public function __construct(array $config = []) {
        $this->config = $config;
    }
    
    /**
     * Execute the task
     */
    abstract public function execute(array $config): mixed;
    
    /**
     * Get task name
     */
    public function getName(): string {
        return static::class;
    }
    
    /**
     * Validate configuration
     */
    protected function validateConfig(array $required): bool {
        foreach ($required as $key) {
            if (!isset($this->config[$key])) {
                throw new \Exception("Missing required config: {$key}");
            }
        }
        return true;
    }
    
    /**
     * Log activity
     */
    protected function log(string $level, string $message): void {
        Capsule::table('mod_{module}_logs')->insert([
            'level' => $level,
            'task' => $this->getName(),
            'message' => $message,
            'created_at' => date('Y-m-d H:i:s'),
        ]);
    }
}
```

## Task Implementations

```php
<?php
/**
 * Cleanup Expired Sessions Task
 */

namespace CronAutomation\Tasks;

use CronAutomation\BaseTask;
use WHMCS\Database\Capsule;

class CleanupExpiredSessionsTask extends BaseTask {
    
    public function execute(array $config): array {
        $maxAge = $config['max_age'] ?? 86400; // 24 hours default
        $cutoffTime = date('Y-m-d H:i:s', time() - $maxAge);
        
        // Clean up WHMCS sessions
        $sessionsDeleted = Capsule::table('tblloginlog')
            ->where('lastvisit', '<', $cutoffTime)
            ->delete();
        
        // Clean up module sessions
        $moduleSessionsDeleted = Capsule::table('mod_{module}_sso_sessions')
            ->where('expires_at', '<', date('Y-m-d H:i:s'))
            ->delete();
        
        $result = [
            'sessions_deleted' => $sessionsDeleted,
            'module_sessions_deleted' => $moduleSessionsDeleted,
            'cutoff_time' => $cutoffTime,
        ];
        
        $this->log('info', "Cleaned up {$sessionsDeleted} sessions");
        
        return $result;
    }
}

/**
 * Invoice Reminder Task
 */

namespace CronAutomation\Tasks;

use CronAutomation\BaseTask;
use WHMCS\Database\Capsule;

class InvoiceReminderTask extends BaseTask {
    
    public function execute(array $config): array {
        $daysBefore = $config['days_before'] ?? [3, 7, 14];
        $sent = [];
        
        foreach ($daysBefore as $days) {
            $targetDate = date('Y-m-d', strtotime("+{$days} days"));
            
            // Find invoices due on target date
            $invoices = Capsule::table('tblinvoices')
                ->where('duedate', $targetDate)
                ->whereIn('status', ['Unpaid', 'Overdue'])
                ->get();
            
            foreach ($invoices as $invoice) {
                // Check if reminder already sent
                $existing = Capsule::table('tblactivitylog')
                    ->where('description', 'LIKE', "%Invoice Reminder Sent: {$invoice->id}%")
                    ->first();
                
                if ($existing) {
                    continue;
                }
                
                // Send reminder
                send_msg([
                    'id' => $invoice->id,
                    'type' => 'invoice',
                    'send_to_name' => $invoice->firstname . ' ' . $invoice->lastname,
                    'send_to_email' => $invoice->email,
                ]);
                
                logActivity("Invoice reminder sent for Invoice #{$invoice->id}");
                
                $sent[] = $invoice->id;
            }
        }
        
        return [
            'reminders_sent' => count($sent),
            'invoice_ids' => $sent,
        ];
    }
}

/**
 * Domain Sync Task
 */

namespace CronAutomation\Tasks;

use CronAutomation\BaseTask;
use WHMCS\Database\Capsule;

class DomainSyncTask extends BaseTask {
    
    public function execute(array $config): array {
        $synced = 0;
        $failed = 0;
        
        // Get domains pending sync
        $domains = Capsule::table('tbldomains')
            ->where('status', '!=', 'Expired')
            ->limit(100)
            ->get();
        
        foreach ($domains as $domain) {
            try {
                // Call registrar sync (example)
                $result = Capsule::table('tbldomains')
                    ->where('id', $domain->id)
                    ->update([
                        'nextduedate' => date('Y-m-d', strtotime('+1 year')),
                        'updated_at' => date('Y-m-d H:i:s'),
                    ]);
                
                $synced++;
                
            } catch (\Exception $e) {
                $failed++;
                $this->log('warning', "Failed to sync domain {$domain->domain}: {$e->getMessage()}");
            }
        }
        
        return [
            'synced' => $synced,
            'failed' => $failed,
        ];
    }
}

/**
 * Daily Report Task
 */

namespace CronAutomation\Tasks;

use CronAutomation\BaseTask;
use WHMCS\Database\Capsule;

class DailyReportTask extends BaseTask {
    
    public function execute(array $config): array {
        $today = date('Y-m-d');
        $yesterday = date('Y-m-d', strtotime('-1 day'));
        
        // Collect statistics
        $stats = [
            'new_clients' => Capsule::table('tblclients')
                ->whereDate('created_at', $yesterday)
                ->count(),
            
            'new_orders' => Capsule::table('tblorders')
                ->whereDate('date', $yesterday)
                ->count(),
            
            'new_services' => Capsule::table('tblhosting')
                ->whereDate('regdate', $yesterday)
                ->count(),
            
            'invoices_raised' => Capsule::table('tblinvoices')
                ->whereDate('duedate', $yesterday)
                ->count(),
            
            'invoices_paid' => Capsule::table('tblinvoices')
                ->whereDate('datepaid', $yesterday)
                ->count(),
            
            'tickets_opened' => Capsule::table('tbltickets')
                ->whereDate('created', $yesterday)
                ->count(),
        ];
        
        // Calculate income
        $stats['income'] = Capsule::table('tblinvoices')
            ->whereDate('datepaid', $yesterday)
            ->where('status', 'Paid')
            ->sum('total');
        
        // Store report
        Capsule::table('mod_{module}_reports')->insert([
            'report_date' => $yesterday,
            'stats' => json_encode($stats),
            'created_at' => date('Y-m-d H:i:s'),
        ]);
        
        // Send email report if configured
        if (!empty($config['recipients'])) {
            $this->sendReportEmail($stats, $config['recipients']);
        }
        
        return $stats;
    }
    
    private function sendReportEmail(array $stats, array $recipients): void {
        $body = "Daily Report - " . date('Y-m-d', strtotime('-1 day')) . "\n\n";
        $body .= "New Clients: {$stats['new_clients']}\n";
        $body .= "New Orders: {$stats['new_orders']}\n";
        $body .= "Income: $" . number_format($stats['income'], 2) . "\n";
        
        foreach ($recipients as $email) {
            send_msg([
                'id' => 0,
                'type' => 'general',
                'send_to_email' => $email,
                'subject' => 'Daily WHMCS Report',
                'message' => $body,
            ]);
        }
    }
}

/**
 * Cleanup Temp Files Task
 */

namespace CronAutomation\Tasks;

use CronAutomation\BaseTask;

class CleanupTempFilesTask extends BaseTask {
    
    public function execute(array $config): array {
        $maxAge = $config['max_age'] ?? 172800; // 48 hours default
        $tempDir = ini_get('upload_tmp_dir') ?: sys_get_temp_dir();
        $cleaned = 0;
        $spaceFreed = 0;
        
        $files = glob($tempDir . '/whmcs_*');
        
        foreach ($files as $file) {
            if (is_file($file)) {
                $age = time() - filemtime($file);
                
                if ($age > $maxAge) {
                    $size = filesize($file);
                    unlink($file);
                    $cleaned++;
                    $spaceFreed += $size;
                }
            }
        }
        
        $this->log('info', "Cleaned {$cleaned} temp files, freed " . number_format($spaceFreed / 1024, 2) . " KB");
        
        return [
            'files_cleaned' => $cleaned,
            'space_freed' => $spaceFreed,
        ];
    }
}
```

## Admin Configuration Template

```smarty
<div class="cron-automation">
    <h2>Cron Automation</h2>
    
    <div class="row">
        <div class="col-md-12">
            <div class="alert alert-info">
                <p><strong>Cron Command:</strong></p>
                <code>php -q /path/to/whmcs/crons/cron.php --module={module}</code>
                <p class="mt-2">Add to your server crontab to execute scheduled tasks.</p>
            </div>
        </div>
    </div>
    
    <div class="panel panel-default">
        <div class="panel-heading">
            <h3 class="panel-title">Scheduled Tasks</h3>
            <a href="?module={module}&action=schedules&sub=add" class="btn btn-success btn-sm pull-right">
                <i class="fa fa-plus"></i> Add Task
            </a>
        </div>
        <div class="panel-body">
            <table class="table table-striped">
                <thead>
                    <tr>
                        <th>Task Name</th>
                        <th>Schedule</th>
                        <th>Last Run</th>
                        <th>Next Run</th>
                        <th>Status</th>
                        <th>Actions</th>
                    </tr>
                </thead>
                <tbody>
                    {foreach $schedules as $schedule}
                    <tr>
                        <td>{$schedule.task_name}</td>
                        <td><code>{$schedule.schedule}</code></td>
                        <td>{$schedule.last_run}</td>
                        <td>{$schedule.next_run}</td>
                        <td>
                            {if $schedule.is_active}
                                <span class="label label-success">Active</span>
                            {else}
                                <span class="label label-default">Inactive</span>
                            {/if}
                        </td>
                        <td>
                            <a href="?module={module}&action=run&id={$schedule.id}" class="btn btn-xs btn-primary">
                                <i class="fa fa-play"></i> Run
                            </a>
                            <a href="?module={module}&action=schedules&sub=edit&id={$schedule.id}" class="btn btn-xs">
                                <i class="fa fa-edit"></i>
                            </a>
                        </td>
                    </tr>
                    {/foreach}
                </tbody>
            </table>
        </div>
    </div>
    
    <div class="panel panel-default">
        <div class="panel-heading">
            <h3 class="panel-title">Recent Executions</h3>
        </div>
        <div class="panel-body">
            <table class="table">
                <thead>
                    <tr>
                        <th>Task</th>
                        <th>Status</th>
                        <th>Duration</th>
                        <th>Started</th>
                    </tr>
                </thead>
                <tbody>
                    {foreach $executions as $execution}
                    <tr>
                        <td>{$execution.task_name}</td>
                        <td>
                            <span class="label label-{$execution.status_class}">{$execution.status}</span>
                        </td>
                        <td>{$execution.duration}ms</td>
                        <td>{$execution.started_at}</td>
                    </tr>
                    {/foreach}
                </tbody>
            </table>
        </div>
    </div>
</div>
```

## Cron Hook Registration

```php
<?php
/**
 * Register cron tasks with WHMCS
 */

if (!defined("WHMCS")) {
    die("Direct access denied");
}

// Run cron automation tasks
add_hook('DailyCronJob', 1, function($vars) {
    $runner = new \CronAutomation\TaskRunner();
    $executed = $runner->runDueTasks();
    
    logActivity("Cron Automation: Executed {$executed} tasks");
});

// Hourly task execution
add_hook('HourlyCronJob', 1, function($vars) {
    $runner = new \CronAutomation\TaskRunner();
    $runner->runDueTasks();
});
```

## Checklist

```
Pre-Dev:
□ Define tasks to automate
□ Plan execution schedules
□ Design retry strategy
□ Identify task dependencies

Development:
□ Create main module with tables
□ Implement TaskRunner class
□ Implement TaskRegistry class
□ Create BaseTask class
□ Implement cleanup tasks
□ Implement report tasks
□ Implement sync tasks
□ Add cron hook registration
□ Build admin UI
□ Add execution logging

Testing:
□ Test task execution
□ Verify schedule timing
□ Test retry mechanism
□ Verify error handling
□ Check logging
□ Test concurrent execution
```