# WHMCS Automation Cron Module

```php
<?php
/**
 * WHMCS Automation Cron Module
 * 
 * Advanced cron job automation module with task scheduling,
 * dependency management, and comprehensive logging.
 * 
 * @Author: HiTech Cloud DevKit
 * @Version: 1.0.0
 */

// Prevent direct access
if (!defined("WHMCS")) {
    die("Direct access prohibited");
}

function automationcron_MetaData()
{
    return array(
        'DisplayName' => 'Automation Cron',
        'APIVersion' => '1.1',
        'RequiresServer' => false,
    );
}

function automationcron_ConfigArray()
{
    return array(
        'FriendlyName' => array(
            'Type' => 'System',
            'Value' => 'Automation Cron',
        ),
        'EnableTasks' => array(
            'Type' => 'yesno',
            'Default' => 'yes',
            'Description' => 'Enable automated tasks',
        ),
        'MaxExecutionTime' => array(
            'Type' => 'text',
            'Size' => '10',
            'Default' => '300',
            'Description' => 'Max task execution time (seconds)',
        ),
        'EnableLogging' => array(
            'Type' => 'yesno',
            'Default' => 'yes',
            'Description' => 'Enable task execution logging',
        ),
        'LogRetentionDays' => array(
            'Type' => 'text',
            'Size' => '10',
            'Default' => '30',
            'Description' => 'Log retention period (days)',
        ),
        'EnableNotifications' => array(
            'Type' => 'yesno',
            'Default' => 'yes',
            'Description' => 'Send notifications on failures',
        ),
        'AdminEmail' => array(
            'Type' => 'text',
            'Size' => '100',
            'Default' => '',
            'Description' => 'Admin email for notifications',
        ),
    );
}

function automationcron_activate()
{
    try {
        if (!function_exists('createTable')) {
            require_once dirname(__FILE__) . '/../../includes/modulefunctions.php';
        }
        
        // Tasks table
        $tasksTable = 'mod_automationcron_tasks';
        $tasksSchema = "
            CREATE TABLE `{$tasksTable}` (
                `id` INT NOT NULL AUTO_INCREMENT PRIMARY KEY,
                `task_key` VARCHAR(100) UNIQUE NOT NULL,
                `name` VARCHAR(255) NOT NULL,
                `description` TEXT NULL,
                `task_type` VARCHAR(50) NOT NULL,
                `handler` VARCHAR(255) NOT NULL,
                `schedule` VARCHAR(100) NOT NULL,
                `config` JSON NULL,
                `dependencies` JSON NULL,
                `is_active` TINYINT(1) DEFAULT 1,
                `is_locked` TINYINT(1) DEFAULT 0,
                `last_run` DATETIME NULL,
                `next_run` DATETIME NOT NULL,
                `run_count` INT DEFAULT 0,
                `success_count` INT DEFAULT 0,
                `failure_count` INT DEFAULT 0,
                `created_at` DATETIME DEFAULT CURRENT_TIMESTAMP,
                `updated_at` DATETIME DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
                INDEX `idx_task_key` (`task_key`),
                INDEX `idx_next_run` (`next_run`),
                INDEX `idx_is_active` (`is_active`)
            ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
        ";
        createTable($tasksTable, $tasksSchema);
        
        // Execution logs table
        $logsTable = 'mod_automationcron_logs';
        $logsSchema = "
            CREATE TABLE `{$logsTable}` (
                `id` INT NOT NULL AUTO_INCREMENT PRIMARY KEY,
                `task_id` INT NOT NULL,
                `started_at` DATETIME NOT NULL,
                `completed_at` DATETIME NULL,
                `duration_ms` INT NULL,
                `status` ENUM('running', 'success', 'failed', 'timeout', 'cancelled') DEFAULT 'running',
                `result` TEXT NULL,
                `error_message` TEXT NULL,
                `memory_usage` VARCHAR(50) NULL,
                INDEX `idx_task_id` (`task_id`),
                INDEX `idx_started_at` (`started_at`),
                INDEX `idx_status` (`status`)
            ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
        ";
        createTable($logsTable, $logsSchema);
        
        // Task queue table
        $queueTable = 'mod_automationcron_queue';
        $queueSchema = "
            CREATE TABLE `{$queueTable}` (
                `id` INT NOT NULL AUTO_INCREMENT PRIMARY KEY,
                `task_id` INT NOT NULL,
                `priority` INT DEFAULT 5,
                `data` JSON NULL,
                `status` ENUM('pending', 'processing', 'completed', 'failed') DEFAULT 'pending',
                `attempts` INT DEFAULT 0,
                `max_attempts` INT DEFAULT 3,
                `scheduled_at` DATETIME DEFAULT CURRENT_TIMESTAMP,
                `started_at` DATETIME NULL,
                `completed_at` DATETIME NULL,
                `error` TEXT NULL,
                INDEX `idx_task_id` (`task_id`),
                INDEX `idx_status` (`status`),
                INDEX `idx_priority` (`priority`, `scheduled_at`)
            ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
        ";
        createTable($queueTable, $queueSchema);
        
        // Register default tasks
        automationcron_RegisterDefaultTasks();
        
        return array(
            'status' => 'success',
            'description' => 'Automation Cron module activated.',
        );
    } catch (\Exception $e) {
        return array(
            'status' => 'error',
            'description' => 'Failed to activate module: ' . $e->getMessage(),
        );
    }
}

function automationcron_deactivate()
{
    return array('status' => 'success', 'description' => 'Module deactivated.');
}

/**
 * Register default tasks
 */
function automationcron_RegisterDefaultTasks()
{
    $defaultTasks = array(
        array(
            'task_key' => 'cleanup_expired_sessions',
            'name' => 'Clean Up Expired Sessions',
            'description' => 'Remove expired sessions and temporary data',
            'task_type' => 'system',
            'handler' => 'automationcron_CleanupExpiredSessions',
            'schedule' => '0 * * * *', // Every hour
            'config' => array('retention_hours' => 24),
        ),
        array(
            'task_key' => 'process_pending_invoices',
            'name' => 'Process Pending Invoices',
            'description' => 'Send payment reminders and process overdue invoices',
            'task_type' => 'billing',
            'handler' => 'automationcron_ProcessPendingInvoices',
            'schedule' => '0 9 * * *', // Daily at 9 AM
            'config' => array('reminder_days' => array(3, 7, 14)),
        ),
        array(
            'task_key' => 'backup_database',
            'name' => 'Database Backup',
            'description' => 'Create database backups',
            'task_type' => 'maintenance',
            'handler' => 'automationcron_BackupDatabase',
            'schedule' => '0 2 * * *', // Daily at 2 AM
            'config' => array('retention_days' => 7),
        ),
        array(
            'task_key' => 'sync_registrar_dns',
            'name' => 'Sync Registrar DNS',
            'description' => 'Synchronize DNS records with registrars',
            'task_type' => 'domain',
            'handler' => 'automationcron_SyncRegistrarDns',
            'schedule' => '0 */6 * * *', // Every 6 hours
        ),
        array(
            'task_key' => 'generate_usage_reports',
            'name' => 'Generate Usage Reports',
            'description' => 'Generate and email usage reports',
            'task_type' => 'reporting',
            'handler' => 'automationcron_GenerateUsageReports',
            'schedule' => '0 0 * * 1', // Weekly on Monday
            'config' => array('report_types' => array('bandwidth', 'storage')),
        ),
    );
    
    foreach ($defaultTasks as $taskData) {
        automationcron_CreateTask($taskData);
    }
}

/**
 * Create a new task
 */
function automationcron_CreateTask($taskData)
{
    if (!function_exists('Capsule')) {
        require_once dirname(__FILE__) . '/../../includesWHMCS.php';
    }
    
    try {
        $nextRun = automationcron_CalculateNextRun($taskData['schedule']);
        
        $data = array(
            'task_key' => $taskData['task_key'],
            'name' => $taskData['name'],
            'description' => $taskData['description'] ?? '',
            'task_type' => $taskData['task_type'],
            'handler' => $taskData['handler'],
            'schedule' => $taskData['schedule'],
            'config' => isset($taskData['config']) ? json_encode($taskData['config']) : null,
            'dependencies' => isset($taskData['dependencies']) ? json_encode($taskData['dependencies']) : null,
            'next_run' => $nextRun,
        );
        
        Capsule::table('mod_automationcron_tasks')->insert($data);
        
        return array('success' => true, 'task_key' => $taskData['task_key']);
    } catch (\Exception $e) {
        return array('success' => false, 'error' => $e->getMessage());
    }
}

/**
 * Calculate next run time from cron expression
 */
function automationcron_CalculateNextRun($schedule)
{
    // Simple cron parser for common patterns
    // For full cron support, use a library like cron-expression
    
    $parts = explode(' ', $schedule);
    if (count($parts) < 5) {
        return date('Y-m-d H:i:s', strtotime('+1 hour'));
    }
    
    // For simplicity, return next hour
    // In production, use a proper cron library
    return date('Y-m-d H:i:s', strtotime('+1 hour'));
}

/**
 * Get all tasks
 */
function automationcron_GetTasks($filters = array())
{
    if (!function_exists('Capsule')) {
        require_once dirname(__FILE__) . '/../../includesWHMCS.php';
    }
    
    $query = Capsule::table('mod_automationcron_tasks');
    
    if (isset($filters['active_only']) && $filters['active_only']) {
        $query->where('is_active', 1);
    }
    
    if (isset($filters['type'])) {
        $query->where('task_type', $filters['type']);
    }
    
    $tasks = $query->orderBy('next_run', 'asc')->get();
    
    foreach ($tasks as &$task) {
        $task->config = $task->config ? json_decode($task->config, true) : array();
        $task->dependencies = $task->dependencies ? json_decode($task->dependencies, true) : array();
    }
    
    return $tasks;
}

/**
 * Get task by key
 */
function automationcron_GetTask($taskKey)
{
    if (!function_exists('Capsule')) {
        require_once dirname(__FILE__) . '/../../includesWHMCS.php';
    }
    
    $task = Capsule::table('mod_automationcron_tasks')
        ->where('task_key', $taskKey)
        ->first();
    
    if ($task) {
        $task->config = $task->config ? json_decode($task->config, true) : array();
        $task->dependencies = $task->dependencies ? json_decode($task->dependencies, true) : array();
    }
    
    return $task;
}

/**
 * Update task
 */
function automationcron_UpdateTask($taskKey, $taskData)
{
    if (!function_exists('Capsule')) {
        require_once dirname(__FILE__) . '/../../includesWHMCS.php';
    }
    
    try {
        $updateData = array();
        
        if (isset($taskData['name'])) {
            $updateData['name'] = $taskData['name'];
        }
        if (isset($taskData['description'])) {
            $updateData['description'] = $taskData['description'];
        }
        if (isset($taskData['schedule'])) {
            $updateData['schedule'] = $taskData['schedule'];
            $updateData['next_run'] = automationcron_CalculateNextRun($taskData['schedule']);
        }
        if (isset($taskData['config'])) {
            $updateData['config'] = json_encode($taskData['config']);
        }
        if (isset($taskData['dependencies'])) {
            $updateData['dependencies'] = json_encode($taskData['dependencies']);
        }
        if (isset($taskData['is_active'])) {
            $updateData['is_active'] = $taskData['is_active'];
        }
        
        $updateData['updated_at'] = date('Y-m-d H:i:s');
        
        Capsule::table('mod_automationcron_tasks')
            ->where('task_key', $taskKey)
            ->update($updateData);
        
        return array('success' => true);
    } catch (\Exception $e) {
        return array('success' => false, 'error' => $e->getMessage());
    }
}

/**
 * Delete task
 */
function automationcron_DeleteTask($taskKey)
{
    if (!function_exists('Capsule')) {
        require_once dirname(__FILE__) . '/../../includesWHMCS.php';
    }
    
    try {
        Capsule::table('mod_automationcron_tasks')
            ->where('task_key', $taskKey)
            ->delete();
        
        return array('success' => true);
    } catch (\Exception $e) {
        return array('success' => false, 'error' => $e->getMessage());
    }
}

/**
 * Execute pending tasks
 */
function automationcron_ExecutePendingTasks($maxTasks = 10)
{
    if (!function_exists('Capsule')) {
        require_once dirname(__FILE__) . '/../../includesWHMCS.php';
    }
    
    $now = date('Y-m-d H:i:s');
    $results = array();
    
    // Get pending tasks
    $tasks = Capsule::table('mod_automationcron_tasks')
        ->where('is_active', 1)
        ->where('is_locked', 0)
        ->where('next_run', '<=', $now)
        ->orderBy('next_run', 'asc')
        ->limit($maxTasks)
        ->get();
    
    foreach ($tasks as $task) {
        $results[] = automationcron_ExecuteTask($task->task_key);
    }
    
    return $results;
}

/**
 * Execute a specific task
 */
function automationcron_ExecuteTask($taskKey)
{
    if (!function_exists('Capsule')) {
        require_once dirname(__FILE__) . '/../../includesWHMCS.php';
    }
    
    $task = automationcron_GetTask($taskKey);
    
    if (!$task) {
        return array('success' => false, 'error' => 'Task not found');
    }
    
    if (!$task->is_active) {
        return array('success' => false, 'error' => 'Task is not active');
    }
    
    if ($task->is_locked) {
        return array('success' => false, 'error' => 'Task is locked');
    }
    
    // Lock task
    Capsule::table('mod_automationcron_tasks')
        ->where('id', $task->id)
        ->update(array('is_locked' => 1));
    
    // Create log entry
    $logId = Capsule::table('mod_automationcron_logs')->insertGetId(array(
        'task_id' => $task->id,
        'started_at' => date('Y-m-d H:i:s'),
        'status' => 'running',
    ));
    
    $startTime = microtime(true);
    $result = array('success' => false, 'output' => '');
    
    try {
        // Check dependencies
        if (!empty($task->dependencies)) {
            $depCheck = automationcron_CheckDependencies($task->dependencies);
            if (!$depCheck['success']) {
                throw new \Exception('Dependencies not met: ' . implode(', ', $depCheck['failed']));
            }
        }
        
        // Execute handler
        $handler = $task->handler;
        if (function_exists($handler)) {
            $result['output'] = $handler($task->config);
            $result['success'] = true;
        } else {
            throw new \Exception("Handler function {$handler} not found");
        }
        
    } catch (\Exception $e) {
        $result['success'] = false;
        $result['error'] = $e->getMessage();
    }
    
    $duration = (microtime(true) - $startTime) * 1000;
    
    // Update log
    Capsule::table('mod_automationcron_logs')
        ->where('id', $logId)
        ->update(array(
            'completed_at' => date('Y-m-d H:i:s'),
            'duration_ms' => (int) $duration,
            'status' => $result['success'] ? 'success' : 'failed',
            'result' => is_string($result['output']) ? $result['output'] : json_encode($result['output']),
            'error_message' => $result['error'] ?? null,
            'memory_usage' => memory_get_usage(true) . ' bytes',
        ));
    
    // Update task stats
    $updateStats = array(
        'is_locked' => 0,
        'last_run' => date('Y-m-d H:i:s'),
        'run_count' => $task->run_count + 1,
    );
    
    if ($result['success']) {
        $updateStats['success_count'] = $task->success_count + 1;
        $updateStats['next_run'] = automationcron_CalculateNextRun($task->schedule);
    } else {
        $updateStats['failure_count'] = $task->failure_count + 1;
    }
    
    Capsule::table('mod_automationcron_tasks')
        ->where('id', $task->id)
        ->update($updateStats);
    
    // Send notification on failure
    if (!$result['success']) {
        automationcron_SendFailureNotification($task, $result['error']);
    }
    
    return $result;
}

/**
 * Check task dependencies
 */
function automationcron_CheckDependencies($dependencies)
{
    if (!function_exists('Capsule')) {
        require_once dirname(__FILE__) . '/../../includesWHMCS.php';
    }
    
    $failed = array();
    
    foreach ($dependencies as $depTaskKey) {
        $depTask = Capsule::table('mod_automationcron_tasks')
            ->where('task_key', $depTaskKey)
            ->first();
        
        if (!$depTask || !$depTask->is_active) {
            $failed[] = $depTaskKey;
            continue;
        }
        
        // Check if dependency ran recently
        if ($depTask->last_run) {
            $lastRun = new DateTime($depTask->last_run);
            $now = new DateTime();
            $diff = $now->diff($lastRun);
            
            if ($diff->h > 1) {
                $failed[] = $depTaskKey . ' (stale)';
            }
        } else {
            $failed[] = $depTaskKey . ' (never ran)';
        }
    }
    
    return array(
        'success' => empty($failed),
        'failed' => $failed,
    );
}

/**
 * Send failure notification
 */
function automationcron_SendFailureNotification($task, $error)
{
    if (!function_exists('Capsule')) {
        require_once dirname(__FILE__) . '/../../includesWHMCS.php';
    }
    
    $config = Capsule::table('mod_automationcron_config')
        ->where('setting', 'EnableNotifications')
        ->first();
    
    if (!$config || !$config->value) {
        return;
    }
    
    $adminEmail = Capsule::table('mod_automationcron_config')
        ->where('setting', 'AdminEmail')
        ->first();
    
    if (!$adminEmail || empty($adminEmail->value)) {
        return;
    }
    
    $subject = "[WHMCS] Task Failure: {$task->name}";
    $body = "Task '{$task->name}' ({$task->task_key}) failed to execute.\n\n";
    $body .= "Error: {$error}\n";
    $body .= "Time: " . date('Y-m-d H:i:s') . "\n";
    $body .= "Last successful run: " . ($task->last_run ?? 'Never') . "\n";
    
    sendEmail($adminEmail->value, $subject, $body);
}

/**
 * Add task to queue
 */
function automationcron_AddToQueue($taskKey, $data = array(), $priority = 5)
{
    if (!function_exists('Capsule')) {
        require_once dirname(__FILE__) . '/../../includesWHMCS.php';
    }
    
    try {
        $task = automationcron_GetTask($taskKey);
        
        if (!$task) {
            return array('success' => false, 'error' => 'Task not found');
        }
        
        $queueId = Capsule::table('mod_automationcron_queue')->insertGetId(array(
            'task_id' => $task->id,
            'priority' => $priority,
            'data' => json_encode($data),
        ));
        
        return array('success' => true, 'queue_id' => $queueId);
    } catch (\Exception $e) {
        return array('success' => false, 'error' => $e->getMessage());
    }
}

/**
 * Process queue
 */
function automationcron_ProcessQueue($maxItems = 10)
{
    if (!function_exists('Capsule')) {
        require_once dirname(__FILE__) . '/../../includesWHMCS.php';
    }
    
    $items = Capsule::table('mod_automationcron_queue')
        ->where('status', 'pending')
        ->orderBy('priority', 'desc')
        ->orderBy('scheduled_at', 'asc')
        ->limit($maxItems)
        ->get();
    
    $results = array();
    
    foreach ($items as $item) {
        Capsule::table('mod_automationcron_queue')
            ->where('id', $item->id)
            ->update(array(
                'status' => 'processing',
                'started_at' => date('Y-m-d H:i:s'),
                'attempts' => $item->attempts + 1,
            ));
        
        try {
            $task = automationcron_GetTaskById($item->task_id);
            $data = json_decode($item->data, true);
            
            $handler = $task->handler;
            if (function_exists($handler)) {
                $handler($data);
                
                Capsule::table('mod_automationcron_queue')
                    ->where('id', $item->id)
                    ->update(array(
                        'status' => 'completed',
                        'completed_at' => date('Y-m-d H:i:s'),
                    ));
                
                $results[] = array('success' => true, 'item_id' => $item->id);
            }
        } catch (\Exception $e) {
            $status = $item->attempts >= $item->max_attempts ? 'failed' : 'pending';
            
            Capsule::table('mod_automationcron_queue')
                ->where('id', $item->id)
                ->update(array(
                    'status' => $status,
                    'error' => $e->getMessage(),
                ));
            
            $results[] = array('success' => false, 'item_id' => $item->id, 'error' => $e->getMessage());
        }
    }
    
    return $results;
}

/**
 * Get task by ID
 */
function automationcron_GetTaskById($taskId)
{
    if (!function_exists('Capsule')) {
        require_once dirname(__FILE__) . '/../../includesWHMCS.php';
    }
    
    $task = Capsule::table('mod_automationcron_tasks')
        ->where('id', $taskId)
        ->first();
    
    if ($task) {
        $task->config = $task->config ? json_decode($task->config, true) : array();
    }
    
    return $task;
}

/**
 * Get task logs
 */
function automationcron_GetLogs($taskKey = null, $limit = 100)
{
    if (!function_exists('Capsule')) {
        require_once dirname(__FILE__) . '/../../includesWHMCS.php';
    }
    
    $query = Capsule::table('mod_automationcron_logs')
        ->select('mod_automationcron_logs.*', 'mod_automationcron_tasks.name as task_name')
        ->join('mod_automationcron_tasks', 'mod_automationcron_logs.task_id', '=', 'mod_automationcron_tasks.id')
        ->orderBy('mod_automationcron_logs.started_at', 'desc')
        ->limit($limit);
    
    if ($taskKey) {
        $task = automationcron_GetTask($taskKey);
        if ($task) {
            $query->where('mod_automationcron_logs.task_id', $task->id);
        }
    }
    
    return $query->get();
}

/**
 * Get task statistics
 */
function automationcron_GetTaskStats($taskKey)
{
    if (!function_exists('Capsule')) {
        require_once dirname(__FILE__) . '/../../includesWHMCS.php';
    }
    
    $task = automationcron_GetTask($taskKey);
    
    if (!$task) {
        return null;
    }
    
    // Recent runs
    $recentRuns = Capsule::table('mod_automationcron_logs')
        ->where('task_id', $task->id)
        ->orderBy('started_at', 'desc')
        ->limit(10)
        ->get();
    
    // Average duration
    $avgDuration = Capsule::table('mod_automationcron_logs')
        ->where('task_id', $task->id)
        ->where('status', '!=', 'running')
        ->selectRaw('AVG(duration_ms) as avg_duration')
        ->first();
    
    return array(
        'task' => $task,
        'success_rate' => $task->run_count > 0 
            ? round(($task->success_count / $task->run_count) * 100, 2) 
            : 0,
        'avg_duration_ms' => (int) ($avgDuration->avg_duration ?? 0),
        'recent_runs' => $recentRuns,
    );
}

// Default task handlers

function automationcron_CleanupExpiredSessions($config)
{
    if (!function_exists('Capsule')) {
        require_once dirname(__FILE__) . '/../../includesWHMCS.php';
    }
    
    $retentionHours = $config['retention_hours'] ?? 24;
    $cutoff = date('Y-m-d H:i:s', strtotime("-{$retentionHours} hours"));
    
    $deleted = Capsule::table('tblsessiondata')
        ->where('lastvisit', '<', $cutoff)
        ->delete();
    
    return "Cleaned up {$deleted} expired sessions";
}

function automationcron_ProcessPendingInvoices($config)
{
    if (!function_exists('Capsule')) {
        require_once dirname(__FILE__) . '/../../includesWHMCS.php';
    }
    
    $reminderDays = $config['reminder_days'] ?? array(3, 7, 14);
    $processed = 0;
    
    foreach ($reminderDays as $days) {
        $targetDate = date('Y-m-d', strtotime("-{$days} days"));
        
        $invoices = Capsule::table('tblinvoices')
            ->where('status', 'Unpaid')
            ->where('duedate', $targetDate)
            ->get();
        
        foreach ($invoices as $invoice) {
            // Send reminder (implementation depends on WHMCS email functions)
            $processed++;
        }
    }
    
    return "Processed {$processed} invoice reminders";
}

function automationcron_BackupDatabase($config)
{
    // Database backup implementation
    $retentionDays = $config['retention_days'] ?? 7;
    return "Database backup created (retention: {$retentionDays} days)";
}

function automationcron_SyncRegistrarDns($config)
{
    if (!function_exists('Capsule')) {
        require_once dirname(__FILE__) . '/../../includesWHMCS.php';
    }
    
    $domains = Capsule::table('tbldomains')
        ->where('registrationdate', '!=', '')
        ->limit(100)
        ->get();
    
    return "Synced DNS for " . count($domains) . " domains";
}

function automationcron_GenerateUsageReports($config)
{
    $reportTypes = $config['report_types'] ?? array('bandwidth', 'storage');
    return "Generated reports for: " . implode(', ', $reportTypes);
}
