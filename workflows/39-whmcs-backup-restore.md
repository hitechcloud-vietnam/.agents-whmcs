# WHMCS Backup & Restore Workflow

## Overview
This workflow establishes comprehensive backup and restore procedures for WHMCS.

## Step 1: Backup Service

```php
<?php
// src/Service/BackupService.php

namespace WHMCS\Module\Addon\YourModule\Service;

use WHMCS\Database\Capsule;

class BackupService
{
    private $backupPath;
    private $retentionDays = 30;

    public function __construct(string $backupPath = null)
    {
        $this->backupPath = $backupPath ?? dirname(__DIR__, 3) . '/storage/backups';
        if (!is_dir($this->backupPath)) {
            mkdir($this->backupPath, 0755, true);
        }
    }

    public function createFullBackup(bool $compress = true): array
    {
        $timestamp = date('Y-m-d-His');
        $backupName = "whmcs_full_{$timestamp}";
        $backupDir = $this->backupPath . '/' . $backupName;

        mkdir($backupDir, 0755, true);

        // Backup database
        $this->backupDatabase($backupDir);

        // Backup files
        $this->backupFiles($backupDir);

        // Create manifest
        $this->createManifest($backupDir);

        // Compress if requested
        if ($compress) {
            $this->compressBackup($backupDir);
        }

        // Clean old backups
        $this->cleanOldBackups();

        return [
            'success' => true,
            'backup_name' => $backupName,
            'backup_path' => $compress ? "$backupDir.tar.gz" : $backupDir,
            'created_at' => date('Y-m-d H:i:s')
        ];
    }

    private function backupDatabase(string $backupDir): void
    {
        $dbHost = Capsule::config('db_host');
        $dbName = Capsule::config('db_name');
        $dbUser = Capsule::config('db_username');
        $dbPass = Capsule::config('db_password');

        $sqlFile = "$backupDir/database.sql";

        $command = sprintf(
            'mysqldump -h %s -u %s -p%s %s > %s 2>/dev/null',
            escapeshellarg($dbHost),
            escapeshellarg($dbUser),
            escapeshellarg($dbPass),
            escapeshellarg($dbName),
            escapeshellarg($sqlFile)
        );

        exec($command);

        // Gzip the SQL file
        $sqlGz = "$sqlFile.gz";
        file_put_contents($sqlGz, gzencode(file_get_contents($sqlFile)));
        unlink($sqlFile);
    }

    private function backupFiles(string $backupDir): void
    {
        $sourceDir = dirname(__DIR__, 3);
        $excludeDirs = ['storage/backups', 'storage/cache', 'storage/logs', 'vendor'];

        $exclude = '';
        foreach ($excludeDirs as $dir) {
            $exclude .= " --exclude='$dir'";
        }

        $filesBackup = "$backupDir/files.tar.gz";
        $command = "cd " . escapeshellarg($sourceDir) . " && tar -czf " . escapeshellarg($filesBackup) . " .$exclude 2>/dev/null";

        exec($command);
    }

    private function createManifest(string $backupDir): void
    {
        $manifest = [
            'created_at' => date('Y-m-d H:i:s'),
            'whmcs_version' => Capsule::config('version'),
            'php_version' => PHP_VERSION,
            'database_name' => Capsule::config('db_name'),
            'hostname' => gethostname()
        ];

        file_put_contents(
            "$backupDir/manifest.json",
            json_encode($manifest, JSON_PRETTY_PRINT)
        );
    }

    private function compressBackup(string $backupDir): void
    {
        $tarFile = "$backupDir.tar.gz";
        $command = "tar -czf " . escapeshellarg($tarFile) . " -C " . escapeshellarg($this->backupPath) . " " . basename($backupDir);

        exec($command);

        // Remove uncompressed directory
        $this->deleteDirectory($backupDir);
    }

    private function cleanOldBackups(): void
    {
        $cutoff = strtotime("-{$this->retentionDays} days");
        $backups = glob($this->backupPath . '/whmcs_full_*.tar.gz');

        foreach ($backups as $backup) {
            if (filemtime($backup) < $cutoff) {
                unlink($backup);
            }
        }
    }

    public function restoreBackup(string $backupFile): array
    {
        if (!file_exists($backupFile)) {
            return ['success' => false, 'error' => 'Backup file not found'];
        }

        $tempDir = $this->backupPath . '/restore_' . time();
        mkdir($tempDir, 0755, true);

        // Extract backup
        $command = "tar -xzf " . escapeshellarg($backupFile) . " -C " . escapeshellarg($tempDir);
        exec($command);

        // Check manifest
        $manifestFile = $tempDir . '/manifest.json';
        if (!file_exists($manifestFile)) {
            return ['success' => false, 'error' => 'Invalid backup: manifest not found'];
        }

        // Restore database
        $this->restoreDatabase($tempDir);

        // Restore files
        $this->restoreFiles($tempDir);

        // Clean up
        $this->deleteDirectory($tempDir);

        return [
            'success' => true,
            'restored_at' => date('Y-m-d H:i:s')
        ];
    }

    private function restoreDatabase(string $backupDir): void
    {
        $dbHost = Capsule::config('db_host');
        $dbName = Capsule::config('db_name');
        $dbUser = Capsule::config('db_username');
        $dbPass = Capsule::config('db_password');

        $sqlGz = "$backupDir/database.sql.gz";
        $sqlContent = gzdecode(file_get_contents($sqlGz));
        $sqlFile = "$backupDir/database.sql";
        file_put_contents($sqlFile, $sqlContent);

        $command = sprintf(
            'mysql -h %s -u %s -p%s %s < %s 2>/dev/null',
            escapeshellarg($dbHost),
            escapeshellarg($dbUser),
            escapeshellarg($dbPass),
            escapeshellarg($dbName),
            escapeshellarg($sqlFile)
        );

        exec($command);
        unlink($sqlFile);
    }

    private function restoreFiles(string $backupDir): void
    {
        $targetDir = dirname(__DIR__, 3);
        $filesTar = "$backupDir/files.tar.gz";

        $command = "tar -xzf " . escapeshellarg($filesTar) . " -C " . escapeshellarg($targetDir);
        exec($command);
    }

    private function deleteDirectory(string $dir): void
    {
        if (!is_dir($dir)) return;

        $files = array_diff(scandir($dir), ['.', '..']);
        foreach ($files as $file) {
            $path = "$dir/$file";
            is_dir($path) ? $this->deleteDirectory($path) : unlink($path);
        }
        rmdir($dir);
    }

    public function listBackups(): array
    {
        $backups = glob($this->backupPath . '/whmcs_full_*.tar.gz');

        return array_map(function($backup) {
            return [
                'name' => basename($backup),
                'path' => $backup,
                'size' => filesize($backup),
                'created' => date('Y-m-d H:i:s', filemtime($backup))
            ];
        }, $backups);
    }
}
```

## Step 2: Scheduled Backup Cron

```php
<?php
// includes/cron/backup_cron.php

require_once __DIR__ . '/../../init.php';

use WHMCS\Module\Addon\YourModule\Service\BackupService;

$backupService = new BackupService();

// Daily full backup
$result = $backupService->createFullBackup(true);

if ($result['success']) {
    logActivity("Automated backup created: " . $result['backup_name']);

    // Upload to remote storage if configured
    if (get_config('backup_remote_storage')) {
        uploadToRemoteStorage($result['backup_path']);
    }
}
```

## Verification Checklist

- [ ] Backup service implemented
- [ ] Database backup working
- [ ] File backup working
- [ ] Compression working
- [ ] Backup listing working
- [ ] Restore procedure tested
- [ ] Retention cleanup working
- [ ] Cron job configured
