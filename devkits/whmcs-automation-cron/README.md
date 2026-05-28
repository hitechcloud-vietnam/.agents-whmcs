# WHMCS Automation Cron Module

Advanced cron job automation module with task scheduling, dependency management, and comprehensive logging.

## Features

- Cron-based task scheduling
- Task dependencies
- Execution locking
- Comprehensive logging
- Failure notifications
- Task queue system
- Statistics and analytics
- Built-in task handlers
- Configurable retention

## Installation

1. Copy the module to your WHMCS installation:
   ```
   /path/to/whmcs/modules/servers/automationcron/
   ```

2. Activate through WHMCS Admin:
   - Go to **Setup > Addon Modules**
   - Find "Automation Cron"
   - Click **Activate**
   - Configure settings

3. Add to WHMCS cron schedule:
   ```
   * * * * * php -q /path/to/whmcs/crons/cron.php /crons/automation.php
   ```

## Default Tasks

The module registers these tasks by default:

| Task | Schedule | Description |
|------|----------|-------------|
| cleanup_expired_sessions | Hourly | Clean up expired sessions |
| process_pending_invoices | Daily 9AM | Send payment reminders |
| backup_database | Daily 2AM | Create database backups |
| sync_registrar_dns | Every 6 hours | Sync DNS with registrars |
| generate_usage_reports | Weekly Monday | Generate usage reports |

## Usage

### Managing Tasks

```php
// Create a new task
automationcron_CreateTask(array(
    'task_key' => 'my_custom_task',
    'name' => 'My Custom Task',
    'description' => 'Description here',
    'task_type' => 'custom',
    'handler' => 'my_task_handler',
    'schedule' => '0 * * * *', // Every hour
    'config' => array('key' => 'value'),
    'dependencies' => array('other_task_key'),
));

// Get all tasks
$tasks = automationcron_GetTasks(array('active_only' => true));

// Get task by key
$task = automationcron_GetTask('my_custom_task');

// Update task
automationcron_UpdateTask('my_custom_task', array(
    'schedule' => '0 9 * * *',
    'is_active' => true,
));

// Delete task
automationcron_DeleteTask('my_custom_task');
```

### Executing Tasks

```php
// Execute pending tasks (for cron)
$results = automationcron_ExecutePendingTasks(10);
foreach ($results as $result) {
    if ($result['success']) {
        echo "Task completed: " . $result['output'];
    } else {
        echo "Task failed: " . $result['error'];
    }
}

// Execute specific task
$result = automationcron_ExecuteTask('my_custom_task');
```

### Task Handlers

```php
// Create a custom task handler
function my_task_handler($config) {
    // $config contains the task configuration
    $setting = $config['setting'] ?? 'default';
    
    // Do something...
    
    return "Task completed successfully"; // or array for structured output
}

// Register the handler
automationcron_CreateTask(array(
    'task_key' => 'my_custom_task',
    'name' => 'Custom Task',
    'task_type' => 'custom',
    'handler' => 'my_task_handler',
    'schedule' => '*/15 * * * *', // Every 15 minutes
));
```

### Task Queue

```php
// Add item to queue
automationcron_AddToQueue('my_custom_task', array(
    'param1' => 'value1',
    'param2' => 'value2',
), 5); // Priority 1-10, higher = more urgent

// Process queue
$results = automationcron_ProcessQueue(10);
```

### Dependencies

```php
// Tasks can depend on other tasks completing first
automationcron_CreateTask(array(
    'task_key' => 'dependent_task',
    'name' => 'Dependent Task',
    'handler' => 'dependent_handler',
    'schedule' => '0 * * * *',
    'dependencies' => array('cleanup_expired_sessions'),
));

// Dependencies are automatically checked before execution
// If dependencies fail, the task won't run
```

### Logging and Statistics

```php
// Get task logs
$logs = automationcron_GetLogs('my_custom_task', 50);

// Get task statistics
$stats = automationcron_GetTaskStats('my_custom_task');
// Returns: task info, success_rate, avg_duration_ms, recent_runs
```

## Cron Expression Format

Uses standard cron format: `minute hour day month weekday`

Examples:
- `* * * * *` - Every minute
- `0 * * * *` - Every hour
- `0 9 * * *` - Daily at 9:00 AM
- `0 */6 * * *` - Every 6 hours
- `0 0 * * 1` - Weekly on Monday at midnight
- `*/15 * * * *` - Every 15 minutes

## Configuration Options

| Setting | Default | Description |
|---------|---------|-------------|
| EnableTasks | yes | Enable automated tasks |
| MaxExecutionTime | 300 | Max execution time (seconds) |
| EnableLogging | yes | Enable execution logging |
| LogRetentionDays | 30 | Days to keep logs |
| EnableNotifications | yes | Send failure emails |
| AdminEmail | - | Email for notifications |

## Database Tables

- `mod_automationcron_tasks` - Task definitions
- `mod_automationcron_logs` - Execution logs
- `mod_automationcron_queue` - Task queue

## API Functions

| Function | Description |
|----------|-------------|
| `automationcron_CreateTask()` | Create new task |
| `automationcron_GetTasks()` | Get all tasks |
| `automationcron_GetTask()` | Get task by key |
| `automationcron_UpdateTask()` | Update task |
| `automationcron_DeleteTask()` | Delete task |
| `automationcron_ExecuteTask()` | Execute specific task |
| `automationcron_ExecutePendingTasks()` | Execute pending tasks |
| `automationcron_AddToQueue()` | Add to task queue |
| `automationcron_ProcessQueue()` | Process queue |
| `automationcron_GetLogs()` | Get task logs |
| `automationcron_GetTaskStats()` | Get task statistics |

## Version History

- **1.0.0** - Initial release
  - Task scheduling
  - Dependency management
  - Execution logging
  - Task queue
  - Failure notifications
  - Built-in handlers
