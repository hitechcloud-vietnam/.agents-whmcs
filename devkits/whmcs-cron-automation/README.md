# WHMCS Cron Automation DevKit

An advanced cron job automation module for WHMCS that enables scheduled task execution, task scheduling, and automatic retries.

## Features

- Cron expression-based scheduling
- Multiple concurrent task execution
- Automatic retry on failure
- Task execution logging
- Built-in common tasks (cleanup, reports, sync)
- Admin interface for task management
- Custom task support

## Installation

1. Copy module files to:
   ```
   modules/addons/whmcs_cron_automation/
   ```

2. Activate the module in WHMCS Admin > Addon Modules

3. Add to server crontab:
   ```
   * * * * * php -q /path/to/whmcs/crons/cron.php --module={module}
   ```

## Default Tasks

The module comes with several pre-configured tasks:

- **Cleanup Expired Sessions** - Runs every 6 hours
- **Invoice Reminders** - Runs daily at 9 AM
- **Domain Sync** - Runs every 4 hours
- **Daily Report** - Runs at midnight
- **Cleanup Temp Files** - Runs daily at 3 AM

## Configuration

### Task Settings

Each task can be configured with:
- **Schedule**: Cron expression (e.g., `0 9 * * *` for daily at 9 AM)
- **Max Runtime**: Maximum execution time in seconds
- **Retry Count**: Number of retries on failure
- **Custom Config**: Task-specific configuration options

### Cron Expression Format

```
* * * * *
| | | | └── Day of week (0-7)
| | | └──── Month (1-12)
| | └────── Day of month (1-31)
| └──────── Hour (0-23)
└───────── Minute (0-59)
```

Examples:
- `0 9 * * *` - Daily at 9 AM
- `*/15 * * * *` - Every 15 minutes
- `0 0 * * 0` - Weekly on Sunday at midnight

## Usage

### Running Tasks Manually

```bash
# Run all due tasks
php -q /path/to/whmcs/crons/cron.php --module={module}

# Run specific task
# Access via admin panel or:
run_task($scheduleId);
```

### Programmatic Task Execution

```php
$runner = new \CronAutomation\TaskRunner();

// Run all due tasks
$count = $runner->runDueTasks();

// Run specific task
$runner->forceRun($scheduleId);

// Get running tasks
$running = $runner->getRunningTasks();
```

### Creating Custom Tasks

```php
namespace CronAutomation\Tasks;

use CronAutomation\BaseTask;
use WHMCS\Database\Capsule;

class MyCustomTask extends BaseTask {
    
    public function execute(array $config): mixed {
        // Your task logic here
        
        return [
            'success' => true,
            'processed' => 100,
        ];
    }
}
```

### Registering Custom Tasks

```php
TaskRegistry::register(
    'my_custom_task',
    'MyCustomTask',
    'Description of what this task does'
);
```

## Task API

### BaseTask Class

All tasks extend `BaseTask` which provides:

```php
// Get configuration
$maxAge = $this->config['max_age'] ?? 86400;

// Validate config
$this->validateConfig(['required_field']);

// Log messages
$this->log('info', 'Task completed successfully');
```

### Return Values

Tasks should return data that will be stored in execution logs:

```php
return [
    'items_processed' => 100,
    'items_failed' => 5,
    'duration' => '2.5s',
];
```

## Admin Interface

### Task Management

- View all scheduled tasks
- Enable/disable tasks
- Edit task schedules
- Run tasks manually
- View execution history

### Execution Logs

View detailed logs for each task execution:
- Start time
- Duration
- Output
- Errors
- Status

### Configuration

Edit task-specific configuration through the admin interface.

## Built-in Task Reference

### CleanupExpiredSessionsTask

```php
$config = [
    'max_age' => 86400, // 24 hours in seconds
];
```

### InvoiceReminderTask

```php
$config = [
    'days_before' => [3, 7, 14], // Send reminders 3, 7, and 14 days before due date
];
```

### DomainSyncTask

```php
$config = [
    // No config needed - syncs all domains
];
```

### DailyReportTask

```php
$config = [
    'recipients' => ['admin@example.com'], // Email report recipients
];
```

### CleanupTempFilesTask

```php
$config = [
    'max_age' => 172800, // 48 hours in seconds
];
```

## Hooks

The module integrates with WHMCS cron hooks:

- `DailyCronJob` - Daily task execution
- `HourlyCronJob` - Hourly task execution

## Database Schema

### schedules table
- `id` - Task ID
- `task_name` - Display name
- `task_class` - PHP class name
- `schedule` - Cron expression
- `config` - JSON configuration
- `is_active` - Enable/disable
- `last_run` - Last execution time
- `next_run` - Next scheduled run
- `status` - Current status

### executions table
- `id` - Execution ID
- `schedule_id` - Related schedule
- `status` - running/completed/failed
- `output` - Execution output
- `error` - Error message if failed
- `duration` - Execution time
- `started_at` - Start timestamp
- `completed_at` - Completion timestamp

### logs table
- `id` - Log ID
- `level` - debug/info/warning/error
- `task` - Task name
- `message` - Log message
- `created_at` - Timestamp

## File Structure

```
whmcs-cron-automation/
├── cron-automation.php     # Main module
├── lib/
│   ├── CronScheduler.php    # Task scheduling
│   ├── TaskRunner.php       # Task execution
│   └── TaskRegistry.php    # Task registration
├── tasks/
│   ├── CleanupTask.php
│   ├── ReportTask.php
│   ├── SyncTask.php
│   └── NotificationTask.php
└── templates/
    └── cron-config.tpl
```

## Requirements

- WHMCS 7.0+
- PHP 7.4+
- CLI access for cron execution

## Support

For issues and feature requests, please contact the developer.