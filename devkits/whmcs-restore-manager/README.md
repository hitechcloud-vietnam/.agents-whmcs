# WHMCS Restore Manager Module

Data restore management with point-in-time recovery, selective restore, and verification.

## Features

- Backup verification
- Selective restore (full, database, table, file)
- Step-by-step restore execution
- Automatic pre-restore backup
- Rollback capability
- Point-in-time snapshots
- Progress tracking
- Detailed logging

## Installation

1. Copy `restoremanager.php` to `/path/to/whmcs/modules/addons/restoremanager/`
2. Activate in WHMCS Admin > System Settings > Module Addons
3. Configure backup path

## Usage

```php
// Register a backup
restoremanager_RegisterBackup(array(
    'backup_name' => 'Daily Backup 2026-05-28',
    'backup_path' => '/backups/daily_20260528.tar.gz',
    'backup_type' => 'full',
    'backup_date' => '2026-05-28 02:00:00',
    'size_bytes' => filesize('/backups/daily_20260528.tar.gz'),
    'checksum' => hash_file('sha256', '/backups/daily_20260528.tar.gz'),
    'compressed' => true,
    'tables_count' => 45,
    'files_count' => 1200
));

// Get backups
$backups = restoremanager_GetBackups();
$backups = restoremanager_GetBackups(array(
    'type' => 'full',
    'from_date' => '2026-05-01',
    'to_date' => '2026-05-28',
    'verified' => true
));

// Get backup info
$backup = restoremanager_GetBackup($backupId);

// Verify backup
$result = restoremanager_VerifyBackup($backupId);
// Returns: success, passed, checks (file_exists, checksum_valid, etc.)

// Create restore job (full database restore)
$result = restoremanager_CreateRestore(array(
    'backup_id' => $backupId,
    'restore_type' => 'database',
    'scope' => array('database'),
    'initiated_by' => $adminId,
    'reason' => 'Data corruption recovery'
));

// Selective restore (specific tables)
$result = restoremanager_CreateRestore(array(
    'backup_id' => $backupId,
    'restore_type' => 'selective',
    'scope' => array('table:tblclients', 'table:tblhosting'),
    'initiated_by' => $adminId,
    'reason' => 'Client data recovery'
));

// Restore with files
$result = restoremanager_CreateRestore(array(
    'backup_id' => $backupId,
    'restore_type' => 'selective',
    'scope' => array('database', 'file:/attachments/client_123')
));

// Start restore execution
$result = restoremanager_StartRestore($restoreId);
// Returns: success, restore_id, rows_restored

// Get restore status
$restore = restoremanager_GetRestore($restoreId);
// Returns: status, progress, steps_completed, rows_restored, etc.

// List restores
$restores = restoremanager_GetRestores();
$restores = restoremanager_GetRestores(array('status' => 'failed'));
$restores = restoremanager_GetRestores(array('backup_id' => $backupId));

// Get restore steps
$steps = restoremanager_GetRestoreSteps($restoreId);
// Returns detailed step information

// Cancel pending restore
restoremanager_CancelRestore($restoreId);

// Create point-in-time snapshot
restoremanager_CreateSnapshot(array(
    'snapshot_name' => 'Pre-upgrade snapshot',
    'snapshot_key' => 'pre_upgrade_2026_05',
    'backup_id' => $backupId,
    'point_in_time' => '2026-05-28 00:00:00',
    'retention_days' => 30,
    'created_by' => $adminId
));

// Get snapshots
$snapshots = restoremanager_GetSnapshots();
```

## Configuration Options

| Option | Type | Default | Description |
|--------|------|---------|-------------|
| BackupPath | text | /storage/backups | Backup directory |
| EnableVerification | yesno | yes | Verify backups |
| EnablePreRestore | yesno | yes | Pre-restore backup |
| MaxConcurrentRestores | text | 1 | Concurrent restores |
| RestoreTimeout | text | 3600 | Timeout (seconds) |

## Restore Types

| Type | Description |
|------|-------------|
| full | Complete restore |
| database | Database only |
| table | Single table |
| file | Single file |
| selective | Multiple items |

## Restore Scope

```php
// Full database
'scope' => array('database')

// Specific tables
'scope' => array('table:tblclients', 'table:tblhosting')

// Configuration file
'scope' => array('config')

// Specific files/directories
'scope' => array('file:/attachments', 'file:/templates')

// Combined
'scope' => array('database', 'config', 'file:/attachments')
```

## Verification Checks

| Check | Description |
|-------|-------------|
| file_exists | File exists |
| file_readable | File is readable |
| checksum_valid | SHA256 checksum matches |
| can_decompress | Compression intact |

## Restore Status

| Status | Description |
|--------|-------------|
| pending | Waiting to start |
| running | In progress |
| completed | Successfully completed |
| failed | Failed with error |
| cancelled | Cancelled by user |

## Restore Steps

Each restore is broken into executable steps:

1. Verify backup
2. Extract archive
3. Restore database
4. Restore configuration
5. Restore files
6. Verify restore

## Pre-Restore Backup

When enabled, automatically creates a database backup before restore:

```php
$result = restoremanager_CreateRestore(array(
    'backup_id' => $backupId,
    'restore_type' => 'database',
    'scope' => array('database')
));
// Automatically creates pre-restore backup
```

## Database Tables

- `mod_restoremanager_backups` - Backup registry
- `mod_restoremanager_restores` - Restore jobs
- `mod_restoremanager_restore_steps` - Restore steps
- `mod_restoremanager_verification` - Verification results
- `mod_restoremanager_snapshots` - Point-in-time snapshots
