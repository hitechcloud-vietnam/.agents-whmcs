# WHMCS Backup Scheduler Module

Automated backup scheduling with multiple storage targets, retention policies, and notifications.

## Features

- Flexible scheduling (hourly to monthly)
- Multiple backup types
- Multiple storage targets (local, S3, FTP, SFTP)
- Retention policies
- Compression and encryption
- Job monitoring and logging
- Email notifications
- Automatic cleanup

## Installation

1. Copy `backupscheduler.php` to `/path/to/whmcs/modules/addons/backupscheduler/`
2. Activate in WHMCS Admin > System Settings > Module Addons
3. Configure storage targets

## Usage

```php
// Create backup schedule
$result = backupscheduler_CreateSchedule(array(
    'schedule_name' => 'Daily Database Backup',
    'schedule_key' => 'daily_db_backup',
    'backup_type' => 'database',
    'scope' => array('database'),
    'frequency' => 'daily',
    'run_time' => '02:00:00',
    'retention_count' => 7,
    'retention_days' => 30,
    'storage_type' => 'local',
    'storage_config' => array('path' => '/backups/whmcs'),
    'compression_level' => 5,
    'notify_on_success' => true,
    'notify_on_failure' => true,
    'recipients' => array('admin@example.com')
));

// Create full weekly backup
backupscheduler_CreateSchedule(array(
    'schedule_name' => 'Weekly Full Backup',
    'schedule_key' => 'weekly_full_backup',
    'backup_type' => 'full',
    'scope' => array('database', 'config', 'attachments', 'templates', 'modules'),
    'frequency' => 'weekly',
    'run_time' => '03:00:00',
    'day_of_week' => 0, // Sunday
    'retention_count' => 4,
    'storage_type' => 's3',
    'storage_config' => array(
        'bucket' => 'my-backups',
        'region' => 'us-east-1',
        'access_key' => 'xxx',
        'secret_key' => 'xxx'
    )
));

// Create monthly backup
backupscheduler_CreateSchedule(array(
    'schedule_name' => 'Monthly Archive',
    'schedule_key' => 'monthly_archive',
    'backup_type' => 'full',
    'scope' => array('database', 'config', 'attachments', 'templates', 'modules'),
    'frequency' => 'monthly',
    'run_time' => '01:00:00',
    'day_of_month' => 1,
    'retention_count' => 12,
    'retention_days' => 365,
    'storage_type' => 'local',
    'encrypt_backup' => true
));

// Get all schedules
$schedules = backupscheduler_GetSchedules();
$activeSchedules = backupscheduler_GetSchedules(true);

// Get specific schedule
$schedule = backupscheduler_GetSchedule($scheduleId);

// Update schedule
backupscheduler_UpdateSchedule($scheduleId, array(
    'frequency' => 'daily',
    'run_time' => '04:00:00',
    'retention_count' => 14
));

// Enable/disable schedule
backupscheduler_UpdateSchedule($scheduleId, array('is_active' => false));

// Delete schedule
backupscheduler_DeleteSchedule($scheduleId);

// Run backup manually
$result = backupscheduler_RunBackup($scheduleId);
// Returns: success, job_id, file_path, size, checksum

// Get due backups (for cron)
$dueSchedules = backupscheduler_GetDueSchedules();
foreach ($dueSchedules as $schedule) {
    backupscheduler_RunBackup($schedule->id);
}

// Get jobs
$jobs = backupscheduler_GetJobs();
$jobs = backupscheduler_GetJobs($scheduleId);

// Get specific job
$job = backupscheduler_GetJob($jobId);

// Get job logs
$logs = backupscheduler_GetLogs($jobId);

// Add storage target
backupscheduler_AddStorage(array(
    'storage_name' => 'AWS S3',
    'storage_key' => 'aws_primary',
    'storage_type' => 's3',
    'config' => array(
        'bucket' => 'my-backups',
        'region' => 'us-east-1',
        'access_key' => 'xxx',
        'secret_key' => 'xxx'
    ),
    'is_default' => true
));

// Add FTP storage
backupscheduler_AddStorage(array(
    'storage_name' => 'Backup Server',
    'storage_key' => 'ftp_backup',
    'storage_type' => 'ftp',
    'config' => array(
        'host' => 'backup.example.com',
        'port' => 21,
        'username' => 'backup_user',
        'password' => 'xxx',
        'path' => '/backups/whmcs'
    )
));

// Get storages
$storages = backupscheduler_GetStorages();
```

## Configuration Options

| Option | Type | Default | Description |
|--------|------|---------|-------------|
| DefaultStorage | dropdown | local | Default storage |
| EnableNotifications | yesno | yes | Notifications |
| EnableEncryption | yesno | yes | Encrypt backups |
| CompressionLevel | dropdown | 5 | Compression 1-9 |
| MaxBackupSize | text | 5GB | Max backup size |
| ConcurrentBackups | text | 2 | Concurrent jobs |

## Backup Scope

| Scope | Description |
|-------|-------------|
| database | MySQL database |
| config | configuration.php |
| attachments | Client files |
| templates | Templates |
| modules | Custom modules |

## Frequency

| Frequency | Description |
|-----------|-------------|
| hourly | Every hour |
| daily | Once per day |
| weekly | Weekly |
| monthly | Monthly |

## Storage Types

| Type | Description |
|------|-------------|
| local | Local filesystem |
| s3 | Amazon S3 |
| ftp | FTP server |
| sftp | SFTP server |

## Retention Policies

```php
// Keep 7 daily backups
'retention_count' => 7

// Delete after 30 days
'retention_days' => 30

// Combined: Keep 7 backups OR 30 days
```

## Cron Integration

Add to WHMCS cron or system cron:

```php
// Run every 5 minutes
// Check for due backups and execute
$dueSchedules = backupscheduler_GetDueSchedules();
foreach ($dueSchedules as $schedule) {
    backupscheduler_RunBackup($schedule->id);
}
```

## Database Tables

- `mod_backupscheduler_schedules` - Backup schedules
- `mod_backupscheduler_jobs` - Backup jobs
- `mod_backupscheduler_storages` - Storage targets
- `mod_backupscheduler_logs` - Job logs
- `mod_backupscheduler_restores` - Restore jobs
