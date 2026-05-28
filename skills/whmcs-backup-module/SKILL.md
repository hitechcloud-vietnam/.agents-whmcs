# WHMCS Backup Module Skill
# Version: 1.0 | Updated: 2026-05-28

## Purpose

Guide for building backup and restore modules for WHMCS.

## When to Use

- Creating automated backup modules
- Building restore functionality
- Managing backup rotation

## Backup Module Pattern

```php
<?php
// modules/addons/{backupmodule}/{backupmodule}.php
if (!defined("WHMCS")) { die("Direct access denied"); }

function {backupmodule}_config(): array {
    return [
        'name' => 'Backup Manager',
        'description' => 'Automated backup and restore system',
        'version' => '1.0',
        'author' => 'Author',
        'backup_path' => ['FriendlyName' => 'Backup Directory', 'Type' => 'text', 'Default' => '/backup'],
        'retention_days' => ['FriendlyName' => 'Retention Days', 'Type' => 'text', 'Default' => '30'],
        'compression' => ['FriendlyName' => 'Compression', 'Type' => 'yesno'],
    ];
}

function {backupmodule}_activate(): array {
    Capsule::schema()->create('mod_backup_backups', function($t) {
        $t->increments('id');
        $t->string('name', 255);
        $t->string('type', 50);
        $t->string('file_path', 500);
        $t->bigInteger('file_size')->unsigned();
        $t->string('status', 50);
        $t->timestamp('started_at');
        $t->timestamp('completed_at')->nullable();
        $t->text('error_log')->nullable();
    });

    Capsule::schema()->create('mod_backup_schedules', function($t) {
        $t->increments('id');
        $t->string('name', 100);
        $t->string('type', 50);
        $t->string('schedule', 100);
        $t->boolean('enabled')->default(true);
        $t->timestamp('last_run')->nullable();
        $t->timestamp('next_run')->nullable();
        $t->json('options')->nullable();
        $t->timestamps();
    });

    return ['status' => 'success', 'description' => 'Backup module activated'];
}

function {backupmodule}_deactivate(): array {
    Capsule::schema()->dropIfExists('mod_backup_backups');
    Capsule::schema()->dropIfExists('mod_backup_schedules');
    return ['status' => 'success', 'description' => 'Backup module deactivated'];
}

function {backupmodule}_output(array $vars): void {
    $action = $_REQUEST['action'] ?? 'dashboard';
    include __DIR__ . '/templates/admin/' . $action . '.tpl';
}

function {backupmodule}_cron(): void {
    $schedules = Capsule::table('mod_backup_schedules')
        ->where('enabled', 1)
        ->where('next_run', '<=', date('Y-m-d H:i:s'))
        ->get();

    foreach ($schedules as $schedule) {
        executeBackup($schedule);
    }
}

function executeBackup($schedule): void {
    $backupId = Capsule::table('mod_backup_backups')->insertGetId([
        'name' => $schedule->name . '_' . date('Y-m-d_His'),
        'type' => $schedule->type,
        'status' => 'running',
        'started_at' => date('Y-m-d H:i:s'),
    ]);

    try {
        $options = json_decode($schedule->options, true) ?? [];
        $backupPath = Capsule::table('mod_configuration')
            ->where('setting', 'backup_path')
            ->value('value') ?? '/backup';

        $result = match ($schedule->type) {
            'full' => performFullBackup($backupPath, $options),
            'database' => performDatabaseBackup($backupPath, $options),
            'files' => performFilesBackup($backupPath, $options),
            'configs' => performConfigBackup($backupPath, $options),
            default => throw new \Exception('Unknown backup type'),
        };

        Capsule::table('mod_backup_backups')
            ->where('id', $backupId)
            ->update([
                'file_path' => $result['path'],
                'file_size' => $result['size'],
                'status' => 'completed',
                'completed_at' => date('Y-m-d H:i:s'),
            ]);

        // Update next run
        $nextRun = calculateNextRun($schedule->schedule);
        Capsule::table('mod_backup_schedules')
            ->where('id', $schedule->id)
            ->update([
                'last_run' => date('Y-m-d H:i:s'),
                'next_run' => $nextRun,
            ]);

        // cleanup old backups
        cleanupOldBackups();

    } catch (\Exception $e) {
        Capsule::table('mod_backup_backups')
            ->where('id', $backupId)
            ->update([
                'status' => 'failed',
                'error_log' => $e->getMessage(),
                'completed_at' => date('Y-m-d H:i:s'),
            ]);
    }
}

function performFullBackup(string $basePath, array $options): array {
    // Backup database
    $dbBackup = performDatabaseBackup($basePath, $options);

    // Backup files
    $filesBackup = performFilesBackup($basePath, $options);

    // Create combined archive
    $filename = 'full_backup_' . date('Ymd_His') . '.tar.gz';
    $path = $basePath . '/full/' . $filename;

    // Combine into single archive
    $combined = [];
    $combined['path'] = $path;
    $combined['size'] = $dbBackup['size'] + $filesBackup['size'];

    return $combined;
}

function performDatabaseBackup(string $basePath, array $options): array {
    $filename = 'db_backup_' . date('Ymd_His') . '.sql';
    $path = $basePath . '/database/' . $filename;

    // Use mysqldump if available
    $command = sprintf(
        'mysqldump -h%s -u%s -p%s %s > %s',
        Config::get('mysql_host'),
        Config::get('mysql_username'),
        Config::get('mysql_password'),
        Config::get('mysql_database'),
        $path
    );

    exec($command);

    return [
        'path' => $path,
        'size' => filesize($path),
    ];
}

function performFilesBackup(string $basePath, array $options): array {
    $filename = 'files_backup_' . date('Ymd_His') . '.tar.gz';
    $path = $basePath . '/files/' . $filename;

    $exclude = $options['exclude'] ?? ['vendor', 'node_modules', '.git'];

    $excludeArgs = '';
    foreach ($exclude as $dir) {
        $excludeArgs .= " --exclude='$dir'";
    }

    $command = sprintf(
        'tar -czf %s%s %s',
        $path,
        $excludeArgs,
        WHMCS_ROOT
    );

    exec($command);

    return [
        'path' => $path,
        'size' => filesize($path),
    ];
}

function performConfigBackup(string $basePath, array $options): array {
    $filename = 'configs_backup_' . date('Ymd_His') . '.zip';
    $path = $basePath . '/configs/' . $filename;

    $configFiles = [
        WHMCS_ROOT . '/configuration.php',
        WHMCS_ROOT . '/.env',
    ];

    $zip = new \ZipArchive();
    $zip->open($path, \ZipArchive::CREATE);

    foreach ($configFiles as $file) {
        if (file_exists($file)) {
            $zip->addFile($file, basename($file));
        }
    }

    $zip->close();

    return [
        'path' => $path,
        'size' => filesize($path),
    ];
}

function calculateNextRun(string $schedule): string {
    $dt = new \DateTime();

    switch ($schedule) {
        case 'daily':
            $dt->modify('+1 day');
            break;
        case 'weekly':
            $dt->modify('+1 week');
            break;
        case 'monthly':
            $dt->modify('+1 month');
            break;
    }

    return $dt->format('Y-m-d H:i:s');
}

function cleanupOldBackups(): void {
    $retentionDays = (int) Capsule::table('mod_configuration')
        ->where('setting', 'retention_days')
        ->value('value') ?? 30;

    $cutoffDate = date('Y-m-d H:i:s', strtotime("-$retentionDays days"));

    $oldBackups = Capsule::table('mod_backup_backups')
        ->where('completed_at', '<', $cutoffDate)
        ->where('status', 'completed')
        ->get();

    foreach ($oldBackups as $backup) {
        if (file_exists($backup->file_path)) {
            unlink($backup->file_path);
        }
        Capsule::table('mod_backup_backups')
            ->where('id', $backup->id)
            ->delete();
    }
}

function {backupmodule}_restore(int $backupId): bool {
    $backup = Capsule::table('mod_backup_backups')
        ->where('id', $backupId)
        ->first();

    if (!$backup || $backup->status !== 'completed') {
        return false;
    }

    try {
        match ($backup->type) {
            'database' => restoreDatabase($backup->file_path),
            'files' => restoreFiles($backup->file_path),
            'full' => restoreFullBackup($backup->file_path),
            default => throw new \Exception('Unknown backup type'),
        };

        Capsule::table('mod_backup_backups')
            ->where('id', $backupId)
            ->update([
                'status' => 'restored',
                'completed_at' => date('Y-m-d H:i:s'),
            ]);

        return true;

    } catch (\Exception $e) {
        logActivity('Restore failed: ' . $e->getMessage());
        return false;
    }
}
```

### Scheduled Cron Registration

```php
// hooks.php or in activate()

add_hook('DailyCronJob', 1, function() {
    // Run backup module's scheduled tasks
    include_once __DIR__ . '/modules/addons/{backupmodule}/{backupmodule}.php';
    {backupmodule}_cron();
});
```

---

**Related Skills:**
- whmcs-cron-automation
- whmcs-admin-ui-builder
- whmcs-deployment
