---
name: whmcs-backup-scheduling
description: Backup scheduling for WHMCS services
category: Provisioning & Cloud
version: 1.0.0
---

# WHMCS Backup Scheduling Skill

## Overview
This skill provides patterns and implementations for managing backup scheduling in WHMCS, including automated backup configuration, retention policies, backup verification, and restore workflows.

## Implementation Patterns

### Backup Scheduler Class
```php
<?php
/**
 * WHMCS Backup Scheduling System
 * Manages automated backups for hosted services
 */

namespace WHMCS\Module\Server\Backup;

class BackupScheduler {
    private $db;
    private $storageAdapters = [];
    private $backupQueue = [];

    public function __construct() {
        $this->db = \WHMCS\Database\Capsule::connection();
        $this->initializeStorageAdapters();
    }

    private function initializeStorageAdapters(): void {
        $this->storageAdapters = [
            'local' => new LocalBackupStorage(),
            's3' => new S3BackupStorage(),
            'nfs' => new NFSBackupStorage(),
            'sftp' => new SFTPBackupStorage()
        ];
    }

    /**
     * Create backup schedule for a service
     */
    public function createSchedule(array $params): array {
        $this->validateScheduleParams($params);

        $serviceId = $params['service_id'];
        $scheduleType = $params['schedule_type']; // hourly, daily, weekly, monthly
        $retentionCount = $params['retention_count'] ?? 7;
        $backupType = $params['backup_type'] ?? 'incremental';
        $compressionEnabled = $params['compression'] ?? true;

        // Calculate cron expression based on schedule type
        $cronExpression = $this->calculateCronExpression($scheduleType, $params);

        // Create schedule record
        $scheduleId = 'sched_' . bin2hex(random_bytes(12));

        $this->db->insert('mod_backup_schedules', [
            'id' => $scheduleId,
            'service_id' => $serviceId,
            'cron_expression' => $cronExpression,
            'backup_type' => $backupType,
            'compression_enabled' => $compressionEnabled,
            'retention_count' => $retentionCount,
            'storage_backend' => $params['storage_backend'] ?? 'local',
            'enabled' => $params['enabled'] ?? true,
            'next_run' => $this->calculateNextRun($cronExpression),
            'created_at' => date('Y-m-d H:i:s')
        ]);

        // Create initial backup job
        $this->queueBackupJob($scheduleId, $serviceId);

        return [
            'success' => true,
            'schedule_id' => $scheduleId,
            'cron_expression' => $cronExpression,
            'next_run' => $this->calculateNextRun($cronExpression)
        ];
    }

    /**
     * Execute scheduled backup
     */
    public function executeBackup(string $scheduleId): array {
        $schedule = $this->getSchedule($scheduleId);

        if (!$schedule) {
            throw new \Exception("Schedule not found: {$scheduleId}");
        }

        // Get VM details
        $vmDetails = $this->getVmDetails($schedule['service_id']);
        $hypervisor = $vmDetails['hypervisor_type'];

        // Create backup based on hypervisor
        $backupId = $this->createBackup($schedule, $vmDetails);

        // Upload to storage
        $this->uploadBackup($backupId, $schedule['storage_backend']);

        // Verify backup integrity
        $verificationResult = $this->verifyBackup($backupId);

        // Update schedule next run
        $this->updateScheduleNextRun($scheduleId);

        // Prune old backups
        $this->pruneOldBackups($schedule);

        return [
            'success' => true,
            'backup_id' => $backupId,
            'size_mb' => $verificationResult['size_mb'],
            'checksum' => $verificationResult['checksum'],
            'verified' => $verificationResult['valid']
        ];
    }

    /**
     * List backups for a service
     */
    public function listBackups(int $serviceId, array $filters = []): array {
        $query = "SELECT * FROM mod_service_backups WHERE service_id = ?";
        $bindings = [$serviceId];

        if (!empty($filters['status'])) {
            $query .= " AND status = ?";
            $bindings[] = $filters['status'];
        }

        if (!empty($filters['from_date'])) {
            $query .= " AND created_at >= ?";
            $bindings[] = $filters['from_date'];
        }

        if (!empty($filters['to_date'])) {
            $query .= " AND created_at <= ?";
            $bindings[] = $filters['to_date'];
        }

        $query .= " ORDER BY created_at DESC";

        if (!empty($filters['limit'])) {
            $query .= " LIMIT ?";
            $bindings[] = (int) $filters['limit'];
        }

        $backups = $this->db->select($query, $bindings);

        return array_map(function($backup) {
            return [
                'id' => $backup->id,
                'service_id' => $backup->service_id,
                'schedule_id' => $backup->schedule_id,
                'size_mb' => round($backup->size_mb, 2),
                'checksum' => $backup->checksum,
                'status' => $backup->status,
                'storage_path' => $backup->storage_path,
                'created_at' => $backup->created_at,
                'verified_at' => $backup->verified_at
            ];
        }, $backups);
    }

    /**
     * Restore from backup
     */
    public function restoreBackup(string $backupId, array $options = []): array {
        $backup = $this->getBackup($backupId);

        if (!$backup) {
            throw new \Exception("Backup not found: {$backupId}");
        }

        // Verify backup exists and is valid
        if ($backup['status'] !== 'completed') {
            throw new \Exception("Backup is not in completed state: {$backup['status']}");
        }

        $vmDetails = $this->getVmDetails($backup['service_id']);
        $hypervisor = $vmDetails['hypervisor_type'];

        // Download backup if remote storage
        $localPath = $this->downloadBackupIfNeeded($backup);

        // Create pre-restore snapshot
        $preRestoreSnapshot = $this->createPreRestoreSnapshot($backup['service_id']);

        // Stop VM
        $this->stopVM($vmDetails);

        // Restore disk
        $this->restoreVMDisk($vmDetails, $localPath, $options);

        // Start VM
        $this->startVM($vmDetails);

        // Update backup record
        $this->db->update('mod_service_backups', [
            'restored_at' => date('Y-m-d H:i:s')
        ], ['id' => $backupId]);

        return [
            'success' => true,
            'backup_id' => $backupId,
            'restored_at' => date('Y-m-d H:i:s'),
            'pre_restore_snapshot' => $preRestoreSnapshot
        ];
    }

    /**
     * Get backup schedules due for execution
     */
    public function getDueSchedules(): array {
        $now = date('Y-m-d H:i:s');

        $schedules = $this->db->select(
            "SELECT * FROM mod_backup_schedules
             WHERE enabled = 1 AND next_run <= ?
             ORDER BY next_run ASC",
            [$now]
        );

        return $schedules;
    }

    /**
     * Calculate backup retention and cleanup
     */
    public function pruneOldBackups(array $schedule): int {
        $retentionCount = $schedule['retention_count'];

        $backups = $this->db->select(
            "SELECT id, storage_path FROM mod_service_backups
             WHERE schedule_id = ? AND status = 'completed'
             ORDER BY created_at DESC",
            [$schedule['id']]
        );

        $deleted = 0;
        foreach ($backups as $index => $backup) {
            if ($index >= $retentionCount) {
                // Delete backup file
                $this->deleteBackupFile($backup->storage_path);

                // Delete database record
                $this->db->delete('mod_service_backups', ['id' => $backup->id]);

                $deleted++;
            }
        }

        return $deleted;
    }

    // Private helper methods

    private function createBackup(array $schedule, array $vmDetails): string {
        $backupId = 'bkp_' . bin2hex(random_bytes(12));
        $timestamp = date('Y-m-d_His');

        $vmId = $vmDetails['vm_id'];
        $hypervisor = $vmDetails['hypervisor_type'];

        // Create backup based on hypervisor type
        switch ($hypervisor) {
            case 'proxmox':
                $backupPath = $this->createProxmoxBackup($vmId, $backupId);
                break;
            case 'vmware':
                $backupPath = $this->createVMwareBackup($vmId, $backupId);
                break;
            case 'libvirt':
                $backupPath = $this->createLibvirtBackup($vmId, $backupId);
                break;
            default:
                throw new \Exception("Unsupported hypervisor: {$hypervisor}");
        }

        // Store backup metadata
        $this->db->insert('mod_service_backups', [
            'id' => $backupId,
            'service_id' => $schedule['service_id'],
            'schedule_id' => $schedule['id'],
            'backup_type' => $schedule['backup_type'],
            'storage_path' => $backupPath,
            'size_mb' => filesize($backupPath) / (1024 * 1024),
            'checksum' => hash_file('sha256', $backupPath),
            'status' => 'completed',
            'created_at' => date('Y-m-d H:i:s')
        ]);

        return $backupId;
    }

    private function createProxmoxBackup(string $vmId, string $backupId): string {
        $node = $this->getNodeForVM($vmId);
        $storage = $this->getBackupStorage();

        $command = "vzdump {$vmId} --storage {$storage} --mode snapshot --compress zstd";
        exec($command, $output, $return);

        if ($return !== 0) {
            throw new \Exception("Proxmox backup failed: " . implode("\n", $output));
        }

        // Find the created backup file
        $backupFiles = glob("/var/lib/vz/dump/*-{$vmId}-*.vma.zst");
        return $backupFiles[0] ?? '';
    }

    private function createVMwareBackup(string $vmId, string $backupId): string {
        $datastore = $this->getVM Datastore($vmId);
        $backupDir = "/backups/vmware/{$backupId}";

        $command = "govc snapshot.create -vm {$vmId} backup-{$backupId}";
        exec($command);

        $command = "govc vm.clone -vm {$vmId} -snapshot backup-{$backupId} -folder {$backupDir}";
        exec($command);

        return $backupDir;
    }

    private function uploadBackup(string $backupId, string $storageBackend): void {
        if (!isset($this->storageAdapters[$storageBackend])) {
            throw new \Exception("Unknown storage backend: {$storageBackend}");
        }

        $adapter = $this->storageAdapters[$storageBackend];
        $backup = $this->getBackup($backupId);

        $adapter->upload($backupId, $backup['storage_path']);
    }

    private function verifyBackup(string $backupId): array {
        $backup = $this->getBackup($backupId);

        if (!file_exists($backup['storage_path'])) {
            throw new \Exception("Backup file not found: {$backup['storage_path']}");
        }

        $calculatedChecksum = hash_file('sha256', $backup['storage_path']);
        $storedChecksum = $backup['checksum'];

        return [
            'valid' => $calculatedChecksum === $storedChecksum,
            'size_mb' => filesize($backup['storage_path']) / (1024 * 1024),
            'checksum' => $calculatedChecksum,
            'verified_at' => date('Y-m-d H:i:s')
        ];
    }

    private function calculateCronExpression(string $scheduleType, array $params): string {
        switch ($scheduleType) {
            case 'hourly':
                return '0 * * * *';
            case 'daily':
                return '0 ' . ($params['hour'] ?? 2) . ' * * *';
            case 'weekly':
                return '0 ' . ($params['hour'] ?? 2) . ' * * ' . ($params['day_of_week'] ?? 0);
            case 'monthly':
                return '0 ' . ($params['hour'] ?? 2) . ' 1 * *';
            default:
                throw new \Exception("Unknown schedule type: {$scheduleType}");
        }
    }

    private function calculateNextRun(string $cronExpression): string {
        // Simple next run calculation
        // Production should use a proper cron library
        return date('Y-m-d H:i:s', strtotime('+1 hour'));
    }

    private function updateScheduleNextRun(string $scheduleId): void {
        $schedule = $this->getSchedule($scheduleId);
        $nextRun = $this->calculateNextRun($schedule['cron_expression']);

        $this->db->update('mod_backup_schedules', [
            'next_run' => $nextRun,
            'last_run' => date('Y-m-d H:i:s')
        ], ['id' => $scheduleId]);
    }

    private function queueBackupJob(string $scheduleId, int $serviceId): void {
        $this->backupQueue[] = [
            'schedule_id' => $scheduleId,
            'service_id' => $serviceId,
            'queued_at' => date('Y-m-d H:i:s')
        ];
    }
}

/**
 * Backup Storage Adapter Interface
 */
interface BackupStorageInterface {
    public function upload(string $backupId, string $localPath): void;
    public function download(string $backupId, string $localPath): void;
    public function delete(string $backupId): void;
    public function list(string $prefix = ''): array;
    public function getUrl(string $backupId): string;
}

class S3BackupStorage implements BackupStorageInterface {
    private $config;

    public function configure(array $config): void {
        $this->config = $config;
    }

    public function upload(string $backupId, string $localPath): void {
        $s3 = new \Aws\S3\S3Client([
            'region' => $this->config['region'],
            'version' => 'latest'
        ]);

        $s3->putObject([
            'Bucket' => $this->config['bucket'],
            'Key' => "backups/{$backupId}.tar.gz",
            'SourceFile' => $localPath
        ]);
    }

    public function download(string $backupId, string $localPath): void {
        $s3 = new \Aws\S3\S3Client([
            'region' => $this->config['region'],
            'version' => 'latest'
        ]);

        $result = $s3->getObject([
            'Bucket' => $this->config['bucket'],
            'Key' => "backups/{$backupId}.tar.gz",
            'SaveAs' => $localPath
        ]);
    }

    public function delete(string $backupId): void {
        $s3 = new \Aws\S3\S3Client([
            'region' => $this->config['region'],
            'version' => 'latest'
        ]);

        $s3->deleteObject([
            'Bucket' => $this->config['bucket'],
            'Key' => "backups/{$backupId}.tar.gz"
        ]);
    }

    public function list(string $prefix = ''): array {
        $s3 = new \Aws\S3\S3Client([
            'region' => $this->config['region'],
            'version' => 'latest'
        ]);

        $result = $s3->listObjects([
            'Bucket' => $this->config['bucket'],
            'Prefix' => "backups/{$prefix}"
        ]);

        return array_map(function($obj) {
            return $obj['Key'];
        }, $result['Contents']);
    }

    public function getUrl(string $backupId): string {
        return "https://{$this->config['bucket']}.s3.amazonaws.com/backups/{$backupId}.tar.gz";
    }
}
```

## Database Schema
```sql
CREATE TABLE `mod_backup_schedules` (
  `id` VARCHAR(50) PRIMARY KEY,
  `service_id` INT NOT NULL,
  `cron_expression` VARCHAR(100) NOT NULL,
  `backup_type` ENUM('full', 'incremental', 'differential') DEFAULT 'incremental',
  `compression_enabled` TINYINT(1) DEFAULT 1,
  `encryption_enabled` TINYINT(1) DEFAULT 0,
  `retention_count` INT DEFAULT 7,
  `storage_backend` VARCHAR(50) DEFAULT 'local',
  `enabled` TINYINT(1) DEFAULT 1,
  `next_run` DATETIME,
  `last_run` DATETIME,
  `created_at` DATETIME NOT NULL,
  INDEX `idx_service_id` (`service_id`),
  INDEX `idx_next_run` (`next_run`)
);

CREATE TABLE `mod_service_backups` (
  `id` VARCHAR(50) PRIMARY KEY,
  `service_id` INT NOT NULL,
  `schedule_id` VARCHAR(50) NOT NULL,
  `backup_type` VARCHAR(20) NOT NULL,
  `storage_path` VARCHAR(500) NOT NULL,
  `size_mb` BIGINT DEFAULT 0,
  `checksum` VARCHAR(64) NOT NULL,
  `status` ENUM('pending', 'in_progress', 'completed', 'failed', 'verified') DEFAULT 'pending',
  `created_at` DATETIME NOT NULL,
  `verified_at` DATETIME,
  `restored_at` DATETIME,
  INDEX `idx_service_id` (`service_id`),
  INDEX `idx_status` (`status`)
);
```

## Cron Job Setup
```bash
#!/bin/bash
# /etc/cron.d/whmcs-backup-scheduler
# Run backup scheduler every 5 minutes

*/5 * * * * root /usr/bin/php /path/to/whmcs/crons/backup_scheduler.php
```

## Best Practices

1. **Incremental Backups**: Use incremental backups between full backups
2. **Encryption**: Encrypt sensitive data at rest and in transit
3. **Verification**: Always verify backup integrity after creation
4. **Off-site Storage**: Store critical backups in remote locations
5. **Testing**: Regularly test backup restoration procedures

## Related Skills

- whmcs-snapshot-management
- whmcs-disaster-recovery-vm
- whmcs-storage-provisioning
- whmcs-monitoring-agent