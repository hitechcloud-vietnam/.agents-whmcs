# WHMCS Backup & Restore Skill
# Version: 1.0 | Updated: 2026-05-28

## Purpose

Guide for implementing backup and restore functionality in WHMCS modules.

## When to Use

- Creating module data backups
- Implementing restore functionality
- Scheduled backup automation

## Backup Patterns

### Module Data Backup

```php
function backupModuleData(string $backupDir): string {
    $timestamp = date('Y-m-d_His');
    $backupFile = $backupDir . '/backup_' . $timestamp . '.zip';

    $zip = new ZipArchive();
    $zip->open($backupFile, ZipArchive::CREATE);

    // Backup tables
    $tables = ['settings', 'logs', 'data'];

    foreach ($tables as $table) {
        $tableName = 'mod_{module}_' . $table;
        $data = Capsule::table($tableName)->get()->toArray();
        $zip->addFromString($table . '.json', json_encode($data));
    }

    // Backup config
    $config = getModuleConfig();
    $zip->addFromString('config.json', json_encode($config));

    $zip->close();

    return $backupFile;
}

function getModuleConfig(): array {
    return Capsule::table('mod_{module}_settings')
        ->pluck('setting_value', 'setting_key')
        ->toArray();
}
```

### Restore from Backup

```php
function restoreFromBackup(string $backupFile): bool {
    $zip = new ZipArchive();
    $zip->open($backupFile);

    // Validate backup
    if (!$zip->locateName('config.json')) {
        throw new \Exception('Invalid backup file');
    }

    // Clear existing data
    clearModuleData();

    // Restore tables
    foreach (['settings', 'logs', 'data'] as $table) {
        $json = $zip->getFromName($table . '.json');
        if ($json) {
            $records = json_decode($json, true);
            foreach ($records as $record) {
                Capsule::table('mod_{module}_' . $table)->insert($record);
            }
        }
    }

    $zip->close();

    logActivity('Backup restored from: ' . $backupFile);
    return true;
}

function clearModuleData(): void {
    foreach (['settings', 'logs', 'data'] as $table) {
        Capsule::table('mod_{module}_' . $table)->truncate();
    }
}
```

### Scheduled Backup

```php
add_hook('DailyCronJob', 1, function($vars) {
    $backupDir = '/backups/modules/{module}';

    if (!is_dir($backupDir)) {
        mkdir($backupDir, 0755, true);
    }

    $backupFile = backupModuleData($backupDir);

    // Keep only last 7 backups
    cleanupOldBackups($backupDir, 7);

    // Notify admin
    notifyBackupComplete($backupFile);
});
```

## Checklist

- [ ] Backup function
- [ ] Restore function
- [ ] Validation
- [ ] Scheduled backups
- [ ] Cleanup old backups

---

**Related Skills:**
- whmcs-database-design
- whmcs-cron-automation
- whmcs-deployment