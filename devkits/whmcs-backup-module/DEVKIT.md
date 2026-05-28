# WHMCS Backup Module DevKit
# Version: 1.0 | Updated: 2026-05-28

## DevKit Structure

```
devkits/whmcs-backup-module/
├── backup.php            # Backup controller
├── lib/
│   ├── BackupManager.php  # Backup management
│   ├── StorageAdapter.php # Storage provider adapters
│   ├── Scheduler.php      # Backup scheduling
│   └── Restorer.php       # Restore functionality
└── templates/
    ├── admin.tpl         # Admin templates
    └── client.tpl        # Client templates
```

## Backup Module Template

```php
<?php
/**
 * WHMCS Backup Module: {Module}
 * DevKit Template
 * 
 * Installation: Upload to modules/addons/{module}/
 */

if (!defined("WHMCS")) {
    die("Direct access denied");
}

use WHMCS\Database\Capsule;

/**
 * Config function
 */
function {module}_config(): array {
    return [
        'name' => '{Backup Module}',
        'description' => 'Automated backup and restore functionality',
        'version' => '1.0',
        'author' => '{Author}',
    ];
}

/**
 * Activate
 */
function {module}_activate(): array {
    Capsule::schema()->create('mod_{module}_backups', function($t) {
        $t->increments('id');
        $t->string('backup_type'); // full, database, files
        $t->string('storage_type'); // local, s3, ftp, sftp
        $t->string('filename');
        $t->string('path');
        $t->bigInteger('size');
        $t->string('status'); // pending, running, completed, failed
        $t->text('error_message')->nullable();
        $t->string('checksum');
        $t->integer('compressed');
        $t->integer('encrypted');
        $t->timestamp('started_at')->nullable();
        $t->timestamp('completed_at')->nullable();
        $t->timestamp('expires_at')->nullable();
        $t->timestamp('created_at');
    });
    
    Capsule::schema()->create('mod_{module}_backup_schedules', function($t) {
        $t->increments('id');
        $t->string('schedule_name');
        $t->string('backup_type');
        $t->string('frequency'); // hourly, daily, weekly, monthly
        $t->time('run_time');
        $t->string('retention_count');
        $t->string('storage_type');
        $t->text('storage_config');
        $t->boolean('is_active');
        $t->timestamp('last_run')->nullable();
        $t->timestamp('next_run');
        $t->timestamp('created_at');
    });
    
    Capsule::schema()->create('mod_{module}_backup_logs', function($t) {
        $t->increments('id');
        $t->integer('backup_id');
        $t->string('action');
        $t->text('message');
        $t->string('level'); // info, warning, error
        $t->timestamp('created_at');
    });
    
    Capsule::schema()->create('mod_{module}_restore_points', function($t) {
        $t->increments('id');
        $t->integer('backup_id');
        $t->string('restore_type');
        $t->string('status');
        $t->text('logs');
        $t->timestamp('started_at');
        $t->timestamp('completed_at')->nullable();
    });
    
    return ['status' => 'success', 'description' => 'Module activated'];
}

/**
 * Deactivate
 */
function {module}_deactivate(): array {
    Capsule::schema()->dropIfExists('mod_{module}_backups');
    Capsule::schema()->dropIfExists('mod_{module}_backup_schedules');
    Capsule::schema()->dropIfExists('mod_{module}_backup_logs');
    Capsule::schema()->dropIfExists('mod_{module}_restore_points');
    
    return ['status' => 'success'];
}

/**
 * Output function
 */
function {module}_output(array $vars): void {
    $action = $_REQUEST['action'] ?? 'dashboard';
    
    if ($_SERVER['REQUEST_METHOD'] === 'POST') {
        check_token('WHMCS.admin.default');
    }
    
    switch ($action) {
        case 'create':
            {module}_createBackup();
            break;
        case 'restore':
            {module}_restoreBackup();
            break;
        case 'schedules':
            {module}_manageSchedules();
            break;
        case 'download':
            {module}_downloadBackup();
            break;
        case 'delete':
            {module}_deleteBackup();
            break;
        case 'settings':
            {module}_showSettings();
            break;
        case 'logs':
            {module}_viewLogs();
            break;
        default:
            {module}_showDashboard();
    }
}

/**
 * Show Dashboard
 */
function {module}_showDashboard(): void {
    $stats = [
        'total_backups' => Capsule::table('mod_{module}_backups')->count(),
        'completed' => Capsule::table('mod_{module}_backups')
            ->where('status', 'completed')->count(),
        'failed' => Capsule::table('mod_{module}_backups')
            ->where('status', 'failed')->count(),
        'total_size' => Capsule::table('mod_{module}_backups')
            ->where('status', 'completed')->sum('size'),
        'active_schedules' => Capsule::table('mod_{module}_backup_schedules')
            ->where('is_active', 1)->count(),
    ];
    
    $latestBackups = Capsule::table('mod_{module}_backups')
        ->orderBy('created_at', 'desc')
        ->limit(5)
        ->get();
    
    echo <<<HTML
<div class="backup-module">
    <div class="row">
        <div class="col-md-12">
            <div class="panel panel-default">
                <div class="panel-heading">
                    <h3 class="panel-title">Backup Dashboard</h3>
                </div>
                <div class="panel-body">
                    <div class="row">
                        <div class="col-md-2">
                            <div class="stat-box">
                                <div class="stat-value">{$stats['total_backups']}</div>
                                <div class="stat-label">Total Backups</div>
                            </div>
                        </div>
                        <div class="col-md-2">
                            <div class="stat-box">
                                <div class="stat-value">{$stats['completed']}</div>
                                <div class="stat-label">Completed</div>
                            </div>
                        </div>
                        <div class="col-md-2">
                            <div class="stat-box">
                                <div class="stat-value">{$stats['failed']}</div>
                                <div class="stat-label">Failed</div>
                            </div>
                        </div>
                        <div class="col-md-3">
                            <div class="stat-box">
                                <div class="stat-value">{($stats['total_size'] / 1073741824)|number_format:2} GB</div>
                                <div class="stat-label">Total Size</div>
                            </div>
                        </div>
                        <div class="col-md-3">
                            <div class="stat-box">
                                <div class="stat-value">{$stats['active_schedules']}</div>
                                <div class="stat-label">Active Schedules</div>
                            </div>
                        </div>
                    </div>
                </div>
            </div>
        </div>
    </div>
    
    <div class="row">
        <div class="col-md-12">
            <div class="btn-group">
                <button type="button" class="btn btn-success" onclick="createBackup('full')">
                    <i class="fa fa-download"></i> Full Backup
                </button>
                <button type="button" class="btn btn-primary" onclick="createBackup('database')">
                    <i class="fa fa-database"></i> Database Backup
                </button>
                <button type="button" class="btn btn-info" onclick="createBackup('files')">
                    <i class="fa fa-file-archive"></i> Files Backup
                </button>
                <a href="?module={module}&action=schedules" class="btn btn-default">
                    <i class="fa fa-clock"></i> Schedules
                </a>
                <a href="?module={module}&action=logs" class="btn btn-default">
                    <i class="fa fa-file-alt"></i> Logs
                </a>
                <a href="?module={module}&action=settings" class="btn btn-default">
                    <i class="fa fa-cog"></i> Settings
                </a>
            </div>
        </div>
    </div>
    
    <div class="row" style="margin-top: 20px;">
        <div class="col-md-12">
            <div class="panel panel-default">
                <div class="panel-heading">
                    <h3 class="panel-title">Recent Backups</h3>
                </div>
                <div class="panel-body">
                    <table class="table table-striped">
                        <thead>
                            <tr>
                                <th>Date</th>
                                <th>Type</th>
                                <th>Size</th>
                                <th>Status</th>
                                <th>Actions</th>
                            </tr>
                        </thead>
                        <tbody>
HTML;
    
    foreach ($latestBackups as $backup) {
        $statusClass = match($backup->status) {
            'completed' => 'success',
            'failed' => 'danger',
            'running' => 'info',
            default => 'warning',
        };
        
        $size = $backup->size > 1073741824 
            ? number_format($backup->size / 1073741824, 2) . ' GB'
            : number_format($backup->size / 1048576, 2) . ' MB';
        
        echo "<tr>
            <td>{$backup->created_at}</td>
            <td>" . ucfirst($backup->backup_type) . "</td>
            <td>{$size}</td>
            <td><span class='label label-{$statusClass}'>{$backup->status}</span></td>
            <td>
                <a href='?module={module}&action=download&id={$backup->id}' class='btn btn-xs btn-default'>
                    <i class='fa fa-download'></i>
                </a>
                <a href='?module={module}&action=restore&id={$backup->id}' class='btn btn-xs btn-warning'>
                    <i class='fa fa-undo'></i>
                </a>
                <a href='?module={module}&action=delete&id={$backup->id}' class='btn btn-xs btn-danger' onclick='return confirm(\"Delete this backup?\")'>
                    <i class='fa fa-trash'></i>
                </a>
            </td>
        </tr>";
    }
    
    echo "</tbody></table></div></div></div></div>";
}
```

## Backup Manager Class

```php
<?php
namespace {Module};

use WHMCS\Database\Capsule;

class BackupManager {
    
    private string $backupPath;
    private array $config;
    
    public function __construct() {
        $this->backupPath = ROOTDIR . '/storage/backups/' . date('Y/m/d');
        $this->config = $this->loadConfig();
    }
    
    private function loadConfig(): array {
        $settings = Capsule::table('mod_{module}_backup_schedules')->get();
        $config = [];
        
        foreach ($settings as $setting) {
            $config[$setting->setting_name] = $setting->setting_value;
        }
        
        return $config;
    }
    
    public function createFullBackup(array $options = []): int {
        $backupId = $this->initBackup('full', $options);
        
        try {
            $this->updateBackupStatus($backupId, 'running');
            
            // Create backup directory
            if (!is_dir($this->backupPath)) {
                mkdir($this->backupPath, 0755, true);
            }
            
            $filename = 'full_backup_' . date('Y-m-d_His') . '.tar.gz';
            $filepath = $this->backupPath . '/' . $filename;
            
            // Backup database
            $dbBackup = $this->backupDatabase();
            $this->log($backupId, 'info', 'Database backup created');
            
            // Backup files
            $filesBackup = $this->backupFiles();
            $this->log($backupId, 'info', 'Files backup created');
            
            // Create archive
            $this->createArchive($filepath, $dbBackup, $filesBackup);
            
            // Calculate checksum
            $checksum = hash_file('sha256', $filepath);
            $size = filesize($filepath);
            
            // Update backup record
            Capsule::table('mod_{module}_backups')
                ->where('id', $backupId)
                ->update([
                    'filename' => $filename,
                    'path' => $filepath,
                    'size' => $size,
                    'checksum' => $checksum,
                    'status' => 'completed',
                    'completed_at' => date('Y-m-d H:i:s'),
                ]);
            
            // Cleanup temp files
            @unlink($dbBackup);
            $this->removeDirectory($filesBackup);
            
            $this->log($backupId, 'info', 'Full backup completed successfully');
            logActivity("{Module}: Full backup completed - {$filename}");
            
            return $backupId;
            
        } catch (\Exception $e) {
            $this->updateBackupStatus($backupId, 'failed', $e->getMessage());
            $this->log($backupId, 'error', 'Backup failed: ' . $e->getMessage());
            throw $e;
        }
    }
    
    public function createDatabaseBackup(array $options = []): int {
        $backupId = $this->initBackup('database', $options);
        
        try {
            $this->updateBackupStatus($backupId, 'running');
            
            if (!is_dir($this->backupPath)) {
                mkdir($this->backupPath, 0755, true);
            }
            
            $filename = 'database_backup_' . date('Y-m-d_His') . '.sql.gz';
            $filepath = $this->backupPath . '/' . $filename;
            
            $this->backupDatabase($filepath);
            
            $checksum = hash_file('sha256', $filepath);
            $size = filesize($filepath);
            
            Capsule::table('mod_{module}_backups')
                ->where('id', $backupId)
                ->update([
                    'filename' => $filename,
                    'path' => $filepath,
                    'size' => $size,
                    'checksum' => $checksum,
                    'status' => 'completed',
                    'completed_at' => date('Y-m-d H:i:s'),
                ]);
            
            $this->log($backupId, 'info', 'Database backup completed');
            logActivity("{Module}: Database backup completed - {$filename}");
            
            return $backupId;
            
        } catch (\Exception $e) {
            $this->updateBackupStatus($backupId, 'failed', $e->getMessage());
            throw $e;
        }
    }
    
    public function createFilesBackup(array $options = [], string $exclude = ''): int {
        $backupId = $this->initBackup('files', $options);
        
        try {
            $this->updateBackupStatus($backupId, 'running');
            
            if (!is_dir($this->backupPath)) {
                mkdir($this->backupPath, 0755, true);
            }
            
            $filename = 'files_backup_' . date('Y-m-d_His') . '.tar.gz';
            $filepath = $this->backupPath . '/' . $filename;
            
            $tempDir = $this->backupFiles($exclude);
            
            $this->createTarGz($filepath, $tempDir);
            
            $checksum = hash_file('sha256', $filepath);
            $size = filesize($filepath);
            
            Capsule::table('mod_{module}_backups')
                ->where('id', $backupId)
                ->update([
                    'filename' => $filename,
                    'path' => $filepath,
                    'size' => $size,
                    'checksum' => $checksum,
                    'status' => 'completed',
                    'completed_at' => date('Y-m-d H:i:s'),
                ]);
            
            $this->removeDirectory($tempDir);
            
            $this->log($backupId, 'info', 'Files backup completed');
            
            return $backupId;
            
        } catch (\Exception $e) {
            $this->updateBackupStatus($backupId, 'failed', $e->getMessage());
            throw $e;
        }
    }
    
    private function backupDatabase(string $outputPath = null): string {
        $outputPath = $outputPath ?? tempnam(sys_get_temp_dir(), 'db_') . '.sql';
        
        $dbHost = Capsule::connection()->getConfig()['host'] ?? 'localhost';
        $dbName = Capsule::connection()->getConfig()['database'] ?? '';
        $dbUser = Capsule::connection()->getConfig()['username'] ?? '';
        $dbPass = Capsule::connection()->getConfig()['password'] ?? '';
        
        $command = sprintf(
            'mysqldump --host=%s --user=%s --password=%s --single-transaction --routines --triggers %s | gzip > %s',
            escapeshellarg($dbHost),
            escapeshellarg($dbUser),
            escapeshellarg($dbPass),
            escapeshellarg($dbName),
            escapeshellarg($outputPath)
        );
        
        exec($command, $output, $returnCode);
        
        if ($returnCode !== 0) {
            throw new \Exception('Database backup failed');
        }
        
        return $outputPath;
    }
    
    private function backupFiles(string $exclude = ''): string {
        $tempDir = sys_get_temp_dir() . '/backup_files_' . uniqid();
        mkdir($tempDir, 0755, true);
        
        $sourceDir = ROOTDIR;
        $excludeDirs = ['storage/backups', 'storage/logs', 'vendor', 'node_modules', '.git'];
        
        if (!empty($exclude)) {
            $excludeDirs[] = $exclude;
        }
        
        $this->copyDirectory($sourceDir, $tempDir, $excludeDirs);
        
        return $tempDir;
    }
    
    private function createArchive(string $archivePath, string $dbBackup, string $filesDir): void {
        // Combine database backup into files backup directory
        copy($dbBackup, $filesDir . '/database.sql.gz');
        
        $this->createTarGz($archivePath, $filesDir);
    }
    
    private function createTarGz(string $archivePath, string $sourceDir): void {
        $archive = new \PharData($archivePath . '.tar');
        $archive->buildFromDirectory($sourceDir);
        $archive->compress(\Phar::GZ);
        
        rename($archivePath . '.tar.gz', $archivePath);
        @unlink($archivePath . '.tar');
    }
    
    private function copyDirectory(string $src, string $dst, array $exclude = []): void {
        $dir = opendir($src);
        
        while (($file = readdir($dir)) !== false) {
            if ($file === '.' || $file === '..') {
                continue;
            }
            
            $srcPath = $src . '/' . $file;
            $dstPath = $dst . '/' . $file;
            
            // Check exclusions
            $isExcluded = false;
            foreach ($exclude as $excluded) {
                if (strpos($srcPath, $excluded) !== false) {
                    $isExcluded = true;
                    break;
                }
            }
            
            if ($isExcluded) {
                continue;
            }
            
            if (is_dir($srcPath)) {
                mkdir($dstPath, 0755, true);
                $this->copyDirectory($srcPath, $dstPath, $exclude);
            } else {
                copy($srcPath, $dstPath);
            }
        }
        
        closedir($dir);
    }
    
    private function removeDirectory(string $dir): void {
        if (!is_dir($dir)) {
            return;
        }
        
        $files = array_diff(scandir($dir), ['.', '..']);
        
        foreach ($files as $file) {
            $path = $dir . '/' . $file;
            is_dir($path) ? $this->removeDirectory($path) : @unlink($path);
        }
        
        rmdir($dir);
    }
    
    private function initBackup(string $type, array $options = []): int {
        return Capsule::table('mod_{module}_backups')->insertGetId([
            'backup_type' => $type,
            'storage_type' => $options['storage_type'] ?? 'local',
            'status' => 'pending',
            'created_at' => date('Y-m-d H:i:s'),
        ]);
    }
    
    private function updateBackupStatus(int $backupId, string $status, string $error = null): void {
        $data = ['status' => $status];
        
        if ($status === 'running') {
            $data['started_at'] = date('Y-m-d H:i:s');
        }
        
        if ($status === 'failed' && $error) {
            $data['error_message'] = $error;
        }
        
        Capsule::table('mod_{module}_backups')
            ->where('id', $backupId)
            ->update($data);
    }
    
    private function log(int $backupId, string $level, string $message): void {
        Capsule::table('mod_{module}_backup_logs')->insert([
            'backup_id' => $backupId,
            'action' => 'backup',
            'message' => $message,
            'level' => $level,
            'created_at' => date('Y-m-d H:i:s'),
        ]);
    }
    
    public function uploadToStorage(int $backupId, string $storageType, array $config): bool {
        $backup = Capsule::table('mod_{module}_backups')
            ->where('id', $backupId)
            ->first();
        
        if (!$backup) {
            throw new \Exception('Backup not found');
        }
        
        $adapter = StorageAdapter::create($storageType, $config);
        $result = $adapter->upload($backup->path, basename($backup->path));
        
        $this->log($backupId, 'info', "Uploaded to {$storageType}: " . ($result ? 'success' : 'failed'));
        
        return $result;
    }
    
    public function cleanupOldBackups(int $retentionDays, string $backupType = null): int {
        $cutoffDate = date('Y-m-d H:i:s', strtotime("-{$retentionDays} days"));
        
        $query = Capsule::table('mod_{module}_backups')
            ->where('created_at', '<', $cutoffDate)
            ->where('status', 'completed');
        
        if ($backupType) {
            $query->where('backup_type', $backupType);
        }
        
        $oldBackups = $query->get();
        $deleted = 0;
        
        foreach ($oldBackups as $backup) {
            if (file_exists($backup->path)) {
                @unlink($backup->path);
            }
            
            Capsule::table('mod_{module}_backups')
                ->where('id', $backup->id)
                ->delete();
            
            $deleted++;
        }
        
        return $deleted;
    }
}
```

## Restorer Class

```php
<?php
namespace {Module};

use WHMCS\Database\Capsule;

class Restorer {
    
    private int $restoreId;
    
    public function restore(int $backupId, array $options = []): bool {
        $backup = Capsule::table('mod_{module}_backups')
            ->where('id', $backupId)
            ->first();
        
        if (!$backup) {
            throw new \Exception('Backup not found');
        }
        
        $this->restoreId = Capsule::table('mod_{module}_restore_points')->insertGetId([
            'backup_id' => $backupId,
            'restore_type' => $backup->backup_type,
            'status' => 'running',
            'logs' => json_encode([]),
            'started_at' => date('Y-m-d H:i:s'),
        ]);
        
        try {
            $this->log('Starting restore process');
            
            if (!file_exists($backup->path)) {
                throw new \Exception('Backup file not found');
            }
            
            // Verify checksum
            $currentChecksum = hash_file('sha256', $backup->path);
            if ($currentChecksum !== $backup->checksum) {
                throw new \Exception('Backup file checksum mismatch - file may be corrupted');
            }
            
            $tempDir = sys_get_temp_dir() . '/restore_' . uniqid();
            mkdir($tempDir, 0755, true);
            
            // Extract archive
            $this->log('Extracting backup archive');
            $this->extractArchive($backup->path, $tempDir);
            
            // Restore based on type
            if ($backup->backup_type === 'database' || $backup->backup_type === 'full') {
                $this->restoreDatabase($tempDir);
            }
            
            if ($backup->backup_type === 'files' || $backup->backup_type === 'full') {
                $this->restoreFiles($tempDir, $options['exclude'] ?? []);
            }
            
            // Cleanup
            $this->removeDirectory($tempDir);
            
            Capsule::table('mod_{module}_restore_points')
                ->where('id', $this->restoreId)
                ->update([
                    'status' => 'completed',
                    'completed_at' => date('Y-m-d H:i:s'),
                ]);
            
            $this->log('Restore completed successfully');
            logActivity("{Module}: Restore completed from backup #{$backupId}");
            
            return true;
            
        } catch (\Exception $e) {
            Capsule::table('mod_{module}_restore_points')
                ->where('id', $this->restoreId)
                ->update([
                    'status' => 'failed',
                    'logs' => json_encode(['error' => $e->getMessage()]),
                ]);
            
            $this->log('Restore failed: ' . $e->getMessage());
            throw $e;
        }
    }
    
    private function extractArchive(string $archivePath, string $destination): void {
        $phar = new \PharData($archivePath);
        $phar->extractTo($destination);
    }
    
    private function restoreDatabase(string $tempDir): void {
        $dbFile = $tempDir . '/database.sql.gz';
        
        if (!file_exists($dbFile)) {
            throw new \Exception('Database backup file not found in archive');
        }
        
        $dbHost = Capsule::connection()->getConfig()['host'] ?? 'localhost';
        $dbName = Capsule::connection()->getConfig()['database'] ?? '';
        $dbUser = Capsule::connection()->getConfig()['username'] ?? '';
        $dbPass = Capsule::connection()->getConfig()['password'] ?? '';
        
        $this->log('Restoring database');
        
        $command = sprintf(
            'gunzip < %s | mysql --host=%s --user=%s --password=%s %s',
            escapeshellarg($dbFile),
            escapeshellarg($dbHost),
            escapeshellarg($dbUser),
            escapeshellarg($dbPass),
            escapeshellarg($dbName)
        );
        
        exec($command, $output, $returnCode);
        
        if ($returnCode !== 0) {
            throw new \Exception('Database restore failed');
        }
        
        $this->log('Database restored successfully');
    }
    
    private function restoreFiles(string $tempDir, array $exclude = []): void {
        $this->log('Restoring files');
        
        $sourceFiles = $tempDir . '/';
        $destinationFiles = ROOTDIR . '/';
        
        // Copy files back (excluding certain directories)
        $this->copyDirectory($sourceFiles, $destinationFiles, $exclude);
        
        $this->log('Files restored successfully');
    }
    
    private function copyDirectory(string $src, string $dst, array $exclude = []): void {
        $dir = opendir($src);
        
        while (($file = readdir($dir)) !== false) {
            if ($file === '.' || $file === '..') {
                continue;
            }
            
            $srcPath = $src . '/' . $file;
            $dstPath = $dst . '/' . $file;
            
            $isExcluded = false;
            foreach ($exclude as $excluded) {
                if (strpos($srcPath, $excluded) !== false) {
                    $isExcluded = true;
                    break;
                }
            }
            
            if ($isExcluded) {
                continue;
            }
            
            if (is_dir($srcPath)) {
                if (!is_dir($dstPath)) {
                    mkdir($dstPath, 0755, true);
                }
                $this->copyDirectory($srcPath, $dstPath, $exclude);
            } else {
                copy($srcPath, $dstPath);
            }
        }
        
        closedir($dir);
    }
    
    private function removeDirectory(string $dir): void {
        if (!is_dir($dir)) {
            return;
        }
        
        $files = array_diff(scandir($dir), ['.', '..']);
        
        foreach ($files as $file) {
            $path = $dir . '/' . $file;
            is_dir($path) ? $this->removeDirectory($path) : @unlink($path);
        }
        
        rmdir($dir);
    }
    
    private function log(string $message): void {
        $logs = Capsule::table('mod_{module}_restore_points')
            ->where('id', $this->restoreId)
            ->value('logs');
        
        $logs = json_decode($logs, true) ?? [];
        $logs[] = [
            'time' => date('Y-m-d H:i:s'),
            'message' => $message,
        ];
        
        Capsule::table('mod_{module}_restore_points')
            ->where('id', $this->restoreId)
            ->update(['logs' => json_encode($logs)]);
    }
}
```

## Cron Hook

```php
<?php
/**
 * Backup Module Cron Hook
 */

if (!defined("WHMCS")) {
    die("Direct access denied");
}

add_hook('DailyCronJob', 1, function() {
    $schedules = Capsule::table('mod_{module}_backup_schedules')
        ->where('is_active', 1)
        ->where('next_run', '<=', date('Y-m-d H:i:s'))
        ->get();
    
    foreach ($schedules as $schedule) {
        try {
            $manager = new \{Module}\BackupManager();
            
            switch ($schedule->backup_type) {
                case 'full':
                    $manager->createFullBackup();
                    break;
                case 'database':
                    $manager->createDatabaseBackup();
                    break;
                case 'files':
                    $manager->createFilesBackup();
                    break;
            }
            
            // Cleanup old backups based on retention
            $manager->cleanupOldBackups($schedule->retention_count * 7);
            
            // Update next run time
            Capsule::table('mod_{module}_backup_schedules')
                ->where('id', $schedule->id)
                ->update([
                    'last_run' => date('Y-m-d H:i:s'),
                    'next_run' => calculateNextRun($schedule->frequency, $schedule->run_time),
                ]);
            
            logActivity("{Module}: Scheduled backup '{$schedule->schedule_name}' completed");
            
        } catch (\Exception $e) {
            logActivity("{Module}: Scheduled backup '{$schedule->schedule_name}' failed: " . $e->getMessage());
        }
    }
});

function calculateNextRun(string $frequency, string $runTime): string {
    $time = strtotime($runTime);
    
    switch ($frequency) {
        case 'hourly':
            return date('Y-m-d H:i:s', strtotime('+1 hour', $time));
        case 'daily':
            return date('Y-m-d', strtotime('+1 day')) . ' ' . $runTime;
        case 'weekly':
            return date('Y-m-d', strtotime('+1 week')) . ' ' . $runTime;
        case 'monthly':
            return date('Y-m-d', strtotime('+1 month')) . ' ' . $runTime;
        default:
            return date('Y-m-d', strtotime('+1 day')) . ' ' . $runTime;
    }
}
```

## Checklist

```
Pre-Dev:
□ Define backup types (full, database, files)
□ Plan storage options (local, S3, FTP, SFTP)
□ Design retention policy
□ Plan scheduling mechanism
□ Identify files to exclude

Development:
□ Create backup tables
□ Implement BackupManager class
□ Implement Restorer class
□ Implement Scheduler class
□ Create storage adapters (S3, FTP, SFTP)
□ Add database backup (mysqldump)
□ Add files backup (tar/gzip)
□ Create archive management
□ Implement checksum verification
□ Add backup scheduling
□ Create restore functionality
□ Build admin interface
□ Add download functionality
□ Implement retention cleanup

Testing:
□ Test full backup creation
□ Test database backup
□ Test files backup
□ Verify archive integrity
□ Verify checksum validation
□ Test restore functionality
□ Test with large files
□ Verify exclusion patterns
□ Test storage uploads
□ Test scheduled backups
□ Test retention cleanup
```