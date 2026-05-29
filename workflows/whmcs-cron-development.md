# WHMCS Cron Development Workflow

## Purpose
Create custom cron tasks for automated operations

## Prerequisites
- WHMCS installed
- SSH access
- PHP development knowledge

## Step 1: Create Custom Cron Directory

```bash
mkdir -p /var/www/whmcs/crons/custom
```

## Step 2: Create Custom Cron Script

```php
<?php
// /var/www/whmcs/crons/custom/custom_task.php

// Initialise WHMCS application
define('ADMINAREA', true);
require_once '/var/www/whmcs/init.php';

// Check if run from CLI
if (php_sapi_name() !== 'cli') {
    die('This script must be run from command line');
}

logActivity('Starting custom cron task');

try {
    $result = executeCustomTask();
    
    if ($result) {
        logActivity('Custom cron task completed successfully');
    } else {
        logActivity('Custom cron task completed with warnings', 'warning');
    }
} catch (Exception $e) {
    logActivity('Custom cron task failed: ' . $e->getMessage(), 'error');
    sendAlertEmail($e->getMessage());
}

function executeCustomTask()
{
    $results = [];
    
    // Task 1: Check expiring services
    $results['expiring_services'] = checkExpiringServices();
    
    // Task 2: Send usage reports
    $results['usage_reports'] = sendUsageReports();
    
    // Task 3: Clean up old records
    $results['cleanup'] = cleanupOldRecords();
    
    return $results;
}

function checkExpiringServices()
{
    $expiringServices = \WHMCS\Database\Capsule::table('tblhosting')
        ->where('nextduedate', '<=', Carbon::now()->addDays(7)->toDateString())
        ->where('nextduedate', '>=', Carbon::now()->toDateString())
        ->where('domainstatus', 'Active')
        ->get();
    
    foreach ($expiringServices as $service) {
        // Send notification
        sendServiceExpiryNotification($service);
    }
    
    return count($expiringServices);
}

function sendUsageReports()
{
    $clients = \WHMCS\Database\Capsule::table('tblclients')
        ->where('status', 'Active')
        ->limit(100)
        ->get();
    
    foreach ($clients as $client) {
        generateUsageReport($client);
    }
    
    return count($clients);
}

function cleanupOldRecords()
{
    $cutoffDate = Carbon::now()->subDays(90);
    
    $deleted = \WHMCS\Database\Capsule::table('tblactivitylog')
        ->where('date', '<', $cutoffDate)
        ->delete();
    
    return $deleted;
}

function sendAlertEmail($message)
{
    $mail = new WHMCS\Mail\Mail();
    $mail->send(
        ['admin@yourdomain.com'],
        'Cron Error Alert',
        ['error_message' => $message]
    );
}
```

## Step 3: Add to System Crontab

```bash
crontab -e
```

Add:
```cron
# Custom WHMCS cron tasks
# Run every 15 minutes
*/15 * * * * /usr/bin/php /var/www/whmcs/crons/custom/custom_task.php

# Run every hour
0 * * * * /usr/bin/php /var/www/whmcs/crons/custom/hourly_task.php

# Run daily at midnight
0 0 * * * /usr/bin/php /var/www/whmcs/crons/custom/daily_task.php
```

## Step 4: Create Hook-Based Cron

```php
<?php
// hooks.php - Custom cron via hook

add_hook('AfterCronJob', 1, function($vars) {
    $lastRun = getCustomCronLastRun('custom_task');
    
    if (shouldRunTask($lastRun, 15)) {
        runCustomTask();
        updateCustomCronLastRun('custom_task');
    }
});

function runCustomTask()
{
    logActivity('Running hook-based cron task');
    // Task logic here
}
```

## Step 5: Create Queue-Based Cron

```php
<?php
// queue_task.php - Queue-based processing

$queue = new WHMCS\Cron\Queue\TaskQueue();

while ($task = $queue->pop()) {
    try {
        processTask($task);
        $task->markComplete();
    } catch (Exception $e) {
        $task->markFailed($e->getMessage());
    }
}
```

## Step 6: Monitor Cron Performance

```php
<?php
// performance_monitor.php

function monitorCronPerformance()
{
    $lastCron = \WHMCS\Database\Capsule::table('tblactivitylog')
        ->where('description', 'like', '%Cron Job%')
        ->orderBy('id', 'desc')
        ->first();
    
    $executionTime = $lastCron ? time() - strtotime($lastCron->date) : null;
    
    if ($executionTime > 3600) {
        sendCronAlert('Cron not running for over an hour');
    }
}
```

## Cron Development Checklist

- [ ] Custom cron directory created
- [ ] Cron script created
- [ ] Crontab entry added
- [ ] Hook-based cron implemented
- [ ] Queue processing set up
- [ ] Performance monitoring configured
