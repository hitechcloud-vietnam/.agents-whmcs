# WHMCS Disaster Recovery Workflow

## Purpose

Comprehensive guide to backup strategies, disaster recovery procedures, and business continuity planning for WHMCS installations.

## Prerequisites

- WHMCS installation
- Server access (SSH, control panel)
- Backup storage location
- Recovery testing environment
- Documentation of system architecture

## Workflow Steps

### Step 1: Backup Strategy Implementation

Set up comprehensive backup system:

```php
// includes/backup/backup_manager.php

class WHMCSBackupManager
{
    private $backupPath;
    private $retentionDays = 30;
    private $compressionLevel = 6;
    
    public function __construct(string $backupPath = null)
    {
        $this->backupPath = $backupPath ?? __DIR__ . '/../storage/backups';
        
        if (!is_dir($this->backupPath)) {
            mkdir($this->backupPath, 0750, true);
        }
    }
    
    /**
     * Create full backup
     */
    public function createFullBackup(): array
    {
        $timestamp = date('Y-m-d_His');
        $backupName = "whmcs_full_{$timestamp}";
        
        $results = [
            'started_at' => date('Y-m-d H:i:s'),
            'files' => false,
            'database' => false,
        ];
        
        // Backup files
        try {
            $this->backupFiles($backupName);
            $results['files'] = true;
            $results['files_size'] = filesize("{$this->backupPath}/{$backupName}_files.tar.gz");
        } catch (Exception $e) {
            $results['files_error'] = $e->getMessage();
        }
        
        // Backup database
        try {
            $this->backupDatabase($backupName);
            $results['database'] = true;
            $results['db_size'] = filesize("{$this->backupPath}/{$backupName}_db.sql.gz");
        } catch (Exception $e) {
            $results['database_error'] = $e->getMessage();
        }
        
        $results['completed_at'] = date('Y-m-d H:i:s');
        $results['backup_name'] = $backupName;
        
        // Log backup
        $this->logBackup($results);
        
        // Cleanup old backups
        $this->cleanupOldBackups();
        
        return $results;
    }
    
    /**
     * Backup WHMCS files
     */
    private function backupFiles(string $backupName): void
    {
        $whmcsRoot = ROOTDIR;
        $outputFile = "{$this->backupPath}/{$backupName}_files.tar.gz";
        
        // Exclude unnecessary directories
        $excludeDirs = [
            'storage/logs',
            'storage/cache',
            'storage/uploads',
            'templates_c',
            '.git',
            'node_modules',
        ];
        
        $excludeArgs = '';
        foreach ($excludeDirs as $dir) {
            $excludeArgs .= " --exclude='{$dir}'";
        }
        
        $command = "cd " . escapeshellarg($whmcsRoot) . " && " .
            "tar -czf " . escapeshellarg($outputFile) . " " .
            "--exclude='*.log' " .
            $excludeArgs .
            " .";
        
        exec($command, $output, $returnCode);
        
        if ($returnCode !== 0) {
            throw new Exception("File backup failed: " . implode("\n", $output));
        }
    }
    
    /**
     * Backup database
     */
    private function backupDatabase(string $backupName): void
    {
        $outputFile = "{$this->backupPath}/{$backupName}_db.sql.gz";
        
        $command = sprintf(
            'mysqldump -h %s -u %s -p%s %s | gzip > %s',
            escapeshellarg(get_config('mysql_host')),
            escapeshellarg(get_config('mysql_username')),
            escapeshellarg(get_config('mysql_password')),
            escapeshellarg(get_config('mysql_database')),
            escapeshellarg($outputFile)
        );
        
        exec($command, $output, $returnCode);
        
        if ($returnCode !== 0) {
            throw new Exception("Database backup failed");
        }
    }
    
    /**
     * Restore from backup
     */
    public function restoreBackup(string $backupName): array
    {
        $results = [
            'started_at' => date('Y-m-d H:i:s'),
            'files' => false,
            'database' => false,
        ];
        
        // Restore files
        try {
            $this->restoreFiles($backupName);
            $results['files'] = true;
        } catch (Exception $e) {
            $results['files_error'] = $e->getMessage();
        }
        
        // Restore database
        try {
            $this->restoreDatabase($backupName);
            $results['database'] = true;
        } catch (Exception $e) {
            $results['database_error'] = $e->getMessage();
        }
        
        $results['completed_at'] = date('Y-m-d H:i:s');
        
        return $results;
    }
    
    private function restoreFiles(string $backupName): void
    {
        $backupFile = "{$this->backupPath}/{$backupName}_files.tar.gz";
        
        if (!file_exists($backupFile)) {
            throw new Exception("Backup file not found: {$backupFile}");
        }
        
        $whmcsRoot = ROOTDIR;
        
        // Create temporary restoration directory
        $tempDir = "{$this->backupPath}/temp_restore";
        if (is_dir($tempDir)) {
            $this->rrmdir($tempDir);
        }
        mkdir($tempDir, 0750);
        
        // Extract to temp directory
        exec("tar -xzf " . escapeshellarg($backupFile) . " -C " . escapeshellarg($tempDir));
        
        // Copy files to WHMCS directory
        $this->rcopy($tempDir, $whmcsRoot);
        
        // Cleanup temp directory
        $this->rrmdir($tempDir);
    }
    
    private function restoreDatabase(string $backupName): void
    {
        $backupFile = "{$this->backupPath}/{$backupName}_db.sql.gz";
        
        if (!file_exists($backupFile)) {
            throw new Exception("Database backup not found: {$backupFile}");
        }
        
        $command = sprintf(
            'gunzip < %s | mysql -h %s -u %s -p%s %s',
            escapeshellarg($backupFile),
            escapeshellarg(get_config('mysql_host')),
            escapeshellarg(get_config('mysql_username')),
            escapeshellarg(get_config('mysql_password')),
            escapeshellarg(get_config('mysql_database'))
        );
        
        exec($command, $output, $returnCode);
        
        if ($returnCode !== 0) {
            throw new Exception("Database restore failed");
        }
    }
    
    /**
     * Upload backup to remote storage
     */
    public function uploadToRemote(string $backupName, string $provider): bool
    {
        switch ($provider) {
            case 's3':
                return $this->uploadToS3($backupName);
            case 'ftp':
                return $this->uploadToFtp($backupName);
            default:
                return false;
        }
    }
    
    private function uploadToS3(string $backupName): bool
    {
        $s3 = new S3Client([
            'version' => 'latest',
            'region' => S3_REGION,
            'credentials' => [
                'key' => S3_ACCESS_KEY,
                'secret' => S3_SECRET_KEY,
            ],
        ]);
        
        $files = [
            "{$this->backupPath}/{$backupName}_files.tar.gz",
            "{$this->backupPath}/{$backupName}_db.sql.gz",
        ];
        
        foreach ($files as $file) {
            if (file_exists($file)) {
                $s3->putObject([
                    'Bucket' => S3_BUCKET,
                    'Key' => basename($file),
                    'SourceFile' => $file,
                ]);
            }
        }
        
        return true;
    }
    
    private function logBackup(array $results): void
    {
        Capsule::table('mod_backup_history')->insert([
            'backup_name' => $results['backup_name'],
            'files_backup' => $results['files'] ? 1 : 0,
            'database_backup' => $results['database'] ? 1 : 0,
            'files_size' => $results['files_size'] ?? 0,
            'db_size' => $results['db_size'] ?? 0,
            'errors' => json_encode(array_filter([
                $results['files_error'] ?? null,
                $results['database_error'] ?? null,
            ])),
            'created_at' => date('Y-m-d H:i:s'),
        ]);
    }
    
    private function cleanupOldBackups(): void
    {
        $cutoff = date('Y-m-d H:i:s', strtotime("-{$this->retentionDays} days"));
        
        $oldBackups = Capsule::table('mod_backup_history')
            ->where('created_at', '<', $cutoff)
            ->get();
        
        foreach ($oldBackups as $backup) {
            $files = glob("{$this->backupPath}/{$backup->backup_name}_*");
            foreach ($files as $file) {
                @unlink($file);
            }
            
            Capsule::table('mod_backup_history')
                ->where('id', $backup->id)
                ->delete();
        }
    }
    
    private function rcopy(string $src, string $dst): void
    {
        $dir = opendir($src);
        @mkdir($dst);
        
        while (false !== ($file = readdir($dir))) {
            if ($file !== '.' && $file !== '..') {
                $srcFile = "{$src}/{$file}";
                $dstFile = "{$dst}/{$file}";
                
                if (is_dir($srcFile)) {
                    $this->rcopy($srcFile, $dstFile);
                } else {
                    copy($srcFile, $dstFile);
                }
            }
        }
        
        closedir($dir);
    }
    
    private function rrmdir(string $dir): void
    {
        if (!is_dir($dir)) return;
        
        $files = array_diff(scandir($dir), ['.', '..']);
        foreach ($files as $file) {
            $path = "{$dir}/{$file}";
            is_dir($path) ? $this->rrmdir($path) : unlink($path);
        }
        rmdir($dir);
    }
}
```

### Step 2: Automated Backup Schedule

Set up automated backup cron:

```php
// modules/addons/backup_scheduler/backup_scheduler.php

add_hook('DailyCronJob', 1, function($vars) {
    $backupManager = new WHMCSBackupManager();
    
    // Daily incremental backup
    $result = $backupManager->createFullBackup();
    
    if ($result['files'] && $result['database']) {
        logActivity("Daily backup completed: {$result['backup_name']}");
        
        // Upload to remote storage
        if (defined('REMOTE_BACKUP_ENABLED') && REMOTE_BACKUP_ENABLED) {
            $backupManager->uploadToRemote($result['backup_name'], 's3');
        }
    } else {
        // Send alert
        sendAlertEmail("Backup failed", json_encode($result));
    }
});

function backup_scheduler_config(): array
{
    return [
        'name' => 'Backup Scheduler',
        'description' => 'Automated WHMCS backup management',
        'version' => '1.0',
    ];
}

function backup_scheduler_activate(): array
{
    Capsule::schema()->create('mod_backup_history', function($table) {
        $table->increments('id');
        $table->string('backup_name', 100);
        $table->boolean('files_backup');
        $table->boolean('database_backup');
        $table->bigInteger('files_size');
        $table->bigInteger('db_size');
        $table->text('errors');
        $table->timestamp('created_at')->useCurrent();
    });
    
    return ['status' => 'success'];
}
```

### Step 3: Disaster Recovery Procedures

Document recovery procedures:

```php
/**
 * DR Scenario 1: Full System Failure
 * 
 * Recovery Time Objective: 4 hours
 * Recovery Point Objective: 24 hours
 */

// Step 1: Assess damage
function assessSystemDamage(): array
{
    return [
        'files_corrupted' => checkFilesCorruption(),
        'database_corrupted' => checkDatabaseIntegrity(),
        'configuration_lost' => !file_exists(ROOTDIR . '/includes/config.php'),
    ];
}

// Step 2: Restore from latest backup
function fullSystemRestore(): array
{
    $backupManager = new WHMCSBackupManager();
    
    // Find latest good backup
    $latestBackup = Capsule::table('mod_backup_history')
        ->where('files_backup', 1)
        ->where('database_backup', 1)
        ->orderBy('created_at', 'desc')
        ->first();
    
    if (!$latestBackup) {
        throw new Exception("No valid backup found");
    }
    
    // Restore
    return $backupManager->restoreBackup($latestBackup->backup_name);
}

/**
 * DR Scenario 2: Database Corruption
 */

// Step 1: Stop web server
function initiateDatabaseRecovery(): void
{
    exec('systemctl stop apache2'); // or nginx
}

// Step 2: Verify backup
function verifyDatabaseBackup(string $backupName): bool
{
    $gzHandle = gzopen("{$this->backupPath}/{$backupName}_db.sql.gz", 'r');
    $valid = false;
    
    // Read first few lines to verify structure
    for ($i = 0; $i < 10; $i++) {
        $line = gzgets($gzHandle);
        if (strpos($line, 'CREATE TABLE') !== false) {
            $valid = true;
            break;
        }
    }
    
    gzclose($gzHandle);
    return $valid;
}

// Step 3: Restore database
function restoreDatabaseOnly(string $backupName): void
{
    $backupManager = new WHMCSBackupManager();
    $backupManager->restoreBackup($backupName);
}

/**
 * DR Scenario 3: Ransomware Attack
 */

// Step 1: Isolate system
function isolateSystem(): void
{
    // Block all external access
    exec('iptables -I INPUT -j DROP');
    
    // Alert security team
    sendAlertEmail('Security Incident', 'System isolated due to suspected ransomware');
}

// Step 2: Identify scope
function identifyRansomwareScope(): array
{
    // Find encrypted files
    $encrypted = [];
    
    $files = new RecursiveIteratorIterator(
        new RecursiveDirectoryIterator(ROOTDIR)
    );
    
    foreach ($files as $file) {
        if ($file->isDir()) continue;
        
        // Check for encryption indicators
        if (detectEncryptedFile($file->getPathname())) {
            $encrypted[] = $file->getPathname();
        }
    }
    
    return [
        'files_encrypted' => count($encrypted),
        'encrypted_files' => $encrypted,
        'earliest_encryption' => findEarliestEncryption(),
    ];
}
```

### Step 4: Recovery Testing

Regular recovery testing procedures:

```php
/**
 * Monthly recovery test
 */
function performRecoveryTest(): array
{
    $results = [
        'test_date' => date('Y-m-d H:i:s'),
        'steps' => [],
    ];
    
    // Step 1: Create test environment
    $testEnv = createTestEnvironment();
    $results['steps']['environment'] = $testEnv ? 'success' : 'failed';
    
    if (!$testEnv) {
        return $results;
    }
    
    // Step 2: Download latest backup
    $backupDownloaded = downloadBackupToTestEnv();
    $results['steps']['download'] = $backupDownloaded ? 'success' : 'failed';
    
    // Step 3: Restore to test environment
    $restored = restoreToTestEnvironment();
    $results['steps']['restore'] = $restored ? 'success' : 'failed';
    
    // Step 4: Verify data integrity
    $integrity = verifyTestEnvironmentIntegrity();
    $results['steps']['integrity'] = $integrity ? 'success' : 'failed';
    
    // Step 5: Verify functionality
    $functionality = verifyBasicFunctionality();
    $results['steps']['functionality'] = $functionality ? 'success' : 'failed';
    
    // Cleanup
    cleanupTestEnvironment();
    
    // Log results
    Capsule::table('mod_recovery_tests')->insert($results);
    
    // Send report
    sendRecoveryTestReport($results);
    
    return $results;
}
```

## Best Practices

1. **Follow 3-2-1 rule** - 3 copies, 2 media types, 1 offsite
2. **Test backups regularly** - Monthly restoration tests
3. **Document everything** - Recovery procedures in writing
4. **Automate backups** - Remove human error
5. **Monitor backup status** - Alert on failures
6. **Encrypt backups** - Protect sensitive data
7. **Plan for scale** - Backup time grows with data
8. **Train staff** - Everyone should know procedures

## Common Pitfalls to Avoid

1. **Not testing backups** - May be corrupted or incomplete
2. **Single backup location** - Vulnerable to site issues
3. **No offsite backup** - Local disaster destroys all
4. **Incomplete backups** - Missing critical files
5. **Undocumented recovery** - Chaos during crisis
6. **Ignoring retention** - Keeping too many old backups
7. **No bandwidth planning** - Slow uploads cause failures
8. **Forgetting database** - Focus only on files
