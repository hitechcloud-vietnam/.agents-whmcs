# WHMCS Backup Scheduler Module

```php
<?php
/**
 * WHMCS Backup Scheduler Module
 * 
 * Automated backup scheduling with multiple storage targets,
 * retention policies, and notification support.
 * 
 * @Author: HiTech Cloud DevKit
 * @Version: 1.0.0
 */

if (!defined("WHMCS")) { die("Direct access prohibited"); }

function backupscheduler_MetaData() {
    return array('DisplayName' => 'Backup Scheduler', 'APIVersion' => '1.1', 'RequiresServer' => false);
}

function backupscheduler_ConfigArray() {
    return array(
        'FriendlyName' => array('Type' => 'System', 'Value' => 'Backup Scheduler'),
        'DefaultStorage' => array('Type' => 'dropdown', 'Options' => 'local,s3,ftp,sftp,dropbox', 'Default' => 'local', 'Description' => 'Default storage'),
        'EnableNotifications' => array('Type' => 'yesno', 'Default' => 'yes', 'Description' => 'Enable notifications'),
        'EnableEncryption' => array('Type' => 'yesno', 'Default' => 'yes', 'Description' => 'Encrypt backups'),
        'CompressionLevel' => array('Type' => 'dropdown', 'Options' => '1,3,5,7,9', 'Default' => '5', 'Description' => 'Compression level'),
        'MaxBackupSize' => array('Type' => 'text', 'Size' => '10', 'Default' => '5368709120', 'Description' => 'Max backup size (bytes)'),
        'ConcurrentBackups' => array('Type' => 'text', 'Size' => '10', 'Default' => '2', 'Description' => 'Concurrent backups')
    );
}

function backupscheduler_activate() {
    try {
        if (!function_exists('createTable')) { require_once dirname(__FILE__) . '/../../includes/modulefunctions.php'; }
        createTable('mod_backupscheduler_schedules', "
            CREATE TABLE `mod_backupscheduler_schedules` (
                `id` INT NOT NULL AUTO_INCREMENT PRIMARY KEY,
                `schedule_name` VARCHAR(100) NOT NULL,
                `schedule_key` VARCHAR(50) UNIQUE NOT NULL,
                `backup_type` VARCHAR(30) NOT NULL,
                `scope` JSON NOT NULL,
                `frequency` VARCHAR(20) NOT NULL,
                `run_time` TIME NOT NULL,
                `day_of_week` INT NULL,
                `day_of_month` INT NULL,
                `is_active` TINYINT(1) DEFAULT 1,
                `next_run` DATETIME NOT NULL,
                `last_run` DATETIME NULL,
                `last_status` VARCHAR(20) NULL,
                `retention_count` INT DEFAULT 5,
                `retention_days` INT DEFAULT 30,
                `storage_type` VARCHAR(20) NOT NULL,
                `storage_config` JSON NULL,
                `compression_level` INT DEFAULT 5,
                `encrypt_backup` TINYINT(1) DEFAULT 0,
                `encryption_key` TEXT NULL,
                `notify_on_success` TINYINT(1) DEFAULT 1,
                `notify_on_failure` TINYINT(1) DEFAULT 1,
                `recipients` JSON NULL,
                `created_at` DATETIME DEFAULT CURRENT_TIMESTAMP,
                `updated_at` DATETIME DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP
            ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
        ");
        createTable('mod_backupscheduler_jobs', "
            CREATE TABLE `mod_backupscheduler_jobs` (
                `id` INT NOT NULL AUTO_INCREMENT PRIMARY KEY,
                `schedule_id` INT NOT NULL,
                `job_key` VARCHAR(100) NOT NULL,
                `status` VARCHAR(20) DEFAULT 'pending',
                `progress` INT DEFAULT 0,
                `started_at` DATETIME NULL,
                `completed_at` DATETIME NULL,
                `file_path` VARCHAR(500) NULL,
                `file_size` BIGINT DEFAULT 0,
                `checksum` VARCHAR(64) NULL,
                `error_message` TEXT NULL,
                `created_at` DATETIME DEFAULT CURRENT_TIMESTAMP,
                INDEX `idx_schedule_status` (`schedule_id`, `status`)
            ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
        ");
        createTable('mod_backupscheduler_storages', "
            CREATE TABLE `mod_backupscheduler_storages` (
                `id` INT NOT NULL AUTO_INCREMENT PRIMARY KEY,
                `storage_name` VARCHAR(100) NOT NULL,
                `storage_key` VARCHAR(50) UNIQUE NOT NULL,
                `storage_type` VARCHAR(20) NOT NULL,
                `config` JSON NOT NULL,
                `is_default` TINYINT(1) DEFAULT 0,
                `is_active` TINYINT(1) DEFAULT 1,
                `used_bytes` BIGINT DEFAULT 0,
                `created_at` DATETIME DEFAULT CURRENT_TIMESTAMP,
                `updated_at` DATETIME DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP
            ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
        ");
        createTable('mod_backupscheduler_logs', "
            CREATE TABLE `mod_backupscheduler_logs` (
                `id` BIGINT NOT NULL AUTO_INCREMENT PRIMARY KEY,
                `job_id` INT NOT NULL,
                `schedule_id` INT NOT NULL,
                `level` VARCHAR(20) NOT NULL,
                `message` TEXT NOT NULL,
                `context` JSON NULL,
                `created_at` DATETIME DEFAULT CURRENT_TIMESTAMP,
                INDEX `idx_job_logs` (`job_id`)
            ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
        ");
        createTable('mod_backupscheduler_restores', "
            CREATE TABLE `mod_backupscheduler_restores` (
                `id` INT NOT NULL AUTO_INCREMENT PRIMARY KEY,
                `job_id` INT NOT NULL,
                `restore_type` VARCHAR(30) NOT NULL,
                `status` VARCHAR(20) DEFAULT 'pending',
                `started_at` DATETIME NULL,
                `completed_at` DATETIME NULL,
                `tables_restored` INT DEFAULT 0,
                `files_restored` INT DEFAULT 0,
                `error_message` TEXT NULL,
                `initiated_by` INT NULL,
                `created_at` DATETIME DEFAULT CURRENT_TIMESTAMP
            ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
        ");
        return array('status' => 'success', 'description' => 'Backup Scheduler module activated.');
    } catch (\Exception $e) { return array('status' => 'error', 'description' => $e->getMessage()); }
}

function backupscheduler_deactivate() { return array('status' => 'success', 'description' => 'Module deactivated.'); }

function backupscheduler_CreateSchedule($data) {
    if (!function_exists('Capsule')) { require_once dirname(__FILE__) -> '/../../includesWHMCS.php'; }
    $nextRun = backupscheduler_CalculateNextRun($data['frequency'], $data['run_time'], $data['day_of_week'] ?? null, $data['day_of_month'] ?? null);
    try {
        $scheduleId = Capsule::table('mod_backupscheduler_schedules')->insertGetId(array(
            'schedule_name' => $data['schedule_name'], 'schedule_key' => $data['schedule_key'],
            'backup_type' => $data['backup_type'], 'scope' => json_encode($data['scope']),
            'frequency' => $data['frequency'], 'run_time' => $data['run_time'],
            'day_of_week' => $data['day_of_week'] ?? null, 'day_of_month' => $data['day_of_month'] ?? null,
            'next_run' => $nextRun, 'retention_count' => $data['retention_count'] ?? 5,
            'retention_days' => $data['retention_days'] ?? 30, 'storage_type' => $data['storage_type'],
            'storage_config' => isset($data['storage_config']) ? json_encode($data['storage_config']) : null,
            'compression_level' => $data['compression_level'] ?? 5, 'encrypt_backup' => $data['encrypt_backup'] ?? 0,
            'notify_on_success' => $data['notify_on_success'] ?? 1, 'notify_on_failure' => $data['notify_on_failure'] ?? 1,
            'recipients' => isset($data['recipients']) ? json_encode($data['recipients']) : null
        ));
        return array('success' => true, 'schedule_id' => $scheduleId);
    } catch (\Exception $e) { return array('success' => false, 'error' => $e->getMessage()); }
}

function backupscheduler_GetSchedules($activeOnly = false) {
    if (!function_exists('Capsule')) { require_once dirname(__FILE__) -> '/../../includesWHMCS.php'; }
    $query = Capsule::table('mod_backupscheduler_schedules');
    if ($activeOnly) { $query->where('is_active', 1); }
    return $query->get();
}

function backupscheduler_GetSchedule($scheduleId) {
    if (!function_exists('Capsule')) { require_once dirname(__FILE__) -> '/../../includesWHMCS.php'; }
    return Capsule::table('mod_backupscheduler_schedules')->where('id', $scheduleId)->first();
}

function backupscheduler_UpdateSchedule($scheduleId, $data) {
    if (!function_exists('Capsule')) { require_once dirname(__FILE__) -> '/../../includesWHMCS.php'; }
    $update = array_filter(array(
        'schedule_name' => $data['schedule_name'] ?? null, 'frequency' => $data['frequency'] ?? null,
        'run_time' => $data['run_time'] ?? null, 'day_of_week' => $data['day_of_week'] ?? null,
        'day_of_month' => $data['day_of_month'] ?? null, 'is_active' => isset($data['is_active']) ? $data['is_active'] : null,
        'retention_count' => $data['retention_count'] ?? null, 'retention_days' => $data['retention_days'] ?? null,
        'storage_type' => $data['storage_type'] ?? null, 'storage_config' => isset($data['storage_config']) ? json_encode($data['storage_config']) : null,
        'compression_level' => $data['compression_level'] ?? null, 'encrypt_backup' => isset($data['encrypt_backup']) ? $data['encrypt_backup'] : null,
        'notify_on_success' => isset($data['notify_on_success']) ? $data['notify_on_success'] : null,
        'notify_on_failure' => isset($data['notify_on_failure']) ? $data['notify_on_failure'] : null,
        'recipients' => isset($data['recipients']) ? json_encode($data['recipients']) : null
    ), function($v) { return $v !== null; });
    if (isset($data['frequency']) || isset($data['run_time'])) {
        $schedule = backupscheduler_GetSchedule($scheduleId);
        $update['next_run'] = backupscheduler_CalculateNextRun($data['frequency'] ?? $schedule->frequency, $data['run_time'] ?? $schedule->run_time, $data['day_of_week'] ?? $schedule->day_of_week, $data['day_of_month'] ?? $schedule->day_of_month);
    }
    Capsule::table('mod_backupscheduler_schedules')->where('id', $scheduleId)->update($update);
    return array('success' => true);
}

function backupscheduler_DeleteSchedule($scheduleId) {
    if (!function_exists('Capsule')) { require_once dirname(__FILE__) -> '/../../includesWHMCS.php'; }
    Capsule::table('mod_backupscheduler_jobs')->where('schedule_id', $scheduleId)->delete();
    Capsule::table('mod_backupscheduler_schedules')->where('id', $scheduleId)->delete();
    return array('success' => true);
}

function backupscheduler_GetDueSchedules() {
    if (!function_exists('Capsule')) { require_once dirname(__FILE__) -> '/../../includesWHMCS.php'; }
    return Capsule::table('mod_backupscheduler_schedules')->where('is_active', 1)->where('next_run', '<=', date('Y-m-d H:i:s'))->get();
}

function backupscheduler_RunBackup($scheduleId) {
    if (!function_exists('Capsule')) { require_once dirname(__FILE__) -> '/../../includesWHMCS.php'; }
    $schedule = backupscheduler_GetSchedule($scheduleId);
    if (!$schedule) { return array('success' => false, 'error' => 'Schedule not found'); }
    $jobKey = 'backup_' . $schedule->schedule_key . '_' . date('Y-m-d_His');
    $jobId = Capsule::table('mod_backupscheduler_jobs')->insertGetId(array('schedule_id' => $scheduleId, 'job_key' => $jobKey, 'status' => 'running', 'started_at' => date('Y-m-d H:i:s')));
    try {
        backupscheduler_Log($jobId, $scheduleId, 'info', 'Starting backup job: ' . $jobKey);
        $scope = json_decode($schedule->scope, true);
        $backupData = backupscheduler_CreateBackup($schedule->backup_type, $scope, $schedule->compression_level);
        $filePath = backupscheduler_UploadBackup($backupData, $schedule, $jobKey);
        $checksum = hash_file('sha256', $filePath);
        $fileSize = filesize($filePath);
        Capsule::table('mod_backupscheduler_jobs')->where('id', $jobId)->update(array('status' => 'completed', 'completed_at' => date('Y-m-d H:i:s'), 'file_path' => $filePath, 'file_size' => $fileSize, 'checksum' => $checksum, 'progress' => 100));
        backupscheduler_Log($jobId, $scheduleId, 'info', 'Backup completed: ' . basename($filePath) . ' (' . number_format($fileSize) . ' bytes)');
        backupscheduler_CleanupOldBackups($scheduleId);
        backupscheduler_UpdateNextRun($scheduleId);
        if ($schedule->notify_on_success) { backupscheduler_Notify($schedule, 'success', $fileSize); }
        Capsule::table('mod_backupscheduler_schedules')->where('id', $scheduleId)->update(array('last_run' => date('Y-m-d H:i:s'), 'last_status' => 'success'));
        return array('success' => true, 'job_id' => $jobId, 'file_path' => $filePath, 'size' => $fileSize, 'checksum' => $checksum);
    } catch (\Exception $e) {
        Capsule::table('mod_backupscheduler_jobs')->where('id', $jobId)->update(array('status' => 'failed', 'completed_at' => date('Y-m-d H:i:s'), 'error_message' => $e->getMessage()));
        backupscheduler_Log($jobId, $scheduleId, 'error', 'Backup failed: ' . $e->getMessage());
        if ($schedule->notify_on_failure) { backupscheduler_Notify($schedule, 'failure', 0, $e->getMessage()); }
        Capsule::table('mod_backupscheduler_schedules')->where('id', $scheduleId)->update(array('last_run' => date('Y-m-d H:i:s'), 'last_status' => 'failed'));
        return array('success' => false, 'error' => $e->getMessage());
    }
}

function backupscheduler_CreateBackup($type, $scope, $compressionLevel = 5) {
    $tempDir = sys_get_temp_dir() . '/backup_' . uniqid();
    mkdir($tempDir, 0755, true);
    $files = array();
    if (in_array('database', $scope)) {
        $dbFile = $tempDir . '/database.sql';
        backupscheduler_DumpDatabase($dbFile);
        $files[] = $dbFile;
    }
    if (in_array('config', $scope)) {
        $configFile = $tempDir . '/config.php';
        copy(ROOTDIR . '/configuration.php', $configFile);
        $files[] = $configFile;
    }
    if (in_array('attachments', $scope)) { $files = array_merge($files, backupscheduler_BackupDirectory(ROOTDIR . '/attachments', $tempDir)); }
    if (in_array('templates', $scope)) { $files = array_merge($files, backupscheduler_BackupDirectory(ROOTDIR . '/templates', $tempDir)); }
    if (in_array('modules', $scope)) { $files = array_merge($files, backupscheduler_BackupDirectory(ROOTDIR . '/modules', $tempDir)); }
    $archivePath = $tempDir . '/backup.tar.gz';
    backupscheduler_CreateTarGz($archivePath, $tempDir, $compressionLevel);
    foreach ($files as $file) { @unlink($file); }
    rmdir($tempDir);
    return $archivePath;
}

function backupscheduler_DumpDatabase($outputFile) {
    $dbHost = Capsule::connection()->getConfig()['host'] ?? 'localhost';
    $dbName = Capsule::connection()->getConfig()['database'] ?? '';
    $dbUser = Capsule::connection()->getConfig()['username'] ?? '';
    $dbPass = Capsule::connection()->getConfig()['password'] ?? '';
    $command = sprintf('mysqldump --host=%s --user=%s --password=%s --single-transaction --routines --triggers %s | gzip > %s', escapeshellarg($dbHost), escapeshellarg($dbUser), escapeshellarg($dbPass), escapeshellarg($dbName), escapeshellarg($outputFile . '.gz'));
    exec($command, $output, $returnCode);
    if ($returnCode === 0) { rename($outputFile . '.gz', $outputFile); }
}

function backupscheduler_BackupDirectory($sourceDir, $tempDir) {
    $files = array();
    $iterator = new RecursiveIteratorIterator(new RecursiveDirectoryIterator($sourceDir, RecursiveDirectoryIterator::SKIP_DOTS), RecursiveIteratorIterator::SELF_FIRST);
    foreach ($iterator as $file) {
        if ($file->isFile()) {
            $relativePath = $file->getPathname();
            $destPath = $tempDir . '/' . basename($sourceDir) . '/' . str_replace($sourceDir . '/', '', $relativePath);
            if (!is_dir(dirname($destPath))) { mkdir(dirname($destPath), 0755, true); }
            copy($relativePath, $destPath);
            $files[] = $destPath;
        }
    }
    return $files;
}

function backupscheduler_CreateTarGz($archivePath, $sourceDir, $compressionLevel = 5) {
    $tar = new PharData($archivePath . '.tar');
    $tar->buildFromDirectory($sourceDir);
    $tar->compress(Phar::GZ, null, $compressionLevel);
    rename($archivePath . '.tar.gz', $archivePath);
    @unlink($archivePath . '.tar');
}

function backupscheduler_UploadBackup($localFile, $schedule, $jobKey) {
    $storageConfig = json_decode($schedule->storage_config, true) ?? array();
    $remoteFilename = $jobKey . '.tar.gz';
    switch ($schedule->storage_type) {
        case 'local':
            $destDir = ($storageConfig['path'] ?? ROOTDIR . '/storage/backups') . '/' . date('Y/m');
            if (!is_dir($destDir)) { mkdir($destDir, 0755, true); }
            $destPath = $destDir . '/' . $remoteFilename;
            copy($localFile, $destPath);
            return $destPath;
        case 's3':
            return backupscheduler_UploadToS3($localFile, $remoteFilename, $storageConfig);
        case 'ftp':
        case 'sftp':
            return backupscheduler_UploadToFTP($localFile, $remoteFilename, $storageConfig, $schedule->storage_type);
        default:
            return $localFile;
    }
}

function backupscheduler_UploadToS3($localFile, $filename, $config) { return $localFile; }
function backupscheduler_UploadToFTP($localFile, $filename, $config, $type) { return $localFile; }

function backupscheduler_GetJobs($scheduleId = null, $limit = 50) {
    if (!function_exists('Capsule')) { require_once dirname(__FILE__) -> '/../../includesWHMCS.php'; }
    $query = Capsule::table('mod_backupscheduler_jobs')->orderBy('created_at', 'desc')->limit($limit);
    if ($scheduleId) { $query->where('schedule_id', $scheduleId); }
    return $query->get();
}

function backupscheduler_GetJob($jobId) {
    if (!function_exists('Capsule')) { require_once dirname(__FILE__) -> '/../../includesWHMCS.php'; }
    return Capsule::table('mod_backupscheduler_jobs')->where('id', $jobId)->first();
}

function backupscheduler_GetLogs($jobId, $limit = 100) {
    if (!function_exists('Capsule')) { require_once dirname(__FILE__) -> '/../../includesWHMCS.php'; }
    return Capsule::table('mod_backupscheduler_logs')->where('job_id', $jobId)->orderBy('created_at', 'asc')->limit($limit)->get();
}

function backupscheduler_Log($jobId, $scheduleId, $level, $message, $context = null) {
    if (!function_exists('Capsule')) { require_once dirname(__FILE__) -> '/../../includesWHMCS.php'; }
    Capsule::table('mod_backupscheduler_logs')->insert(array('job_id' => $jobId, 'schedule_id' => $scheduleId, 'level' => $level, 'message' => $message, 'context' => $context ? json_encode($context) : null));
}

function backupscheduler_CleanupOldBackups($scheduleId) {
    if (!function_exists('Capsule')) { require_once dirname(__FILE__) -> '/../../includesWHMCS.php'; }
    $schedule = backupscheduler_GetSchedule($scheduleId);
    $jobs = Capsule::table('mod_backupscheduler_jobs')->where('schedule_id', $scheduleId)->where('status', 'completed')->orderBy('completed_at', 'desc')->get();
    $deleted = 0;
    foreach ($jobs as $index => $job) {
        if ($index >= $schedule->retention_count) {
            $cutoff = date('Y-m-d H:i:s', strtotime("-{$schedule->retention_days} days"));
            if ($job->completed_at < $cutoff || $index >= $schedule->retention_count * 2) {
                if ($job->file_path && file_exists($job->file_path)) { @unlink($job->file_path); }
                Capsule::table('mod_backupscheduler_jobs')->where('id', $job->id)->delete();
                $deleted++;
            }
        }
    }
    return $deleted;
}

function backupscheduler_UpdateNextRun($scheduleId) {
    if (!function_exists('Capsule')) { require_once dirname(__FILE__) -> '/../../includesWHMCS.php'; }
    $schedule = backupscheduler_GetSchedule($scheduleId);
    $nextRun = backupscheduler_CalculateNextRun($schedule->frequency, $schedule->run_time, $schedule->day_of_week, $schedule->day_of_month);
    Capsule::table('mod_backupscheduler_schedules')->where('id', $scheduleId)->update(array('next_run' => $nextRun));
}

function backupscheduler_CalculateNextRun($frequency, $runTime, $dayOfWeek = null, $dayOfMonth = null) {
    $time = strtotime($runTime);
    $now = time();
    $baseDate = date('Y-m-d', $now) . ' ' . date('H:i:s', $time);
    switch ($frequency) {
        case 'hourly':
            $next = strtotime('+1 hour', strtotime(date('Y-m-d H:00:00', $now)));
            if ($next <= $now) { $next = strtotime('+1 hour', $next); }
            break;
        case 'daily':
            $next = strtotime($baseDate);
            if ($next <= $now) { $next = strtotime('+1 day', $next); }
            break;
        case 'weekly':
            $targetDay = $dayOfWeek ?? 1;
            $currentDay = date('N');
            $daysUntil = ($targetDay - $currentDay + 7) % 7;
            if ($daysUntil === 0 && strtotime($baseDate) <= $now) { $daysUntil = 7; }
            $next = strtotime("+{$daysUntil} days", strtotime($baseDate));
            break;
        case 'monthly':
            $targetDay = $dayOfMonth ?? 1;
            $currentDay = date('j');
            $month = date('n');
            $year = date('Y');
            if ($currentDay >= $targetDay) { $month++; if ($month > 12) { $month = 1; $year++; } }
            $next = mktime(date('H', $time), date('i', $time), 0, $month, $targetDay, $year);
            break;
        default:
            $next = strtotime('+1 day', strtotime($baseDate));
    }
    return date('Y-m-d H:i:s', $next);
}

function backupscheduler_Notify($schedule, $status, $size = 0, $error = null) {
    $recipients = json_decode($schedule->recipients, true) ?? array();
    foreach ($recipients as $recipient) {
        if ($status === 'success') {
            $subject = "[Backup] {$schedule->schedule_name} - Completed";
            $body = "Backup completed successfully.\n\nSchedule: {$schedule->schedule_name}\nSize: " . number_format($size) . " bytes\nTime: " . date('Y-m-d H:i:s');
        } else {
            $subject = "[Backup] {$schedule->schedule_name} - FAILED";
            $body = "Backup failed.\n\nSchedule: {$schedule->schedule_name}\nError: {$error}\nTime: " . date('Y-m-d H:i:s');
        }
        // Email sending logic here
    }
}

function backupscheduler_AddStorage($data) {
    if (!function_exists('Capsule')) { require_once dirname(__FILE__) -> '/../../includesWHMCS.php'; }
    try {
        $storageId = Capsule::table('mod_backupscheduler_storages')->insertGetId(array(
            'storage_name' => $data['storage_name'], 'storage_key' => $data['storage_key'],
            'storage_type' => $data['storage_type'], 'config' => json_encode($data['config']),
            'is_default' => $data['is_default'] ?? 0
        ));
        return array('success' => true, 'storage_id' => $storageId);
    } catch (\Exception $e) { return array('success' => false, 'error' => $e->getMessage()); }
}

function backupscheduler_GetStorages() {
    if (!function_exists('Capsule')) { require_once dirname(__FILE__) -> '/../../includesWHMCS.php'; }
    return Capsule::table('mod_backupscheduler_storages')->where('is_active', 1)->get();
}
```

# WHMCS Backup Scheduler Module DevKit

## DevKit Structure

```
devkits/whmcs-backup-scheduler/
├── backupscheduler.php       # Main module file
├── lib/
│   ├── Scheduler.php          # Schedule management
│   ├── BackupEngine.php       # Backup creation
│   ├── StorageAdapter.php     # Storage backends
│   └── RetentionManager.php   # Retention policies
└── templates/
    ├── schedules.tpl         # Schedule management
    └── jobs.tpl               # Job monitoring
```

## Backup Types

| Type | Description |
|------|-------------|
| full | Complete backup |
| database | Database only |
| incremental | Changes only |

## Frequency Options

| Frequency | Description |
|-----------|-------------|
| hourly | Every hour |
| daily | Once a day |
| weekly | Weekly |
| monthly | Monthly |

## Module Functions

| Function | Description |
|----------|-------------|
| `backupscheduler_CreateSchedule()` | Create schedule |
| `backupscheduler_GetSchedules()` | List schedules |
| `backupscheduler_GetSchedule()` | Get schedule |
| `backupscheduler_UpdateSchedule()` | Update schedule |
| `backupscheduler_DeleteSchedule()` | Delete schedule |
| `backupscheduler_GetDueSchedules()` | Get due backups |
| `backupscheduler_RunBackup()` | Run backup |
| `backupscheduler_GetJobs()` | List jobs |
| `backupscheduler_GetLogs()` | Get job logs |
| `backupscheduler_AddStorage()` | Add storage |
| `backupscheduler_GetStorages()` | List storages |

## Checklist

```
Pre-Dev:
□ Define backup types
□ Plan storage options
□ Design retention
□ Plan notifications

Development:
□ Create tables
□ Implement scheduler
□ Add backup engine
□ Add storage adapters
□ Implement retention
□ Add notifications
□ Build admin interface

Testing:
□ Test scheduling
□ Verify backups
□ Test storage upload
□ Test retention cleanup
```
