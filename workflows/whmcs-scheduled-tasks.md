# WHMCS Scheduled Tasks Workflow

## Overview
This workflow configures and manages scheduled tasks in WHMCS for automated background operations.

## Prerequisites
- WHMCS installation with cron configured
- Admin access to Configuration > System Settings > Automation
- Understanding of cron syntax

## Step-by-Step Process

### Step 1: Configure Cron Jobs
1. Access WHMCS admin area
2. Navigate to **Configuration > System Settings > Automation**
3. Note the cron command:
   ```bash
   php -q /path/to/whmcs/admin/cron.php
   ```

### Step 2: Set Up System Cron
1. SSH to server as root
2. Edit crontab:
   ```bash
   crontab -e
   ```

3. Add cron entry:
   ```bash
   # Run WHMCS cron every 5 minutes
   */5 * * * * php -q /var/www/html/whmcs/admin/cron.php > /dev/null 2>&1
   ```

4. Alternative: Use system scheduler (systemd timer)
   ```bash
   # Create timer unit
   sudo nano /etc/systemd/system/whmcs-cron.timer

   [Timer]
   OnBootSec=5min
   OnUnitActiveSec=5min
   Unit=whmcs-cron.service

   [Install]
   WantedBy=timers.target
   ```

### Step 3: Configure WHMCS Built-in Tasks
Navigate to **Configuration > System Settings > Automation** and configure:

#### Invoice Generation
- **Invoice Days Before Due Date**: Set advance notice (e.g., 7 days)
- **Invoice Day of Month**: Choose billing cycle (1-28)

#### Payment Reminders
- **First Reminder**: Days after due date
- **Second Reminder**: Days after first
- **Third Reminder**: Days after second

#### Suspension & Termination
- **Suspend Services Days**: Days after invoice overdue
- **Terminate Services Days**: Days after suspension

### Step 4: Create Custom Scheduled Tasks

#### Task Registration
```php
// Create file: /includes/hooks/custom_scheduled_tasks.php

use WHMCS\Scheduling\ScheduledTask\AdminTask;

add_hook('DailyCronJob', 1, function($vars) {
    // Your task logic here
});
```

#### Advanced Task with Options
```php
<?php

use WHMCS\Scheduling\ScheduledTask\Task;

add_hook('DailyCronJob', 1, function(Task $task, $args) {
    // Task configuration
    $task->options([
        'description' => 'Custom Data Cleanup Task',
        'help_text' => 'Cleans up stale data older than retention period',
        'run_every' => 1440, // minutes (24 hours)
        'disabled' => false,
    ]);

    // Task logic
    $cutoffDate = date('Y-m-d', strtotime('-90 days'));
    $deleted = Capsule::table('tblactivity_log')
        ->where('date', '<', $cutoffDate)
        ->delete();

    return "Cleaned up {$deleted} old activity log entries";
});
```

### Step 5: Implement Task Logic

#### License Validation Check
```php
add_hook('DailyCronJob', 1, function($task) {
    // Check all active services
    $services = Capsule::table('tblhosting')
        ->where('domainstatus', 'Active')
        ->where('servertype', 'custom_license_module')
        ->get();

    $checked = 0;
    $invalid = 0;

    foreach ($services as $service) {
        $result = LicenseAPI::validate($service->username);

        if (!$result['valid']) {
            $invalid++;
            Capsule::table('tblhosting')
                ->where('id', $service->id)
                ->update(['subscriptionid' => 'LICENSE_INVALID']);

            // Notify client
            sendLicensingNotice($service->userid, $result['reason']);
        }

        $checked++;
    }

    return "License check complete: {$checked} checked, {$invalid} invalid";
});
```

#### Usage-Based Billing Update
```php
add_hook('HourlyCronJob', 1, function($task) {
    // Get all usage-based services
    $services = Capsule::table('tblhosting')
        ->join('tblproducts', 'tblhosting.packageid', '=', 'tblproducts.id')
        ->where('tblproducts.type', 'hosting')
        ->where('tblproducts.billingcycle', 'onetime')
        ->where('tblhosting.domainstatus', 'Active')
        ->get();

    foreach ($services as $service) {
        // Fetch usage from server
        $usage = ServerAPI::getUsage(
            $service->server,
            $service->username
        );

        // Update usage record
        Capsule::table('mod_usage_records')->updateOrInsert(
            ['service_id' => $service->id],
            [
                'disk_used' => $usage['disk'],
                'bw_used' => $usage['bandwidth'],
                'updated_at' => date('Y-m-d H:i:s')
            ]
        );

        // Check thresholds
        checkUsageThresholds($service->id, $usage);
    }

    return "Updated usage for " . count($services) . " services";
});
```

#### Backups Automation
```php
add_hook('WeeklyCronJob', 1, function($task) {
    // Get all active services requiring backups
    $services = Capsule::table('tblhosting')
        ->join('tblproducts', 'tblhosting.packageid', '=', 'tblproducts.id')
        ->where('tblproducts.configoption9', 'on') // Backup enabled
        ->where('tblhosting.domainstatus', 'Active')
        ->get();

    $backedUp = 0;
    $failed = 0;

    foreach ($services as $service) {
        try {
            // Trigger backup
            $result = ServerAPI::createBackup(
                $service->server,
                $service->username,
                ['type' => 'full']
            );

            // Log backup
            Capsule::table('mod_backup_log')->insert([
                'service_id' => $service->id,
                'backup_id' => $result['backup_id'],
                'type' => 'full',
                'size' => $result['size'],
                'created_at' => date('Y-m-d H:i:s')
            ]);

            // Clean old backups (keep last 3)
            cleanupOldBackups($service->id, 3);

            $backedUp++;
        } catch (Exception $e) {
            logActivity("Backup failed for service {$service->id}: " . $e->getMessage());
            $failed++;
        }
    }

    return "Backup complete: {$backedUp} successful, {$failed} failed";
});
```

### Step 6: Monitor Task Execution

#### View Task Status
1. Navigate to **Utilities > Logs > Activity Log**
2. Filter by "Cron" for scheduled task entries

#### Custom Task Logging
```php
add_hook('DailyCronJob', 1, function($task) {
    $startTime = microtime(true);

    // Task logic
    $result = performTaskLogic();

    $executionTime = round(microtime(true) - $startTime, 2);

    // Log to custom table
    Capsule::table('mod_task_log')->insert([
        'task_name' => get_class($task),
        'status' => 'success',
        'execution_time' => $executionTime,
        'result' => $result,
        'executed_at' => date('Y-m-d H:i:s')
    ]);

    return $result;
});
```

### Step 7: Task Failure Handling
```php
add_hook('DailyCronJob', 1, function($task) {
    try {
        return performTaskLogic();
    } catch (Exception $e) {
        // Log failure
        Capsule::table('mod_task_failures')->insert([
            'task_name' => get_class($task),
            'error_message' => $e->getMessage(),
            'failed_at' => date('Y-m-d H:i:s'),
            'retry_count' => 0
        ]);

        // Send alert
        sendAdminEmail(
            "Scheduled Task Failed: " . get_class($task),
            "Task failed with error: " . $e->getMessage()
        );

        return false;
    }
});
```

## Common Scheduled Tasks

### Hourly Tasks
- Usage collection
- Service health checks
- Cache cleanup
- Session cleanup

### Daily Tasks
- Invoice generation
- Payment reminders
- License validation
- Backup execution
- Report generation

### Weekly Tasks
- Full system backups
- Log rotation
- Usage reporting
- Compliance audits

### Monthly Tasks
- Tax calculations
- Financial reconciliation
- Archive old records
- Security scanning

## Cron Schedule Reference
```
┌───────────── minute (0-59)
│ ┌───────────── hour (0-23)
│ │ ┌───────────── day of month (1-31)
│ │ │ ┌───────────── month (1-12)
│ │ │ │ ┌───────────── day of week (0-6, Sunday=0)
│ │ │ │ │
* * * * * command
```

Common schedules:
- `*/5 * * * *` - Every 5 minutes
- `0 * * * *` - Every hour
- `0 0 * * *` - Daily at midnight
- `0 2 * * *` - Daily at 2 AM
- `0 0 * * 0` - Weekly on Sunday
- `0 0 1 * *` - Monthly on 1st

## Troubleshooting

| Issue | Solution |
|-------|----------|
| Cron not running | Check cron is installed and active |
| Tasks miss schedule | Use systemd timer for reliability |
| Long execution time | Optimize queries, use indexes |
| Duplicate runs | Implement task locking |

## Related Workflows
- [WHMCS Cron Automation](./whmcs-cron-automation.md)
- [WHMCS Queue Processing](./whmcs-queue-processing.md)
- [WHMCS Backup Automation](./whmcs-backup-automation.md)