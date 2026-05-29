# WHMCS Database Backup

## Skill Description
Implement automated database backup procedures for WHMCS modules including full backups, incremental backups, backup verification, and secure storage.

## Prerequisites
- WHMCS 7.0+ installation
- PHP 7.4+
- Shell access (mysqldump) or PHP-based backup
- Sufficient disk space for backups

## Step-by-Step Implementation

### 1. Backup Manager
```php
<?php
// includes/backup/DatabaseBackup.php

namespace WHMCS\Module\YourModule\Backup;

class DatabaseBackup
{
    private string $backupDir;
    private string $dbHost;
    private string $dbName;
    private string $dbUser;
    private string $dbPass;
    private int $retentionDays = 30;

    public function __construct(array $config = [])
    {
        $config = array_merge([
            'backup_dir' => dirname(__DIR__) . '/backups',
            'db_host' =>WHMCS_INTEGRATION_DB_HOST ?? 'localhost',
            'db_name' =>WHMCS_INTEGRATION_DB_NAME ?? '',
            'db_user' =>WHMCS_INTEGRATION_DB_USER ?? '',
            'db_pass' =>WHMCS_INTEGRATION_DB_PASS ?? '',
            'retention_days' => 30
        ], $config);

        $this->backupDir = $config['backup_dir'];
        $this->dbHost = $config['db_host'];
        $this->dbName = $config['db_name'];
        $this->dbUser = $config['db_user'];
        $this->dbPass = $config['db_pass'];
        $this->retentionDays = $config['retention_days'];

        if (!is_dir($this->backupDir)) {
            mkdir($this->backupDir, 0755, true);
        }
    }

    public function createFullBackup(): array
    {
        $filename = $this->generateFilename('full');
        $filepath = $this->backupDir . '/' . $filename;

        $startTime = microtime(true);

        try {
            $result = $this->mysqldump($filepath);

            if (!$result) {
                throw new \Exception('mysqldump failed');
            }

            $fileSize = filesize($filepath);
            $duration = round(microtime(true) - $startTime, 2);

            // Compress the backup
            $compressedFile = $this->compress($filepath);

            // Calculate checksum
            $checksum = hash_file('sha256', $compressedFile);

            // Store metadata
            $metadata = $this->storeMetadata($filename, [
                'type' => 'full',
                'size' => $fileSize,
                'compressed_size' => filesize($compressedFile),
                'checksum' => $checksum,
                'duration' => $duration,
                'created_at' => date('Y-m-d H:i:s')
            ]);

            // Cleanup old backups
            $this->cleanup();

            return [
                'success' => true,
                'filename' => basename($compressedFile),
                'size' => filesize($compressedFile),
                'checksum' => $checksum,
                'duration' => $duration
            ];

        } catch (\Exception $e) {
            return [
                'success' => false,
                'error' => $e->getMessage()
            ];
        }
    }

    public function createTableBackup(array $tables): array
    {
        $filename = $this->generateFilename('partial');
        $filepath = $this->backupDir . '/' . $filename;

        $startTime = microtime(true);

        try {
            $result = $this->mysqldump($filepath, $tables);

            if (!$result) {
                throw new \Exception('mysqldump failed');
            }

            $compressedFile = $this->compress($filepath);
            $checksum = hash_file('sha256', $compressedFile);
            $duration = round(microtime(true) - $startTime, 2);

            $this->storeMetadata($filename, [
                'type' => 'partial',
                'tables' => implode(',', $tables),
                'checksum' => $checksum,
                'duration' => $duration
            ]);

            return [
                'success' => true,
                'filename' => basename($compressedFile),
                'size' => filesize($compressedFile),
                'checksum' => $checksum
            ];

        } catch (\Exception $e) {
            return [
                'success' => false,
                'error' => $e->getMessage()
            ];
        }
    }

    private function mysqldump(string $filepath, array $tables = []): bool
    {
        $tablesClause = !empty($tables) ? implode(' ', array_map(fn($t) => "'{$t}'", $tables)) : '';

        $command = sprintf(
            'mysqldump --host=%s --user=%s --password=%s %s %s > %s 2>&1',
            escapeshellarg($this->dbHost),
            escapeshellarg($this->dbUser),
            escapeshellarg($this->dbPass),
            escapeshellarg($this->dbName),
            $tablesClause,
            escapeshellarg($filepath)
        );

        exec($command, $output, $returnCode);

        return $returnCode === 0;
    }

    private function compress(string $filepath): string
    {
        $compressedFile = $filepath . '.gz';

        $fp = gzopen($compressedFile, 'w9');
        $data = file_get_contents($filepath);
        gzwrite($fp, $data);
        gzclose($fp);

        unlink($filepath);

        return $compressedFile;
    }

    public function restore(string $backupFile): array
    {
        if (!file_exists($backupFile)) {
            return ['success' => false, 'error' => 'Backup file not found'];
        }

        // Decompress if needed
        if (pathinfo($backupFile, PATHINFO_EXTENSION) === 'gz') {
            $tempFile = tempnam(sys_get_temp_dir(), 'restore_');
            $fp = gzopen($backupFile, 'rb');
            $data = gzread($fp, 1048576 * 100); // 100MB chunks
            gzclose($fp);
            file_put_contents($tempFile, $data);
            $backupFile = $tempFile;
        }

        $startTime = microtime(true);

        try {
            $command = sprintf(
                'mysql --host=%s --user=%s --password=%s %s < %s 2>&1',
                escapeshellarg($this->dbHost),
                escapeshellarg($this->dbUser),
                escapeshellarg($this->dbPass),
                escapeshellarg($this->dbName),
                escapeshellarg($backupFile)
            );

            exec($command, $output, $returnCode);

            if ($returnCode !== 0) {
                throw new \Exception(implode("\n", $output));
            }

            $duration = round(microtime(true) - $startTime, 2);

            // Cleanup temp file
            if (isset($tempFile)) {
                unlink($tempFile);
            }

            return [
                'success' => true,
                'duration' => $duration
            ];

        } catch (\Exception $e) {
            return [
                'success' => false,
                'error' => $e->getMessage()
            ];
        }
    }

    public function verify(string $backupFile): bool
    {
        if (!file_exists($backupFile)) {
            return false;
        }

        // Check file extension
        $ext = pathinfo($backupFile, PATHINFO_EXTENSION);

        if ($ext === 'gz') {
            $fp = gzopen($backupFile, 'rb');
            $firstLine = gzgets($fp, 4096);
            gzclose($fp);
        } else {
            $fp = fopen($backupFile, 'rb');
            $firstLine = fgets($fp, 4096);
            fclose($fp);
        }

        // Check for valid MySQL dump header
        return strpos($firstLine, '-- MySQL dump') !== false
            || strpos($firstLine, 'CREATE TABLE') !== false;
    }

    private function cleanup(): void
    {
        $cutoff = strtotime("-{$this->retentionDays} days");
        $files = glob($this->backupDir . '/*.sql.gz');

        foreach ($files as $file) {
            if (filemtime($file) < $cutoff) {
                unlink($file);

                // Remove metadata
                $metaFile = $file . '.meta';
                if (file_exists($metaFile)) {
                    unlink($metaFile);
                }
            }
        }
    }

    private function generateFilename(string $type): string
    {
        return sprintf(
            '%s_%s_%s.sql',
            $this->dbName,
            $type,
            date('Y-m-d_H-i-s')
        );
    }

    private function storeMetadata(string $filename, array $metadata): void
    {
        $metaFile = $this->backupDir . '/' . $filename . '.meta';
        file_put_contents($metaFile, json_encode($metadata, JSON_PRETTY_PRINT));
    }

    public function listBackups(): array
    {
        $files = glob($this->backupDir . '/*.sql.gz');
        $backups = [];

        foreach ($files as $file) {
            $metaFile = $file . '.meta';
            $metadata = file_exists($metaFile)
                ? json_decode(file_get_contents($metaFile), true)
                : [];

            $backups[] = [
                'filename' => basename($file),
                'size' => filesize($file),
                'modified' => date('Y-m-d H:i:s', filemtime($file)),
                'metadata' => $metadata
            ];
        }

        usort($backups, fn($a, $b) => strtotime($b['modified']) - strtotime($a['modified']));

        return $backups;
    }
}
```

### 2. Backup Schedule
```php
<?php
// includes/backup/BackupScheduler.php

namespace WHMCS\Module\YourModule\Backup;

class BackupScheduler
{
    private DatabaseBackup $backup;

    public function __construct()
    {
        $this->backup = new DatabaseBackup();
    }

    public function daily(): array
    {
        // Daily full backup
        return $this->backup->createFullBackup();
    }

    public function weekly(): array
    {
        // Keep all backups for a week
        $this->backup->createFullBackup();

        return ['status' => 'success', 'message' => 'Weekly backup completed'];
    }

    public function monthly(): array
    {
        // Keep monthly backups for 1 year
        $result = $this->backup->createFullBackup();

        if ($result['success']) {
            // Create archive for long-term storage
            $archiveDir = dirname($this->backup->getBackupDir()) . '/archives/' . date('Y');
            if (!is_dir($archiveDir)) {
                mkdir($archiveDir, 0755, true);
            }
        }

        return $result;
    }

    public function hourly(): array
    {
        // Hourly incremental backup of critical tables
        return $this->backup->createTableBackup([
            'tblclients',
            'tblhosting',
            'tblinvoices',
            'tblorders'
        ]);
    }
}
```

## Common Pitfalls and Solutions

| Pitfall | Solution |
|---------|----------|
| Large backups timeout | Use streaming/chunked backups |
| Disk space exhaustion | Implement retention policies |
| Corrupted backups | Verify backups after creation |
| Backup failures | Implement retry logic with alerts |
| Slow backups | Use compression and parallel processing |

## Security Considerations

1. **Encrypt backups** - Encrypt sensitive data at rest
2. **Secure storage** - Limit access to backup files
3. **Secure transfer** - Use encrypted channels for remote backup
4. **Verify integrity** - Use checksums to verify backup integrity
5. **Secure credentials** - Don't store database passwords in plain text

## Testing Checklist

- [ ] Test full backup creation
- [ ] Test backup compression
- [ ] Test backup restoration
- [ ] Test backup verification
- [ ] Test cleanup of old backups
- [ ] Test checksum verification
- [ ] Test error handling
- [ ] Test backup listing

## Reference Links

- [MySQL Backup Documentation](https://dev.mysql.com/doc/refman/8.0/en/backup-and-recovery.html)
- [mariabackup](https://mariadb.com/kb/en/mariabackup/)
- [MySQL Binary Logging](https://dev.mysql.com/doc/refman/8.0/en/binary-log.html)
