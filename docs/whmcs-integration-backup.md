# WHMCS Backup Integration

Complete guide for backup strategies and integrations.

## Overview

Implement comprehensive backup solutions for WHMCS data and configurations.

## Database Backup

### MySQL Backup Script

```php
<?php
/**
 * Database backup class
 */
class DatabaseBackup
{
    private string $host;
    private string $database;
    private string $username;
    private string $password;
    private string $backupDir;
    
    public function __construct(array $config)
    {
        $this->host = $config['host'];
        $this->database = $config['database'];
        $this->username = $config['username'];
        $this->password = $config['password'];
        $this->backupDir = $config['backup_dir'] ?? sys_get_temp_dir();
    }
    
    /**
     * Create database backup
     */
    public function backup(): array
    {
        $filename = $this->database . '_' . date('Y-m-d_H-i-s') . '.sql.gz';
        $filepath = $this->backupDir . '/' . $filename;
        
        $command = sprintf(
            'mysqldump -h %s -u %s -p%s %s | gzip > %s',
            escapeshellarg($this->host),
            escapeshellarg($this->username),
            escapeshellarg($this->password),
            escapeshellarg($this->database),
            escapeshellarg($filepath)
        );
        
        exec($command, $output, $returnCode);
        
        if ($returnCode !== 0) {
            return [
                'success' => false,
                'error' => 'Backup command failed',
                'output' => implode("\n", $output),
            ];
        }
        
        return [
            'success' => true,
            'filepath' => $filepath,
            'filename' => $filename,
            'size' => filesize($filepath),
        ];
    }
    
    /**
     * Restore from backup
     */
    public function restore(string $backupFile): array
    {
        if (!file_exists($backupFile)) {
            return [
                'success' => false,
                'error' => 'Backup file not found',
            ];
        }
        
        $tempFile = sys_get_temp_dir() . '/restore_' . uniqid() . '.sql';
        
        // Decompress if gzipped
        if (strpos($backupFile, '.gz') !== false) {
            $fp = gzopen($backupFile, 'rb');
            $out = fopen($tempFile, 'wb');
            
            while (!gzeof($fp)) {
                fwrite($out, gzread($fp, 4096));
            }
            
            gzclose($fp);
            fclose($out);
        } else {
            copy($backupFile, $tempFile);
        }
        
        $command = sprintf(
            'mysql -h %s -u %s -p%s %s < %s',
            escapeshellarg($this->host),
            escapeshellarg($this->username),
            escapeshellarg($this->password),
            escapeshellarg($this->database),
            escapeshellarg($tempFile)
        );
        
        exec($command, $output, $returnCode);
        
        // Cleanup
        unlink($tempFile);
        
        return [
            'success' => $returnCode === 0,
            'output' => implode("\n", $output),
        ];
    }
    
    /**
     * List available backups
     */
    public function listBackups(): array
    {
        $backups = [];
        $files = glob($this->backupDir . '/' . $this->database . '_*.sql*');
        
        foreach ($files as $file) {
            $backups[] = [
                'filename' => basename($file),
                'filepath' => $file,
                'size' => filesize($file),
                'modified' => date('Y-m-d H:i:s', filemtime($file)),
            ];
        }
        
        // Sort by date descending
        usort($backups, fn($a, $b) => strcmp($b['modified'], $a['modified']));
        
        return $backups;
    }
}
```

## File Backup

```php
<?php
/**
 * File backup class
 */
class FileBackup
{
    private string $sourceDir;
    private string $backupDir;
    private array $excludePatterns;
    
    public function __construct(string $sourceDir, string $backupDir)
    {
        $this->sourceDir = rtrim($sourceDir, '/');
        $this->backupDir = rtrim($backupDir, '/');
        $this->excludePatterns = [
            'cache/*',
            'logs/*',
            'temp/*',
            'attachments/*',
            '*.log',
            '.git/*',
            'node_modules/*',
        ];
    }
    
    /**
     * Create file backup
     */
    public function backup(): array
    {
        $filename = 'files_' . date('Y-m-d_H-i-s') . '.tar.gz';
        $filepath = $this->backupDir . '/' . $filename;
        
        // Build exclude options
        $excludeArgs = '';
        foreach ($this->excludePatterns as $pattern) {
            $excludeArgs .= ' --exclude=' . escapeshellarg($pattern);
        }
        
        $command = sprintf(
            'tar -czf %s -C %s . %s',
            escapeshellarg($filepath),
            escapeshellarg($this->sourceDir),
            $excludeArgs
        );
        
        exec($command, $output, $returnCode);
        
        if ($returnCode !== 0) {
            return [
                'success' => false,
                'error' => 'File backup failed',
            ];
        }
        
        return [
            'success' => true,
            'filepath' => $filepath,
            'filename' => $filename,
            'size' => filesize($filepath),
        ];
    }
    
    /**
     * Extract backup
     */
    public function restore(string $backupFile, string $targetDir = null): array
    {
        $targetDir = $targetDir ?? $this->sourceDir;
        
        $command = sprintf(
            'tar -xzf %s -C %s',
            escapeshellarg($backupFile),
            escapeshellarg($targetDir)
        );
        
        exec($command, $output, $returnCode);
        
        return [
            'success' => $returnCode === 0,
        ];
    }
}
```

## Cloud Backup Integration

### S3 Backup

```php
<?php
/**
 * S3 backup storage
 */
class S3BackupStorage
{
    private string $bucket;
    private S3StorageClient $s3;
    
    public function __construct(S3StorageClient $s3, string $bucket)
    {
        $this->s3 = $s3;
        $this->bucket = $bucket;
    }
    
    /**
     * Upload backup to S3
     */
    public function upload(string $localFile, string $remotePath = null): array
    {
        $remotePath = $remotePath ?? 'backups/' . date('Y/m/d/') . basename($localFile);
        
        $success = $this->s3->uploadFile($localFile, $remotePath);
        
        return [
            'success' => $success,
            'remote_path' => $remotePath,
        ];
    }
    
    /**
     * Download backup from S3
     */
    public function download(string $remotePath, string $localFile): bool
    {
        return $this->s3->downloadFile($remotePath, $localFile);
    }
    
    /**
     * List backups in S3
     */
    public function listBackups(string $prefix = 'backups/'): array
    {
        // Using AWS SDK or API
        $ch = curl_init();
        // ... list objects with prefix
        return []; // Implementation depends on S3 client
    }
    
    /**
     * Delete old backups
     */
    public function cleanup(int $daysToKeep = 30): int
    {
        $backups = $this->listBackups();
        $deleted = 0;
        $cutoff = strtotime("-{$daysToKeep} days");
        
        foreach ($backups as $backup) {
            if ($backup['last_modified'] < $cutoff) {
                // Delete backup
                $deleted++;
            }
        }
        
        return $deleted;
    }
}
```

## Automated Backup Manager

```php
<?php
/**
 * Automated backup manager
 */
class BackupManager
{
    private DatabaseBackup $dbBackup;
    private FileBackup $fileBackup;
    private S3BackupStorage $remoteStorage;
    
    public function __construct(array $config)
    {
        $this->dbBackup = new DatabaseBackup($config['database']);
        $this->fileBackup = new FileBackup($config['whmcs_path'], $config['backup_dir']);
        
        if (!empty($config['s3_bucket'])) {
            $s3 = new S3StorageClient($config['s3']);
            $this->remoteStorage = new S3BackupStorage($s3, $config['s3_bucket']);
        }
    }
    
    /**
     * Run full backup
     */
    public function runBackup(bool $uploadToRemote = true): array
    {
        $results = [
            'started_at' => date('Y-m-d H:i:s'),
            'completed_at' => null,
            'success' => true,
            'backups' => [],
            'errors' => [],
        ];
        
        // Database backup
        logActivity('Backup: Starting database backup');
        $dbResult = $this->dbBackup->backup();
        
        if ($dbResult['success']) {
            $results['backups']['database'] = $dbResult;
            logActivity('Backup: Database backup completed - ' . $dbResult['filename']);
        } else {
            $results['success'] = false;
            $results['errors'][] = $dbResult['error'];
            logActivity('Backup: Database backup failed - ' . $dbResult['error']);
        }
        
        // File backup
        logActivity('Backup: Starting file backup');
        $fileResult = $this->fileBackup->backup();
        
        if ($fileResult['success']) {
            $results['backups']['files'] = $fileResult;
            logActivity('Backup: File backup completed - ' . $fileResult['filename']);
        } else {
            $results['success'] = false;
            $results['errors'][] = $fileResult['error'];
        }
        
        // Upload to remote storage
        if ($uploadToRemote && $this->remoteStorage && $results['success']) {
            logActivity('Backup: Uploading to remote storage');
            
            foreach ($results['backups'] as $type => $backup) {
                $uploadResult = $this->remoteStorage->upload($backup['filepath']);
                
                if ($uploadResult['success']) {
                    logActivity("Backup: Uploaded {$type} to S3");
                    $results['backups'][$type]['remote_path'] = $uploadResult['remote_path'];
                } else {
                    logActivity("Backup: Failed to upload {$type} to S3");
                }
            }
        }
        
        $results['completed_at'] = date('Y-m-d H:i:s');
        
        // Log results
        Capsule::table('mod_backup_log')->insert([
            'backup_date' => date('Y-m-d'),
            'results' => json_encode($results),
            'created_at' => date('Y-m-d H:i:s'),
        ]);
        
        return $results;
    }
    
    /**
     * Clean up old local backups
     */
    public function cleanupLocal(int $daysToKeep = 7): int
    {
        $cutoff = strtotime("-{$daysToKeep} days");
        $files = glob($this->dbBackup->getBackupDir() . '/*.sql*');
        
        $deleted = 0;
        foreach ($files as $file) {
            if (filemtime($file) < $cutoff) {
                unlink($file);
                $deleted++;
            }
        }
        
        return $deleted;
    }
}
```

## Scheduled Backups

```php
<?php
/**
 * Scheduled backup cron job
 */
function runScheduledBackups(): void
{
    $backupManager = new BackupManager([
        'database' => [
            'host' => Capsule::connection()->getConfig('host'),
            'database' => Capsule::connection()->getConfig('database'),
            'username' => Capsule::connection()->getConfig('username'),
            'password' => Capsule::connection()->getConfig('password'),
            'backup_dir' => '/backups',
        ],
        'whmcs_path' => ROOTDIR,
        'backup_dir' => '/backups',
        's3' => [
            'bucket' => S3_BUCKET,
            'region' => S3_REGION,
            'access_key' => S3_ACCESS_KEY,
            'secret_key' => S3_SECRET_KEY,
        ],
        's3_bucket' => S3_BUCKET,
    ]);
    
    $results = $backupManager->runBackup();
    
    // Keep only last 7 local backups
    $backupManager->cleanupLocal(7);
    
    // Email report if backup failed
    if (!$results['success']) {
        sendAdminNotification([
            'subject' => 'WHMCS Backup Failed',
            'message' => 'Backup failed: ' . implode(', ', $results['errors']),
        ]);
    }
}
```

## Best Practices

1. **Multiple backup types** - Database, files, and configurations
2. **Off-site storage** - Use cloud backup for redundancy
3. **Regular testing** - Verify backup restoration
4. **Encryption** - Encrypt sensitive backups
5. **Retention policy** - Keep appropriate number of backups
6. **Monitoring** - Alert on backup failures

## Related Documentation

- [whmcs-integration-cloud.md](whmcs-integration-cloud.md)
- [whmcs-integration-api.md](whmcs-integration-api.md)
