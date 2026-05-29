# WHMCS Backup Automation Workflow

## Overview
This workflow implements automated backup systems for WHMCS including database, files, and off-site storage.

## Prerequisites
- WHMCS installation
- Sufficient storage for backups
- Optional: Cloud storage credentials (AWS S3, Google Cloud, etc.)

## Step-by-Step Process

### Step 1: Create Backup Manager
```php
<?php
// /includes/backup/BackupManager.php

namespace WHMCS\Backup;

class BackupManager
{
    private $backupDir;
    private $retentionDays = 30;

    public function __construct()
    {
        $this->backupDir = getConfig('backup_directory') ?? __DIR__ . '/../../backups';
        $this->ensureBackupDir();
    }

    /**
     * Create full backup
     */
    public function createFullBackup(array $options = []): array
    {
        $backupId = uniqid('backup_');
        $backupPath = $this->backupDir . '/' . $backupId;

        mkdir($backupPath, 0755, true);

        $results = [
            'backup_id' => $backupId,
            'started_at' => date('Y-m-d H:i:s'),
            'components' => []
        ];

        // Database backup
        if ($options['include_database'] ?? true) {
            $results['components']['database'] = $this->backupDatabase($backupPath);
        }

        // Files backup
        if ($options['include_files'] ?? true) {
            $results['components']['files'] = $this->backupFiles($backupPath);
        }

        // Configuration backup
        if ($options['include_config'] ?? true) {
            $results['components']['config'] = $this->backupConfig($backupPath);
        }

        // Create archive
        $results['archive'] = $this->createArchive($backupPath, $backupId);

        // Cleanup individual files
        $this->cleanupTempFiles($backupPath);

        $results['completed_at'] = date('Y-m-d H:i:s');
        $results['total_size'] = filesize($results['archive']);

        // Log backup
        $this->logBackup($results);

        // Cleanup old backups
        $this->cleanupOldBackups();

        // Upload to off-site storage
        if ($options['upload_to_cloud'] ?? false) {
            $results['cloud_upload'] = $this->uploadToCloud($results['archive']);
        }

        return $results;
    }

    /**
     * Backup database
     */
    private function backupDatabase(string $backupPath): array
    {
        $filename = $backupPath . '/database.sql';
        $whmcsDir = dirname(__DIR__, 2);

        // Get database credentials
        $dbHost = getConfig('mysql_host');
        $dbName = getConfig('mysql_database');
        $dbUser = getConfig('mysql_username');
        $dbPass = getConfig('mysql_password');

        $command = "mysqldump -h {$dbHost} -u {$dbUser} -p" . escapeshellarg($dbPass) .
            " --single-transaction {$dbName} > {$filename}";

        exec($command, $output, $returnCode);

        return [
            'file' => $filename,
            'size' => file_exists($filename) ? filesize($filename) : 0,
            'success' => $returnCode === 0
        ];
    }

    /**
     * Backup files
     */
    private function backupFiles(string $backupPath): array
    {
        $filename = $backupPath . '/files.tar.gz';
        $whmcsDir = dirname(__DIR__, 2);

        $excludeDirs = ['backups', 'cache', 'temp', 'logs', '.git', 'node_modules'];

        $excludeArgs = '';
        foreach ($excludeDirs as $dir) {
            $excludeArgs .= " --exclude='{$dir}'";
        }

        $command = "cd " . escapeshellarg($whmcsDir) . " && tar -czf " .
            escapeshellarg($filename) . " {$excludeArgs} .";

        exec($command, $output, $returnCode);

        return [
            'file' => $filename,
            'size' => file_exists($filename) ? filesize($filename) : 0,
            'success' => $returnCode === 0
        ];
    }

    /**
     * Backup configuration files
     */
    private function backupConfig(string $backupPath): array
    {
        $whmcsDir = dirname(__DIR__, 2);
        $configFiles = [
            $whmcsDir . '/configuration.php',
            $whmcsDir . '/.htaccess'
        ];

        $filename = $backupPath . '/config.tar.gz';
        $tempDir = $backupPath . '/config_files';

        mkdir($tempDir, 0755, true);

        foreach ($configFiles as $file) {
            if (file_exists($file)) {
                copy($file, $tempDir . '/' . basename($file));
            }
        }

        exec("cd " . escapeshellarg($tempDir) . " && tar -czf " .
            escapeshellarg($filename) . " .");

        return [
            'file' => $filename,
            'size' => file_exists($filename) ? filesize($filename) : 0,
            'success' => true
        ];
    }

    /**
     * Create compressed archive
     */
    private function createArchive(string $sourcePath, string $backupId): string
    {
        $archivePath = $this->backupDir . '/' . $backupId . '.tar.gz';

        exec("tar -czf " . escapeshellarg($archivePath) . " -C " .
            escapeshellarg($sourcePath) . " .");

        return $archivePath;
    }

    /**
     * Upload to cloud storage
     */
    private function uploadToCloud(string $filePath): array
    {
        $provider = getConfig('backup_cloud_provider');

        switch ($provider) {
            case 's3':
                return $this->uploadToS3($filePath);
            case 'gcs':
                return $this->uploadToGCS($filePath);
            default:
                return ['success' => false, 'error' => 'Unknown provider'];
        }
    }

    /**
     * Upload to AWS S3
     */
    private function uploadToS3(string $filePath): array
    {
        $s3 = new S3Client([
            'version' => 'latest',
            'region' => getConfig('aws_region'),
            'credentials' => [
                'key' => getConfig('aws_access_key'),
                'secret' => getConfig('aws_secret_key')
            ]
        ]);

        $bucket = getConfig('aws_backup_bucket');
        $key = 'whmcs-backups/' . basename($filePath);

        try {
            $s3->putObject([
                'Bucket' => $bucket,
                'Key' => $key,
                'SourceFile' => $filePath
            ]);

            return ['success' => true, 'location' => $key];
        } catch (Exception $e) {
            return ['success' => false, 'error' => $e->getMessage()];
        }
    }

    /**
     * Cleanup old backups
     */
    private function cleanupOldBackups(): int
    {
        $cutoffDate = date('Y-m-d', strtotime("-{$this->retentionDays} days"));

        $oldBackups = glob($this->backupDir . '/backup_*.tar.gz');

        $deleted = 0;

        foreach ($oldBackups as $backup) {
            if (filemtime($backup) < strtotime($cutoffDate)) {
                if (unlink($backup)) {
                    $deleted++;
                    // Also remove from database log
                    Capsule::table('mod_backup_log')
                        ->where('archive_path', $backup)
                        ->update(['deleted_at' => date('Y-m-d H:i:s')]);
                }
            }
        }

        return $deleted;
    }

    /**
     * Log backup
     */
    private function logBackup(array $results)
    {
        Capsule::table('mod_backup_log')->insert([
            'backup_id' => $results['backup_id'],
            'components' => json_encode($results['components']),
            'archive_path' => $results['archive'],
            'total_size' => $results['total_size'],
            'started_at' => $results['started_at'],
            'completed_at' => $results['completed_at'],
            'created_at' => date('Y-m-d H:i:s')
        ]);
    }

    private function ensureBackupDir()
    {
        if (!is_dir($this->backupDir)) {
            mkdir($this->backupDir, 0755, true);
        }
    }

    private function cleanupTempFiles(string $path)
    {
        array_map('unlink', glob($path . '/*'));
        rmdir($path);
    }
}
```

### Step 2: Create Backup Cron Hook
```php
<?php
// /includes/hooks/backup_hooks.php

use WHMCS\Backup\BackupManager;

add_hook('DailyCronJob', 1, function($vars) {
    $backupManager = new BackupManager();

    // Daily incremental backup
    $result = $backupManager->createFullBackup([
        'include_database' => true,
        'include_files' => true,
        'include_config' => true,
        'upload_to_cloud' => true
    ]);

    if ($result['components']['database']['success']) {
        logActivity("Daily backup completed: {$result['backup_id']}");
    } else {
        sendAdminEmail('Backup Failed', [
            'backup_id' => $result['backup_id'],
            'error' => 'Database backup failed'
        ]);
    }

    return $result;
});

add_hook('WeeklyCronJob', 1, function($vars) {
    $backupManager = new BackupManager();

    // Weekly full backup with extended retention
    $result = $backupManager->createFullBackup([
        'include_database' => true,
        'include_files' => true,
        'include_config' => true,
        'upload_to_cloud' => true
    ]);

    return $result;
});
```

## Related Workflows
- [WHMCS Backup Strategy](./whmcs-backup-strategy.md)
- [WHMCS Disaster Recovery](./whmcs-disaster-recovery.md)