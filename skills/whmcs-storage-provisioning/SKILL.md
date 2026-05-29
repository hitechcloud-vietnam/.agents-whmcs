---
name: whmcs-storage-provisioning
description: Storage allocation for WHMCS services
category: Provisioning & Cloud
version: 1.0.0
---

# WHMCS Storage Provisioning Skill

## Overview
This skill provides patterns and implementations for managing storage provisioning in WHMCS, including block storage, object storage, quota management, and storage tier optimization.

## Implementation Patterns

### Storage Manager Class
```php
<?php
/**
 * WHMCS Storage Provisioning
 * Manages storage allocation for hosted services
 */

namespace WHMCS\Module\Server\Storage;

class StorageManager {
    private $db;
    private $providers = [];

    public function __construct() {
        $this->db = \WHMCS\Database\Capsule::connection();
        $this->initializeProviders();
    }

    private function initializeProviders(): void {
        $this->providers = [
            'local' => new LocalStorageProvider(),
            'ceph' => new CephStorageProvider(),
            'nfs' => new NFSStorageProvider(),
            'iscsi' => new ISCSIStorageProvider(),
            'aws_ebs' => new AWSEBSProvider(),
            'gcp_persistent' => new GCP PersistentDiskProvider()
        ];
    }

    /**
     * Create storage allocation
     */
    public function createStorage(array $params): array {
        $this->validateStorageParams($params);

        $storageId = 'stor_' . bin2hex(random_bytes(12));
        $volumeName = 'vol_' . bin2hex(random_bytes(8));

        $storage = [
            'id' => $storageId,
            'service_id' => $params['service_id'],
            'volume_name' => $volumeName,
            'size_gb' => $params['size_gb'],
            'type' => $params['type'] ?? 'ssd', // ssd, hdd, nvme
            'tier' => $params['tier'] ?? 'standard', // standard, premium, cold
            'provider' => $params['provider'] ?? 'local',
            'mount_point' => $params['mount_point'] ?? '/data',
            'filesystem' => $params['filesystem'] ?? 'ext4',
            'created_at' => date('Y-m-d H:i:s')
        ];

        $this->db->insert('mod_storage_allocations', $storage);

        // Provision storage via provider
        $provider = $this->getProvider($params['provider'] ?? 'local');
        $providerResult = $provider->create($volumeName, $params['size_gb'], $params);

        // Attach to VM
        $this->attachToVM($storageId, $params['service_id']);

        // Format and mount
        $this->formatAndMount($storageId);

        return [
            'success' => true,
            'storage_id' => $storageId,
            'volume_name' => $volumeName,
            'size_gb' => $params['size_gb'],
            'mount_point' => $storage['mount_point']
        ];
    }

    /**
     * Expand storage volume
     */
    public function expandStorage(string $storageId, int $newSizeGB): array {
        $storage = $this->getStorage($storageId);

        if (!$storage) {
            throw new \Exception("Storage not found: {$storageId}");
        }

        if ($newSizeGB <= $storage['size_gb']) {
            throw new \Exception("New size must be greater than current size");
        }

        $provider = $this->getProvider($storage['provider']);

        // Expand volume on provider
        $provider->expand($storage['volume_name'], $newSizeGB);

        // Resize filesystem
        $this->resizeFilesystem($storageId, $newSizeGB);

        // Update database
        $this->db->update('mod_storage_allocations', [
            'size_gb' => $newSizeGB,
            'expanded_at' => date('Y-m-d H:i:s')
        ], ['id' => $storageId]);

        // Record billing adjustment
        $this->recordStorageUpgradeBilling($storageId, $storage['size_gb'], $newSizeGB);

        return [
            'success' => true,
            'storage_id' => $storageId,
            'old_size_gb' => $storage['size_gb'],
            'new_size_gb' => $newSizeGB
        ];
    }

    /**
     * Create snapshot of storage
     */
    public function createSnapshot(string $storageId, array $params = []): array {
        $storage = $this->getStorage($storageId);

        if (!$storage) {
            throw new \Exception("Storage not found: {$storageId}");
        }

        $snapshotId = 'ss_' . bin2hex(random_bytes(12));
        $snapshotName = $params['name'] ?? "snap-{$storageId}-" . date('YmdHis');

        $provider = $this->getProvider($storage['provider']);
        $providerSnapshotId = $provider->createSnapshot($storage['volume_name'], $snapshotName);

        $this->db->insert('mod_storage_snapshots', [
            'id' => $snapshotId,
            'storage_id' => $storageId,
            'provider_snapshot_id' => $providerSnapshotId,
            'name' => $snapshotName,
            'size_gb' => $storage['size_gb'],
            'status' => 'creating',
            'created_at' => date('Y-m-d H:i:s')
        ]);

        return [
            'success' => true,
            'snapshot_id' => $snapshotId,
            'name' => $snapshotName
        ];
    }

    /**
     * Restore from snapshot
     */
    public function restoreSnapshot(string $snapshotId): array {
        $snapshot = $this->db->select(
            "SELECT * FROM mod_storage_snapshots WHERE id = ?",
            [$snapshotId]
        )[0];

        if (!$snapshot) {
            throw new \Exception("Snapshot not found: {$snapshotId}");
        }

        $storage = $this->getStorage($snapshot->storage_id);
        $provider = $this->getProvider($storage['provider']);

        // Restore volume
        $provider->restoreSnapshot($snapshot->provider_snapshot_id);

        // Update snapshot status
        $this->db->update('mod_storage_snapshots', [
            'restored_at' => date('Y-m-d H:i:s')
        ], ['id' => $snapshotId]);

        return [
            'success' => true,
            'snapshot_id' => $snapshotId,
            'restored_at' => date('Y-m-d H:i:s')
        ];
    }

    /**
     * Set storage quota
     */
    public function setQuota(int $serviceId, array $quota): array {
        $quotaConfig = [
            'service_id' => $serviceId,
            'soft_limit_gb' => $quota['soft_limit'] ?? null,
            'hard_limit_gb' => $quota['hard_limit'] ?? null,
            'warning_threshold' => $quota['warning_threshold'] ?? 80,
            'enabled' => true
        ];

        $this->db->update('mod_storage_quotas', $quotaConfig, ['service_id' => $serviceId]);

        // Apply quota via quota tool
        $this->applyQuotaToService($serviceId, $quotaConfig);

        return [
            'success' => true,
            'service_id' => $serviceId,
            'quota' => $quotaConfig
        ];
    }

    /**
     * Get storage usage statistics
     */
    public function getUsageStats(int $serviceId): array {
        $storages = $this->db->select(
            "SELECT * FROM mod_storage_allocations WHERE service_id = ?",
            [$serviceId]
        );

        $stats = [];
        $totalSize = 0;
        $totalUsed = 0;

        foreach ($storages as $storage) {
            $usage = $this->getVolumeUsage($storage->volume_name);

            $stats[] = [
                'storage_id' => $storage->id,
                'size_gb' => $storage->size_gb,
                'used_gb' => $usage['used'],
                'available_gb' => $usage['available'],
                'usage_percent' => round(($usage['used'] / $storage->size_gb) * 100, 2),
                'mount_point' => $storage->mount_point
            ];

            $totalSize += $storage->size_gb;
            $totalUsed += $usage['used'];
        }

        return [
            'service_id' => $serviceId,
            'volumes' => $stats,
            'total_size_gb' => $totalSize,
            'total_used_gb' => round($totalUsed, 2),
            'total_available_gb' => round($totalSize - $totalUsed, 2),
            'overall_usage_percent' => $totalSize > 0 ? round(($totalUsed / $totalSize) * 100, 2) : 0
        ];
    }

    /**
     * List storage for service
     */
    public function listStorage(int $serviceId): array {
        $storages = $this->db->select(
            "SELECT * FROM mod_storage_allocations WHERE service_id = ? ORDER BY created_at DESC",
            [$serviceId]
        );

        return array_map(function($storage) {
            return [
                'id' => $storage->id,
                'volume_name' => $storage->volume_name,
                'size_gb' => $storage->size_gb,
                'type' => $storage->type,
                'tier' => $storage->tier,
                'provider' => $storage->provider,
                'mount_point' => $storage->mount_point,
                'filesystem' => $storage->filesystem,
                'created_at' => $storage->created_at
            ];
        }, $storages);
    }

    /**
     * Delete storage
     */
    public function deleteStorage(string $storageId, bool $force = false): bool {
        $storage = $this->getStorage($storageId);

        if (!$storage) {
            return false;
        }

        // Check for snapshots
        $snapshots = $this->db->select(
            "SELECT COUNT(*) as count FROM mod_storage_snapshots WHERE storage_id = ? AND restored_at IS NULL",
            [$storageId]
        )[0]->count;

        if ($snapshots > 0 && !$force) {
            throw new \Exception("Storage has active snapshots. Use force=true to delete.");
        }

        // Unmount and detach
        $this->unmountStorage($storageId);
        $this->detachFromVM($storageId, $storage['service_id']);

        // Delete via provider
        $provider = $this->getProvider($storage['provider']);
        $provider->delete($storage['volume_name']);

        // Delete snapshots
        $this->db->delete('mod_storage_snapshots', ['storage_id' => $storageId]);

        // Delete allocation
        $this->db->delete('mod_storage_allocations', ['id' => $storageId]);

        return true;
    }

    // Private helper methods

    private function attachToVM(string $storageId, int $serviceId): void {
        $vmId = $this->getVmId($serviceId);
        $storage = $this->getStorage($storageId);
        $hypervisor = $this->getHypervisorType($serviceId);

        // Attach via hypervisor API
        switch ($hypervisor) {
            case 'proxmox':
                exec("qm set {$vmId} --scsi1 {$storage['provider']}:{$storage['volume_name']}");
                break;
            case 'vmware':
                exec("govc vm.disk.attach -vm {$vmId} -disk {$storage['volume_name']}");
                break;
        }
    }

    private function formatAndMount(string $storageId): void {
        $storage = $this->getStorage($storageId);
        $devicePath = "/dev/{$storage['provider']}/{$storage['volume_name']}";
        $mountPoint = $storage['mount_point'];

        // Create mount point directory
        if (!is_dir($mountPoint)) {
            mkdir($mountPoint, 0755, true);
        }

        // Format filesystem
        exec("mkfs.{$storage['filesystem']} -F {$devicePath}");

        // Mount
        exec("mount {$devicePath} {$mountPoint}");

        // Add to fstab
        $fstabEntry = "{$devicePath} {$mountPoint} {$storage['filesystem']} defaults 0 2";
        file_put_contents('/etc/fstab', $fstabEntry . "\n", FILE_APPEND);
    }

    private function resizeFilesystem(string $storageId, int $newSizeGB): void {
        $storage = $this->getStorage($storageId);
        $devicePath = "/dev/{$storage['provider']}/{$storage['volume_name']}";

        exec("resize2fs {$devicePath}");
    }

    private function getVolumeUsage(string $volumeName): array {
        // Use df command to get usage
        $output = [];
        exec("df -BG | grep {$volumeName}", $output);

        if (empty($output)) {
            return ['used' => 0, 'available' => 0];
        }

        // Parse output
        preg_match('/(\d+)G\s+(\d+)G\s+(\d+)G\s+(\d+)%/', $output[0], $matches);

        return [
            'used' => (int) $matches[2] ?? 0,
            'available' => (int) $matches[3] ?? 0,
            'percent' => (int) $matches[4] ?? 0
        ];
    }

    private function getProvider(string $type) {
        if (!isset($this->providers[$type])) {
            throw new \Exception("Unknown storage provider: {$type}");
        }
        return $this->providers[$type];
    }
}

/**
 * Storage Provider Interface
 */
interface StorageProviderInterface {
    public function create(string $volumeName, int $sizeGB, array $options): array;
    public function delete(string $volumeName): bool;
    public function expand(string $volumeName, int $newSizeGB): void;
    public function createSnapshot(string $volumeName, string $snapshotName): string;
    public function restoreSnapshot(string $snapshotId): void;
}

class CephStorageProvider implements StorageProviderInterface {
    public function create(string $volumeName, int $sizeGB, array $options): array {
        $poolName = $options['pool'] ?? 'vms';

        exec("rbd create {$poolName}/{$volumeName} --size {$sizeGB}G");

        return ['volume_name' => $volumeName, 'pool' => $poolName];
    }

    public function delete(string $volumeName): bool {
        exec("rbd rm {$volumeName}");
        return true;
    }

    public function expand(string $volumeName, int $newSizeGB): void {
        exec("rbd resize {$volumeName} --size {$newSizeGB}G --allow-shrink");
    }

    public function createSnapshot(string $volumeName, string $snapshotName): string {
        exec("rbd snap create {$volumeName}@{$snapshotName}");
        return $snapshotName;
    }

    public function restoreSnapshot(string $snapshotId): void {
        exec("rbd snap rollback {$snapshotId}");
    }
}

class AWSEBSProvider implements StorageProviderInterface {
    private $region;

    public function __construct() {
        $config = $this->loadConfig();
        $this->region = $config['aws_region'];
    }

    public function create(string $volumeName, int $sizeGB, array $options): array {
        $volumeType = $options['type'] ?? 'gp3';
        $encrypted = $options['encrypted'] ?? true;

        $command = "aws ec2 create-volume --size {$sizeGB} --region {$this->region} --volume-type {$volumeType} --encrypted {$encrypted} --tag-specifications ResourceType=volume,Tags=[{Key=Name,Value={$volumeName}}]";

        $result = shell_exec($command);
        $data = json_decode($result, true);

        return [
            'volume_id' => $data['VolumeId'],
            'az' => $data['AvailabilityZone']
        ];
    }

    public function delete(string $volumeName): bool {
        $volumeId = $this->getVolumeId($volumeName);
        exec("aws ec2 delete-volume --volume-id {$volumeId} --region {$this->region}");
        return true;
    }

    public function expand(string $volumeName, int $newSizeGB): void {
        $volumeId = $this->getVolumeId($volumeName);
        exec("aws ec2 modify-volume --volume-id {$volumeId} --size {$newSizeGB} --region {$this->region}");
    }
}
```

## Database Schema
```sql
CREATE TABLE `mod_storage_allocations` (
  `id` VARCHAR(50) PRIMARY KEY,
  `service_id` INT NOT NULL,
  `volume_name` VARCHAR(255) NOT NULL,
  `size_gb` INT NOT NULL,
  `type` ENUM('ssd', 'hdd', 'nvme') DEFAULT 'ssd',
  `tier` ENUM('standard', 'premium', 'cold') DEFAULT 'standard',
  `provider` VARCHAR(50) DEFAULT 'local',
  `mount_point` VARCHAR(255),
  `filesystem` VARCHAR(20) DEFAULT 'ext4',
  `created_at` DATETIME NOT NULL,
  `expanded_at` DATETIME,
  INDEX `idx_service_id` (`service_id`)
);

CREATE TABLE `mod_storage_snapshots` (
  `id` VARCHAR(50) PRIMARY KEY,
  `storage_id` VARCHAR(50) NOT NULL,
  `provider_snapshot_id` VARCHAR(100) NOT NULL,
  `name` VARCHAR(255) NOT NULL,
  `size_gb` INT NOT NULL,
  `status` ENUM('creating', 'available', 'error', 'deleted') DEFAULT 'creating',
  `created_at` DATETIME NOT NULL,
  `restored_at` DATETIME,
  FOREIGN KEY (`storage_id`) REFERENCES `mod_storage_allocations`(`id`)
);

CREATE TABLE `mod_storage_quotas` (
  `service_id` INT PRIMARY KEY,
  `soft_limit_gb` INT,
  `hard_limit_gb` INT,
  `warning_threshold` INT DEFAULT 80,
  `enabled` TINYINT(1) DEFAULT 1,
  `updated_at` DATETIME NOT NULL
);
```

## Best Practices

1. **Storage Tiers**: Use appropriate storage tiers based on access patterns
2. **Snapshots**: Regular snapshots for data protection
3. **Monitoring**: Track storage usage and performance
4. **Expansion**: Plan for growth with easy expansion capabilities
5. **Backup**: Integrate storage snapshots with backup systems

## Related Skills

- whmcs-backup-scheduling
- whmcs-snapshot-management
- whmcs-resource-quotas
- whmcs-provisioning-master