---
name: whmcs-snapshot-management
description: Snapshot workflows for WHMCS VM management
category: Provisioning & Cloud
version: 1.0.0
---

# WHMCS Snapshot Management Skill

## Overview
This skill provides patterns and implementations for managing VM snapshots in WHMCS provisioning, including snapshot creation, restoration, scheduling, and lifecycle management.

## Implementation Patterns

### Snapshot Manager Class
```php
<?php
/**
 * WHMCS VM Snapshot Management
 * Handles VM snapshot lifecycle for backup and recovery
 */

namespace WHMCS\Module\Server\Snapshot;

class SnapshotManager {
    private $db;
    private $hypervisorAdapters = [];

    public function __construct() {
        $this->db = \WHMCS\Database\Capsule::connection();
        $this->initializeHypervisors();
    }

    private function initializeHypervisors(): void {
        $this->hypervisorAdapters = [
            'proxmox' => new ProxmoxSnapshotAdapter(),
            'vmware' => new VMwareSnapshotAdapter(),
            'libvirt' => new LibvirtSnapshotAdapter()
        ];
    }

    /**
     * Create a snapshot of a VM
     */
    public function createSnapshot(array $params): array {
        $this->validateSnapshotParams($params);

        $serviceId = $params['service_id'];
        $vmId = $this->getVmIdForService($serviceId);
        $hypervisor = $this->getHypervisorForService($serviceId);

        $snapshotName = $params['name'] ?? $this->generateSnapshotName($serviceId);
        $description = $params['description'] ?? '';
        $snapshotType = $params['type'] ?? 'online'; // online, offline, memory

        // Get adapter for hypervisor
        $adapter = $this->getAdapter($hypervisor);

        // Create snapshot via hypervisor API
        $hypervisorSnapshotId = $adapter->create($vmId, [
            'name' => $snapshotName,
            'description' => $description,
            'type' => $snapshotType,
            'wait_for_completion' => $params['wait'] ?? true
        ]);

        // Store snapshot metadata in WHMCS
        $snapshotId = $this->storeSnapshotMetadata([
            'service_id' => $serviceId,
            'vm_id' => $vmId,
            'hypervisor_snapshot_id' => $hypervisorSnapshotId,
            'name' => $snapshotName,
            'description' => $description,
            'type' => $snapshotType,
            'size_mb' => $adapter->getSnapshotSize($vmId, $hypervisorSnapshotId),
            'hypervisor_type' => $hypervisor
        ]);

        // Log the action
        $this->logSnapshotAction($snapshotId, 'create', $params);

        return [
            'success' => true,
            'snapshot_id' => $snapshotId,
            'hypervisor_snapshot_id' => $hypervisorSnapshotId,
            'name' => $snapshotName,
            'created_at' => date('Y-m-d H:i:s')
        ];
    }

    /**
     * List snapshots for a service
     */
    public function listSnapshots(int $serviceId): array {
        $snapshots = $this->db->select(
            "SELECT * FROM mod_vm_snapshots WHERE service_id = ? ORDER BY created_at DESC",
            [$serviceId]
        );

        return array_map(function($snapshot) {
            return $this->formatSnapshotResponse($snapshot);
        }, $snapshots);
    }

    /**
     * Restore a snapshot
     */
    public function restoreSnapshot(array $params): array {
        $snapshotId = $params['snapshot_id'];

        $snapshot = $this->getSnapshot($snapshotId);
        if (!$snapshot) {
            throw new \Exception("Snapshot not found: {$snapshotId}");
        }

        $vmId = $snapshot['vm_id'];
        $hypervisor = $snapshot['hypervisor_type'];
        $adapter = $this->getAdapter($hypervisor);

        // Check if service is suspended/terminated
        $service = $this->getService($snapshot['service_id']);
        if ($service['domainstatus'] === 'Suspended') {
            // Auto-unsuspend for restore operation
            $this->unsuspendService($snapshot['service_id']);
        }

        // Perform restore
        $adapter->restore($vmId, $snapshot['hypervisor_snapshot_id'], [
            'preserve_memory' => $params['preserve_memory'] ?? false
        ]);

        // Update snapshot metadata
        $this->db->update(
            'mod_vm_snapshots',
            ['restored_at' => date('Y-m-d H:i:s')],
            ['id' => $snapshotId]
        );

        $this->logSnapshotAction($snapshotId, 'restore', $params);

        return [
            'success' => true,
            'snapshot_id' => $snapshotId,
            'restored_at' => date('Y-m-d H:i:s')
        ];
    }

    /**
     * Delete a snapshot
     */
    public function deleteSnapshot(string $snapshotId, bool $force = false): array {
        $snapshot = $this->getSnapshot($snapshotId);
        if (!$snapshot) {
            throw new \Exception("Snapshot not found: {$snapshotId}");
        }

        // Check if this is the only snapshot (prevent orphan VM)
        $snapshotCount = $this->db->select(
            "SELECT COUNT(*) as count FROM mod_vm_snapshots WHERE service_id = ?",
            [$snapshot['service_id']]
        )[0]->count;

        if ($snapshotCount <= 1 && !$force) {
            throw new \Exception("Cannot delete the only snapshot. Use force=true to override.");
        }

        $adapter = $this->getAdapter($snapshot['hypervisor_type']);

        // Delete from hypervisor
        $adapter->delete($snapshot['vm_id'], $snapshot['hypervisor_snapshot_id']);

        // Delete from database
        $this->db->delete('mod_vm_snapshots', ['id' => $snapshotId]);

        $this->logSnapshotAction($snapshotId, 'delete', ['force' => $force]);

        return [
            'success' => true,
            'snapshot_id' => $snapshotId,
            'deleted_at' => date('Y-m-d H:i:s')
        ];
    }

    /**
     * Schedule automatic snapshots
     */
    public function scheduleSnapshots(array $params): array {
        $serviceId = $params['service_id'];
        $schedule = $params['schedule'];
        $retentionCount = $params['retention_count'] ?? 7;

        // Validate schedule format
        if (!preg_match('/^[\d\*\/\-\,]+$/', $schedule)) {
            throw new \Exception("Invalid cron schedule format: {$schedule}");
        }

        // Store schedule configuration
        $this->db->delete('mod_snapshot_schedules', ['service_id' => $serviceId]);
        $this->db->insert('mod_snapshot_schedules', [
            'service_id' => $serviceId,
            'cron_schedule' => $schedule,
            'retention_count' => $retentionCount,
            'description' => $params['description'] ?? '',
            'enabled' => $params['enabled'] ?? true,
            'created_at' => date('Y-m-d H:i:s')
        ]);

        return [
            'success' => true,
            'service_id' => $serviceId,
            'schedule' => $schedule,
            'retention_count' => $retentionCount
        ];
    }

    /**
     * Get scheduled snapshots for cron execution
     */
    public function getScheduledSnapshots(): array {
        $schedules = $this->db->select(
            "SELECT * FROM mod_snapshot_schedules WHERE enabled = 1"
        );

        $dueSnapshots = [];
        foreach ($schedules as $schedule) {
            if ($this->isCronDue($schedule->cron_schedule)) {
                $dueSnapshots[] = $schedule;
            }
        }

        return $dueSnapshots;
    }

    /**
     * Prune old snapshots based on retention policy
     */
    public function pruneSnapshots(int $serviceId): array {
        $schedule = $this->db->select(
            "SELECT * FROM mod_snapshot_schedules WHERE service_id = ?",
            [$serviceId]
        );

        if (empty($schedule)) {
            return ['pruned' => 0];
        }

        $retentionCount = $schedule[0]->retention_count;

        // Get all snapshots ordered by creation date
        $snapshots = $this->db->select(
            "SELECT id, created_at FROM mod_vm_snapshots
             WHERE service_id = ? ORDER BY created_at DESC",
            [$serviceId]
        );

        $pruned = 0;
        $deletedIds = [];

        foreach ($snapshots as $index => $snapshot) {
            if ($index >= $retentionCount) {
                $this->deleteSnapshot($snapshot->id, true);
                $pruned++;
                $deletedIds[] = $snapshot->id;
            }
        }

        return [
            'pruned' => $pruned,
            'deleted_ids' => $deletedIds
        ];
    }

    // Hypervisor adapter interface

    private function getAdapter(string $hypervisor) {
        if (!isset($this->hypervisorAdapters[$hypervisor])) {
            throw new \Exception("Unsupported hypervisor: {$hypervisor}");
        }
        return $this->hypervisorAdapters[$hypervisor];
    }

    private function formatSnapshotResponse($snapshot): array {
        return [
            'id' => $snapshot->id,
            'service_id' => $snapshot->service_id,
            'vm_id' => $snapshot->vm_id,
            'hypervisor_snapshot_id' => $snapshot->hypervisor_snapshot_id,
            'name' => $snapshot->name,
            'description' => $snapshot->description,
            'type' => $snapshot->type,
            'size_mb' => round($snapshot->size_mb, 2),
            'hypervisor_type' => $snapshot->hypervisor_type,
            'created_at' => $snapshot->created_at,
            'restored_at' => $snapshot->restored_at
        ];
    }

    private function generateSnapshotName(int $serviceId): string {
        return sprintf(
            'auto-snap-%d-%s',
            $serviceId,
            date('Ymd-His')
        );
    }

    private function isCronDue(string $cronExpression): bool {
        // Simple cron checking (production should use a proper cron library)
        return true; // Placeholder
    }
}

/**
 * Hypervisor Snapshot Adapter Interface
 */
interface SnapshotAdapterInterface {
    public function create(string $vmId, array $options): string;
    public function restore(string $vmId, string $snapshotId, array $options): void;
    public function delete(string $vmId, string $snapshotId): void;
    public function getSnapshotSize(string $vmId, string $snapshotId): int;
    public function list(string $vmId): array;
}

class ProxmoxSnapshotAdapter implements SnapshotAdapterInterface {
    private $apiUrl;
    private $apiToken;

    public function __construct() {
        $config = $this->loadConfig();
        $this->apiUrl = $config['api_url'];
        $this->apiToken = $config['api_token'];
    }

    public function create(string $vmId, array $options): string {
        $node = $this->getNodeForVM($vmId);
        $vmid = $this->extractVMId($vmId);

        $url = "https://{$this->apiUrl}/api2/json/nodes/{$node}/qemu/{$vmid}/snapshot";

        $postData = [
            'snapname' => $options['name'],
            'description' => $options['description'] ?? '',
            'description' => $options['description'] ?? ''
        ];

        if ($options['type'] === 'memory') {
            $postData['vmstate'] = 1;
        }

        $response = $this->apiRequest('POST', $url, $postData);

        return $response['data'];
    }

    public function restore(string $vmId, string $snapshotId, array $options): void {
        $node = $this->getNodeForVM($vmId);
        $vmid = $this->extractVMId($vmId);

        $url = "https://{$this->apiUrl}/api2/json/nodes/{$node}/qemu/{$vmid}/snapshot/{$snapshotId}/rollback";

        $this->apiRequest('POST', $url, []);
    }

    public function delete(string $vmId, string $snapshotId): void {
        $node = $this->getNodeForVM($vmId);
        $vmid = $this->extractVMId($vmId);

        $url = "https://{$this->apiUrl}/api2/json/nodes/{$node}/qemu/{$vmid}/snapshot/{$snapshotId}";

        $this->apiRequest('DELETE', $url);
    }

    public function getSnapshotSize(string $vmId, string $snapshotId): int {
        $list = $this->list($vmId);
        foreach ($list as $snap) {
            if ($snap['name'] === $snapshotId) {
                return $snap['size'] ?? 0;
            }
        }
        return 0;
    }

    public function list(string $vmId): array {
        $node = $this->getNodeForVM($vmId);
        $vmid = $this->extractVMId($vmId);

        $url = "https://{$this->apiUrl}/api2/json/nodes/{$node}/qemu/{$vmid}/snapshot";

        $response = $this->apiRequest('GET', $url);

        return $response['data'] ?? [];
    }

    private function apiRequest(string $method, string $url, array $data = []): array {
        $ch = curl_init();
        curl_setopt_array($ch, [
            CURLOPT_URL => $url,
            CURLOPT_CUSTOMREQUEST => $method,
            CURLOPT_POSTFIELDS => http_build_query($data),
            CURLOPT_HTTPHEADER => [
                'Authorization: Bearer ' . $this->apiToken,
                'Content-Type: application/x-www-form-urlencoded'
            ],
            CURLOPT_RETURNTRANSFER => true
        ]);

        $response = curl_exec($ch);
        curl_close($ch);

        return json_decode($response, true) ?? [];
    }
}

class VMwareSnapshotAdapter implements SnapshotAdapterInterface {
    public function create(string $vmId, array $options): string {
        $snapshotName = $options['name'];
        $description = $options['description'] ?? '';

        $command = "govc snapshot.create -vm {$vmId} -name \"{$snapshotName}\" -description=\"{$description}\"";
        exec($command, $output, $return);

        if ($return !== 0) {
            throw new \Exception("Failed to create VMware snapshot");
        }

        return $snapshotName;
    }

    public function restore(string $vmId, string $snapshotId, array $options): void {
        $command = "govc snapshot.revert -vm {$vmId} -snapshot \"{$snapshotId}\"";
        exec($command);
    }

    public function delete(string $vmId, string $snapshotId): void {
        $command = "govc snapshot.remove -vm {$vmId} -snapshot \"{$snapshotId}\"";
        exec($command);
    }

    public function getSnapshotSize(string $vmId, string $snapshotId): int {
        $command = "govc snapshot.list -vm {$vmId} | grep \"{$snapshotId}\"";
        exec($command, $output);
        // Parse output to extract size
        return 0;
    }

    public function list(string $vmId): array {
        $command = "govc snapshot.list -vm {$vmId}";
        exec($command, $output);
        // Parse and return snapshot list
        return [];
    }
}
```

## Database Schema
```sql
CREATE TABLE `mod_vm_snapshots` (
  `id` VARCHAR(50) PRIMARY KEY,
  `service_id` INT NOT NULL,
  `vm_id` VARCHAR(100) NOT NULL,
  `hypervisor_snapshot_id` VARCHAR(100) NOT NULL,
  `name` VARCHAR(255) NOT NULL,
  `description` TEXT,
  `type` ENUM('online', 'offline', 'memory') DEFAULT 'online',
  `size_mb` BIGINT DEFAULT 0,
  `hypervisor_type` VARCHAR(50) NOT NULL,
  `created_at` DATETIME NOT NULL,
  `restored_at` DATETIME,
  INDEX `idx_service_id` (`service_id`),
  INDEX `idx_created_at` (`created_at`)
);

CREATE TABLE `mod_snapshot_schedules` (
  `id` INT AUTO_INCREMENT PRIMARY KEY,
  `service_id` INT NOT NULL,
  `cron_schedule` VARCHAR(100) NOT NULL,
  `retention_count` INT DEFAULT 7,
  `description` TEXT,
  `enabled` TINYINT(1) DEFAULT 1,
  `created_at` DATETIME NOT NULL,
  UNIQUE KEY `unique_service` (`service_id`)
);
```

## Hook Integration
```php
/**
 * Snapshot management hooks for WHMCS
 */

// After service creation - create initial snapshot
add_hook('ServiceProvision', 1, function($vars) {
    $snapshotManager = new \WHMCS\Module\Server\Snapshot\SnapshotManager();

    // Check if auto-snapshot is enabled for this product
    $service = Capsule::table('tblhosting')->where('id', $vars['serviceid'])->first();
    $product = Capsule::table('tblproducts')->where('id', $service->packageid)->first();

    if ($product->configoption10 ?? false) {
        $snapshotManager->createSnapshot([
            'service_id' => $vars['serviceid'],
            'name' => 'initial-' . $vars['serviceid'],
            'description' => 'Initial snapshot after provisioning',
            'type' => 'offline'
        ]);
    }
});

// Before service suspension - create pre-suspension snapshot
add_hook('ServiceSuspend', 1, function($vars) {
    $snapshotManager = new \WHMCS\Module\Server\Snapshot\SnapshotManager();

    // Create snapshot before suspension
    $snapshotManager->createSnapshot([
        'service_id' => $vars['serviceid'],
        'name' => 'pre-suspend-' . date('Ymd-His'),
        'description' => 'Snapshot before service suspension',
        'type' => 'offline'
    ]);
});
```

## Best Practices

1. **Retention Policies**: Implement automatic cleanup of old snapshots
2. **Consistency**: Use memory snapshots for running services
3. **Scheduling**: Use low-traffic periods for automated snapshots
4. **Monitoring**: Track snapshot storage usage
5. **Testing**: Regularly test snapshot restore procedures

## Related Skills

- whmcs-backup-scheduling
- whmcs-disaster-recovery-vm
- whmcs-provisioning-master
- whmcs-cloud-init