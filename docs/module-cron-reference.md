# WHMCS Module Cron Reference

**Version:** 8.0 | **Updated:** 2026-05-28

## Overview

This reference documents cron job implementations for WHMCS modules, including scheduled task registration, execution patterns, and best practices for background processing.

---

## Cron Job Registration

### Automatic Registration

```php
/**
 * Declare cron jobs in module metadata
 */
function your_module_MetaData()
{
    return [
        'DisplayName' => 'Your Module',
        'CronHandles' => [
            'sync' => 'YourModule\Cron\SyncTask',
            'cleanup' => 'YourModule\Cron\CleanupTask',
            'reports' => 'YourModule\Cron\ReportsTask',
        ],
    ];
}
```

### Hook-Based Registration

```php
/**
 * Register cron via hook
 */
function your_module_registerCron()
{
    add_hook('AfterCronJob', 100, function($vars) {
        your_module_processCronTasks();
    });
}

/**
 * Called during module activation
 */
function your_module_activate()
{
    // Register cron jobs
    your_module_registerCron();
    
    return ['status' => 'success'];
}

/**
 * Remove cron jobs on deactivation
 */
function your_module_deactivate()
{
    // Cron hooks auto-remove when module disabled
}
```

## Cron Implementation

### Basic Cron Task

```php
<?php
/**
 * Module Cron Handler
 * 
 * Place in modules/addons/your_addon/cron.php
 */

if (!defined("WHMCS")) {
    die("This file cannot be accessed directly");
}

/**
 * Main cron entry point
 */
function your_module_cron()
{
    $startTime = microtime(true);
    $processed = 0;
    $failed = 0;
    
    // Log cron start
    logActivity("Your Module: Starting cron job");
    
    try {
        // Run sync tasks
        $result = your_module_runSync();
        $processed += $result['processed'];
        $failed += $result['failed'];
        
        // Run cleanup
        your_module_cleanup();
        
    } catch (\Exception $e) {
        logActivity("Your Module: Cron error - " . $e->getMessage());
    }
    
    $duration = round(microtime(true) - $startTime, 2);
    
    logActivity("Your Module: Cron completed in {$duration}s - Processed: {$processed}, Failed: {$failed}");
}

/**
 * Execute cron function
 */
your_module_cron();
```

### Sync Task

```php
/**
 * Sync data with external service
 */
function your_module_runSync($limit = 100)
{
    // Get pending items
    $pending = WHMCS\Database\Capsule::table('mod_your_queue')
        ->where('status', 'pending')
        ->where('task', 'sync')
        ->orderBy('created_at')
        ->limit($limit)
        ->get();
    
    $processed = 0;
    $failed = 0;
    
    foreach ($pending as $item) {
        try {
            $data = json_decode($item->data, true);
            
            // Perform sync
            your_module_syncItem($item->entity_type, $item->entity_id, $data);
            
            // Mark complete
            WHMCS\Database\Capsule::table('mod_your_queue')
                ->where('id', $item->id)
                ->update([
                    'status'     => 'completed',
                    'updated_at' => date('Y-m-d H:i:s'),
                ]);
            
            $processed++;
            
        } catch (\Exception $e) {
            // Mark as failed
            WHMCS\Database\Capsule::table('mod_your_queue')
                ->where('id', $item->id)
                ->update([
                    'status'     => 'failed',
                    'error'      => $e->getMessage(),
                    'attempts'   => $item->attempts + 1,
                    'updated_at' => date('Y-m-d H:i:s'),
                ]);
            
            // Retry later if under limit
            if ($item->attempts < 3) {
                WHMCS\Database\Capsule::table('mod_your_queue')
                    ->where('id', $item->id)
                    ->update(['status' => 'pending']);
            }
            
            $failed++;
        }
    }
    
    return [
        'processed' => $processed,
        'failed'    => $failed,
    ];
}
```

### Cleanup Task

```php
/**
 * Cleanup old data
 */
function your_module_cleanup()
{
    $retentionDays = 30;
    
    // Delete old queue items
    $cutoff = date('Y-m-d H:i:s', strtotime("-{$retentionDays} days"));
    
    $deleted = WHMCS\Database\Capsule::table('mod_your_queue')
        ->where('status', 'completed')
        ->where('updated_at', '<', $cutoff)
        ->delete();
    
    if ($deleted > 0) {
        logActivity("Your Module: Cleaned up {$deleted} old queue items");
    }
    
    // Clean up temp files
    $tempDir = ROOTDIR . '/modules/addons/your_addon/temp';
    if (is_dir($tempDir)) {
        $files = glob("{$tempDir}/*");
        $now = time();
        
        foreach ($files as $file) {
            if (is_file($file) && ($now - filemtime($file)) > 86400) {
                unlink($file);
            }
        }
    }
    
    return $deleted;
}
```

### Notification Task

```php
/**
 * Send scheduled notifications
 */
function your_module_sendNotifications($limit = 50)
{
    $pending = WHMCS\Database\Capsule::table('mod_your_notifications')
        ->where('status', 'pending')
        ->where('scheduled_at', '<=', date('Y-m-d H:i:s'))
        ->orderBy('scheduled_at')
        ->limit($limit)
        ->get();
    
    $sent = 0;
    $failed = 0;
    
    foreach ($pending as $notification) {
        try {
            // Send notification
            your_module_sendNotification($notification);
            
            WHMCS\Database\Capsule::table('mod_your_notifications')
                ->where('id', $notification->id)
                ->update([
                    'status'       => 'sent',
                    'sent_at'      => date('Y-m-d H:i:s'),
                ]);
            
            $sent++;
            
        } catch (\Exception $e) {
            WHMCS\Database\Capsule::table('mod_your_notifications')
                ->where('id', $notification->id)
                ->update([
                    'status'      => 'failed',
                    'error'       => $e->getMessage(),
                ]);
            
            $failed++;
        }
    }
    
    return [
        'sent'   => $sent,
        'failed' => $failed,
    ];
}
```

## Queue Management

### Adding to Queue

```php
/**
 * Add task to queue
 */
function your_module_queueTask($task, $entityType, $entityId, $data = [], $priority = 5)
{
    return WHMCS\Database\Capsule::table('mod_your_queue')->insert([
        'task'        => $task,
        'task_data'   => json_encode($data),
        'entity_type' => $entityType,
        'entity_id'   => $entityId,
        'priority'   => $priority,
        'status'      => 'pending',
        'attempts'    => 0,
        'created_at'  => date('Y-m-d H:i:s'),
    ]);
}

/**
 * Queue from hook
 */
function onInvoicePaid($vars)
{
    your_module_queueTask('sync', 'invoice', $vars['invoiceid'], [
        'amount'    => $vars['total'],
        'client_id' => $vars['userid'],
    ]);
    
    return true;
}
```

### Priority Queue

```php
/**
 * Get prioritized tasks
 */
function your_module_getPrioritizedTasks($limit = 100)
{
    return WHMCS\Database\Capsule::table('mod_your_queue')
        ->where('status', 'pending')
        ->where('attempts', '<', 5)
        ->orderBy('priority', 'desc')
        ->orderBy('created_at', 'asc')
        ->limit($limit)
        ->get();
}

/**
 * Run with priority
 */
function your_module_runPriorityQueue($limit = 100)
{
    $tasks = your_module_getPrioritizedTasks($limit);
    
    foreach ($tasks as $task) {
        // Process based on priority
        $retryDelay = $task->priority > 7 ? 0 : pow(2, $task->attempts);
        
        if ($retryDelay > 0) {
            $retryAt = date('Y-m-d H:i:s', time() + $retryDelay);
            if (strtotime($task->next_attempt_at) > time()) {
                continue;
            }
        }
        
        your_module_processTask($task->id);
    }
}
```

## Database Queue Table

```php
/**
 * Create queue table
 */
function createQueueTable()
{
    $sql = "CREATE TABLE IF NOT EXISTS `mod_your_queue` (
        `id` BIGINT(20) NOT NULL AUTO_INCREMENT PRIMARY KEY,
        `task` VARCHAR(50) NOT NULL,
        `task_data` TEXT NULL,
        `entity_type` VARCHAR(50) NULL,
        `entity_id` INT(10) NULL,
        `priority` TINYINT(2) NOT NULL DEFAULT 5,
        `status` ENUM('pending', 'processing', 'completed', 'failed') NOT NULL DEFAULT 'pending',
        `attempts` TINYINT(2) NOT NULL DEFAULT 0,
        `max_attempts` TINYINT(2) NOT NULL DEFAULT 3,
        `error` TEXT NULL,
        `next_attempt_at` DATETIME NULL,
        `created_at` DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
        `updated_at` DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
        `completed_at` DATETIME NULL,
        INDEX `idx_status_priority` (`status`, `priority`, `created_at`),
        INDEX `idx_entity` (`entity_type`, `entity_id`),
        INDEX `idx_next_attempt` (`next_attempt_at`)
    ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4";
    
    full_query($sql);
}

/**
 * Create notifications table
 */
function createNotificationsTable()
{
    $sql = "CREATE TABLE IF NOT EXISTS `mod_your_notifications` (
        `id` BIGINT(20) NOT NULL AUTO_INCREMENT PRIMARY KEY,
        `client_id` INT(10) NULL,
        `type` VARCHAR(50) NOT NULL,
        `data` TEXT NULL,
        `scheduled_at` DATETIME NOT NULL,
        `status` ENUM('pending', 'sent', 'failed') NOT NULL DEFAULT 'pending',
        `sent_at` DATETIME NULL,
        `error` TEXT NULL,
        `created_at` DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
        INDEX `idx_status_scheduled` (`status`, `scheduled_at`),
        INDEX `idx_client` (`client_id`)
    ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4";
    
    full_query($sql);
}
```

## Cron Scheduling

### Scheduled Task Options

| Schedule | Timing | Use Case |
|----------|--------|----------|
| Every 5 minutes | */5 * * * * | Real-time sync, notifications |
| Every 15 minutes | */15 * * * * | Data updates, reports |
| Every hour | 0 * * * * | Cleanup, aggregations |
| Every day | 0 0 * * * | Daily reports, backups |
| Twice daily | 0 */12 * * * | Batch operations |

### WHMCS Cron Configuration

```
# WHMCS cron command
/usr/bin/php -q /path/to/whmcs/crons/cron.php

# Add to crontab
*/5 * * * * /usr/bin/php -q /path/to/whmcs/crons/cron.php
```

### Module-Specific Cron

```
# Module-specific cron (if supported)
*/5 * * * * /usr/bin/php -q /path/to/whmcs/modules/addons/your_addon/cron.php
```

## Cron Monitoring

### Status Reporting

```php
/**
 * Report cron status
 */
function your_module_reportStatus()
{
    $stats = [
        'pending_tasks' => WHMCS\Database\Capsule::table('mod_your_queue')
            ->where('status', 'pending')
            ->count(),
        'failed_tasks' => WHMCS\Database\Capsule::table('mod_your_queue')
            ->where('status', 'failed')
            ->count(),
        'pending_notifications' => WHMCS\Database\Capsule::table('mod_your_notifications')
            ->where('status', 'pending')
            ->count(),
    ];
    
    // Log summary
    logActivity(sprintf(
        "Your Module Status: Pending: %d, Failed: %d, Notifications: %d",
        $stats['pending_tasks'],
        $stats['failed_tasks'],
        $stats['pending_notifications']
    ));
    
    return $stats;
}
```

### Alert on Failures

```php
/**
 * Send alert if failures detected
 */
function your_module_checkForAlerts()
{
    $failedCount = WHMCS\Database\Capsule::table('mod_your_queue')
        ->where('status', 'failed')
        ->where('attempts', '>=', 3)
        ->count();
    
    if ($failedCount > 10) {
        // Send admin alert
        send_admin_notification(
            'system',
            'Your Module: High Failure Rate',
            "There are {$failedCount} failed tasks requiring attention."
        );
    }
}
```

---

## Related Skills and a Workflows

- `cron-job-reference` - WHMCS cron setup
- `cron-events-reference` - Cron event hooks
- `module-performance-best-practices` - Background processing
