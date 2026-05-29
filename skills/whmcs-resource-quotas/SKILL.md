---
name: whmcs-resource-quotas
description: Resource limits for WHMCS services
category: Provisioning & Cloud
version: 1.0.0
---

# WHMCS Resource Quotas Skill

## Overview
This skill provides patterns and implementations for managing resource quotas in WHMCS, including CPU, memory, storage, bandwidth limits, and quota enforcement mechanisms.

## Implementation Patterns

### Resource Quota Manager
```php
<?php
/**
 * WHMCS Resource Quotas
 * Manages resource limits for hosted services
 */

namespace WHMCS\Module\Server\Quotas;

class ResourceQuotaManager {
    private $db;
    private $enforcementHandlers = [];

    public function __construct() {
        $this->db = \WHMCS\Database\Capsule::connection();
        $this->initializeHandlers();
    }

    private function initializeHandlers(): void {
        $this->enforcementHandlers = [
            'cgroups' => new CgroupsEnforcement(),
            'cloud' => new CloudQuotaEnforcement(),
            'ufd' => new UFDEnforcement()
        ];
    }

    /**
     * Create resource quota
     */
    public function createQuota(array $params): array {
        $quotaId = 'q_' . bin2hex(random_bytes(12));

        $quota = [
            'id' => $quotaId,
            'name' => $params['name'],
            'cpu_cores' => $params['cpu_cores'] ?? null,
            'cpu_percent' => $params['cpu_percent'] ?? null,
            'memory_mb' => $params['memory_mb'] ?? null,
            'memory_percent' => $params['memory_percent'] ?? null,
            'disk_gb' => $params['disk_gb'] ?? null,
            'bandwidth_gb' => $params['bandwidth_gb'] ?? null,
            'iops_limit' => $params['iops_limit'] ?? null,
            'network_mbps' => $params['network_mbps'] ?? null,
            'processes' => $params['processes'] ?? null,
            'description' => $params['description'] ?? '',
            'is_default' => $params['is_default'] ?? false,
            'created_at' => date('Y-m-d H:i:s')
        ];

        $this->db->insert('mod_resource_quotas', $quota);

        return [
            'success' => true,
            'quota_id' => $quotaId,
            'name' => $params['name']
        ];
    }

    /**
     * Assign quota to service
     */
    public function assignQuota(int $serviceId, string $quotaId): array {
        $quota = $this->getQuota($quotaId);

        if (!$quota) {
            throw new \Exception("Quota not found: {$quotaId}");
        }

        // Check if quota already assigned
        $existing = $this->db->select(
            "SELECT id FROM mod_service_quotas WHERE service_id = ?",
            [$serviceId]
        );

        if ($existing) {
            $this->db->update('mod_service_quotas', [
                'quota_id' => $quotaId,
                'updated_at' => date('Y-m-d H:i:s')
            ], ['service_id' => $serviceId]);
        } else {
            $this->db->insert('mod_service_quotas', [
                'service_id' => $serviceId,
                'quota_id' => $quotaId,
                'created_at' => date('Y-m-d H:i:s')
            ]);
        }

        // Apply quota to VM
        $this->applyQuotaToService($serviceId, $quota);

        return [
            'success' => true,
            'service_id' => $serviceId,
            'quota_id' => $quotaId
        ];
    }

    /**
     * Get current resource usage
     */
    public function getUsage(int $serviceId): array {
        $serviceQuota = $this->db->select(
            "SELECT q.* FROM mod_resource_quotas q
             JOIN mod_service_quotas sq ON q.id = sq.quota_id
             WHERE sq.service_id = ?",
            [$serviceId]
        )[0];

        if (!$serviceQuota) {
            return ['status' => 'no_quota', 'service_id' => $serviceId];
        }

        // Get current VM usage
        $vmId = $this->getVmId($serviceId);
        $currentUsage = $this->getCurrentVMUsage($vmId);

        $usage = [];

        // CPU usage
        $usage['cpu'] = [
            'current_cores' => $currentUsage['cpu_cores'],
            'max_cores' => $serviceQuota->cpu_cores,
            'current_percent' => $currentUsage['cpu_percent'],
            'max_percent' => $serviceQuota->cpu_percent,
            'utilization_percent' => $serviceQuota->cpu_cores ?
                round(($currentUsage['cpu_percent'] / $serviceQuota->cpu_percent) * 100, 2) : 0
        ];

        // Memory usage
        $usage['memory'] = [
            'current_mb' => $currentUsage['memory_mb'],
            'max_mb' => $serviceQuota->memory_mb,
            'utilization_percent' => $serviceQuota->memory_mb ?
                round(($currentUsage['memory_mb'] / $serviceQuota->memory_mb) * 100, 2) : 0
        ];

        // Disk usage
        $usage['disk'] = [
            'current_gb' => $currentUsage['disk_gb'],
            'max_gb' => $serviceQuota->disk_gb,
            'utilization_percent' => $serviceQuota->disk_gb ?
                round(($currentUsage['disk_gb'] / $serviceQuota->disk_gb) * 100, 2) : 0
        ];

        return [
            'service_id' => $serviceId,
            'quota_id' => $serviceQuota->id,
            'usage' => $usage,
            'enforced' => (bool) $serviceQuota->enforced
        ];
    }

    /**
     * Check if service is within quota
     */
    public function checkQuota(int $serviceId): array {
        $usage = $this->getUsage($serviceId);

        if (!isset($usage['usage'])) {
            return ['within_quota' => true, 'service_id' => $serviceId];
        }

        $violations = [];

        foreach ($usage['usage'] as $resource => $data) {
            if (isset($data['utilization_percent']) && $data['utilization_percent'] > 100) {
                $violations[] = [
                    'resource' => $resource,
                    'utilization' => $data['utilization_percent'],
                    'limit' => 100
                ];
            }
        }

        return [
            'within_quota' => empty($violations),
            'service_id' => $serviceId,
            'violations' => $violations
        ];
    }

    /**
     * Enforce quota limits
     */
    public function enforceQuota(int $serviceId, string $action = 'warn'): array {
        $violations = $this->checkQuota($serviceId);

        if ($violations['within_quota']) {
            return ['enforced' => false, 'message' => 'No violations'];
        }

        $vmId = $this->getVmId($serviceId);
        $handler = $this->getEnforcementHandler($serviceId);

        foreach ($violations['violations'] as $violation) {
            switch ($action) {
                case 'throttle':
                    $handler->throttle($vmId, $violation['resource']);
                    break;
                case 'kill':
                    $handler->killProcess($vmId, $violation['resource']);
                    break;
                case 'suspend':
                    $this->suspendService($serviceId);
                    break;
                case 'warn':
                default:
                    $this->sendQuotaWarning($serviceId, $violation);
                    break;
            }
        }

        return [
            'enforced' => true,
            'action' => $action,
            'violations' => $violations['violations']
        ];
    }

    /**
     * Update quota limits
     */
    public function updateQuota(string $quotaId, array $updates): array {
        $allowedFields = [
            'cpu_cores', 'cpu_percent', 'memory_mb', 'memory_percent',
            'disk_gb', 'bandwidth_gb', 'iops_limit', 'network_mbps',
            'processes', 'description', 'is_default'
        ];

        $updateData = [];
        foreach ($allowedFields as $field) {
            if (isset($updates[$field])) {
                $updateData[$field] = $updates[$field];
            }
        }

        $updateData['updated_at'] = date('Y-m-d H:i:s');

        $this->db->update('mod_resource_quotas', $updateData, ['id' => $quotaId]);

        // Re-apply to all services using this quota
        $this->reapplyQuotaToServices($quotaId);

        return [
            'success' => true,
            'quota_id' => $quotaId,
            'updated_fields' => array_keys($updateData)
        ];
    }

    /**
     * Get quota templates
     */
    public function getQuotaTemplates(): array {
        $templates = $this->db->select(
            "SELECT * FROM mod_resource_quota_templates ORDER BY tier ASC"
        );

        return array_map(function($template) {
            return [
                'id' => $template->id,
                'name' => $template->name,
                'tier' => $template->tier,
                'cpu_cores' => $template->cpu_cores,
                'memory_mb' => $template->memory_mb,
                'disk_gb' => $template->disk_gb,
                'bandwidth_gb' => $template->bandwidth_gb
            ];
        }, $templates);
    }

    // Private helper methods

    private function applyQuotaToService(int $serviceId, array $quota): void {
        $vmId = $this->getVmId($serviceId);
        $handler = $this->getEnforcementHandler($serviceId);

        $handler->apply([
            'vm_id' => $vmId,
            'cpu_cores' => $quota['cpu_cores'],
            'memory_mb' => $quota['memory_mb'],
            'disk_gb' => $quota['disk_gb'],
            'iops_limit' => $quota['iops_limit']
        ]);
    }

    private function getEnforcementHandler(int $serviceId): object {
        $hypervisor = $this->getHypervisorType($serviceId);

        return match($hypervisor) {
            'proxmox', 'vmware' => $this->enforcementHandlers['cgroups'],
            'aws', 'gcp', 'azure' => $this->enforcementHandlers['cloud'],
            default => $this->enforcementHandlers['cgroups']
        };
    }

    private function getCurrentVMUsage(string $vmId): array {
        // Get real-time usage from hypervisor or monitoring
        return [
            'cpu_cores' => 2,
            'cpu_percent' => 45,
            'memory_mb' => 1024,
            'disk_gb' => 50
        ];
    }
}

/**
 * Quota Enforcement Handlers
 */
interface QuotaEnforcementInterface {
    public function apply(array $quota): void;
    public function throttle(string $vmId, string $resource): void;
}

class CgroupsEnforcement implements QuotaEnforcementInterface {
    public function apply(array $quota): void {
        $vmId = $quota['vm_id'];

        // Apply CPU limits
        if ($quota['cpu_cores']) {
            exec("echo {$quota['cpu_cores']} > /sys/fs/cgroup/cpu/vm{$vmId}/cpu.cfs_quota_us");
        }

        // Apply memory limits
        if ($quota['memory_mb']) {
            exec("echo {$quota['memory_mb']}M > /sys/fs/cgroup/memory/vm{$vmId}/memory.limit_in_bytes");
        }
    }

    public function throttle(string $vmId, string $resource): void {
        // Implement throttling based on resource
        exec("echo 50000 > /sys/fs/cgroup/cpu/vm{$vmId}/cpu.cfs_quota_us");
    }
}

class CloudQuotaEnforcement implements QuotaEnforcementInterface {
    public function apply(array $quota): void {
        // For cloud providers, update instance limits
        $instanceId = $quota['instance_id'];

        // AWS
        exec("aws ec2 modify-instance-attribute --instance-id {$instanceId} --cpu-options");

        // GCP
        exec("gcloud compute instances set-machine-type {$instanceId} --machine-type=custom");
    }

    public function throttle(string $vmId, string $resource): void {
        // Implement cloud-specific throttling
    }
}
```

## Database Schema
```sql
CREATE TABLE `mod_resource_quotas` (
  `id` VARCHAR(50) PRIMARY KEY,
  `name` VARCHAR(255) NOT NULL,
  `cpu_cores` INT,
  `cpu_percent` INT,
  `memory_mb` INT,
  `memory_percent` INT,
  `disk_gb` INT,
  `bandwidth_gb` INT,
  `iops_limit` INT,
  `network_mbps` INT,
  `processes` INT,
  `enforced` TINYINT(1) DEFAULT 0,
  `description` TEXT,
  `is_default` TINYINT(1) DEFAULT 0,
  `created_at` DATETIME NOT NULL,
  `updated_at` DATETIME
);

CREATE TABLE `mod_service_quotas` (
  `service_id` INT PRIMARY KEY,
  `quota_id` VARCHAR(50) NOT NULL,
  `created_at` DATETIME NOT NULL,
  `updated_at` DATETIME
);

CREATE TABLE `mod_resource_quota_templates` (
  `id` INT AUTO_INCREMENT PRIMARY KEY,
  `name` VARCHAR(100) NOT NULL,
  `tier` VARCHAR(50) NOT NULL,
  `cpu_cores` INT NOT NULL,
  `memory_mb` INT NOT NULL,
  `disk_gb` INT NOT NULL,
  `bandwidth_gb` INT NOT NULL,
  `iops_limit` INT
);

CREATE TABLE `mod_quota_violations` (
  `id` INT AUTO_INCREMENT PRIMARY KEY,
  `service_id` INT NOT NULL,
  `resource` VARCHAR(50) NOT NULL,
  `current_usage` FLOAT NOT NULL,
  `limit` FLOAT NOT NULL,
  `action_taken` VARCHAR(50),
  `created_at` DATETIME NOT NULL,
  INDEX `idx_service_id` (`service_id`)
);
```

## Quota Templates
| Tier | CPU Cores | Memory (MB) | Disk (GB) | Bandwidth (GB) |
|------|-----------|-------------|-----------|----------------|
| Starter | 1 | 1024 | 20 | 100 |
| Basic | 2 | 2048 | 50 | 500 |
| Standard | 4 | 4096 | 100 | 1000 |
| Professional | 8 | 8192 | 200 | 2000 |
| Enterprise | 16 | 16384 | 500 | 5000 |

## Best Practices

1. **Graceful Enforcement**: Warn before hard limits
2. **Flexible Quotas**: Allow burst capabilities
3. **Clear Visibility**: Show quota usage to customers
4. **Tiered Approach**: Offer different quota levels
5. **Automatic Upgrades**: Suggest upgrades when approaching limits

## Related Skills

- whmcs-auto-scaling
- whmcs-storage-provisioning
- whmcs-monitoring-agent
- whmcs-billing-dimensions