---
name: whmcs-auto-scaling
description: Auto-scaling configuration for WHMCS services
category: Provisioning & Cloud
version: 1.0.0
---

# WHMCS Auto-Scaling Configuration Skill

## Overview
This skill provides patterns and implementations for configuring auto-scaling in WHMCS, including scale-up/scale-down policies, resource monitoring thresholds, and capacity planning integration.

## Implementation Patterns

### Auto-Scaling Manager Class
```php
<?php
/**
 * WHMCS Auto-Scaling Configuration
 * Handles dynamic resource scaling based on utilization
 */

namespace WHMCS\Module\Server\AutoScale;

class AutoScaleManager {
    private $db;
    private $metricsCollector;
    private $scaleActions = [];

    public function __construct() {
        $this->db = \WHMCS\Database\Capsule::connection();
        $this->metricsCollector = new MetricsCollector();
    }

    /**
     * Create auto-scaling policy for a service
     */
    public function createPolicy(array $params): array {
        $this->validatePolicyParams($params);

        $policyId = 'asp_' . bin2hex(random_bytes(12));

        $policy = [
            'id' => $policyId,
            'service_id' => $params['service_id'],
            'metric_type' => $params['metric_type'], // cpu, memory, disk, network
            'scale_up_threshold' => $params['scale_up_threshold'] ?? 80,
            'scale_down_threshold' => $params['scale_down_threshold'] ?? 20,
            'scale_up_action' => $params['scale_up_action'] ?? 'add_resource',
            'scale_down_action' => $params['scale_down_action'] ?? 'remove_resource',
            'scale_increment' => $params['scale_increment'] ?? 1,
            'cooldown_minutes' => $params['cooldown_minutes'] ?? 5,
            'min_instances' => $params['min_instances'] ?? 1,
            'max_instances' => $params['max_instances'] ?? 10,
            'enabled' => $params['enabled'] ?? true,
            'created_at' => date('Y-m-d H:i:s')
        ];

        $this->db->insert('mod_autoscale_policies', $policy);

        return [
            'success' => true,
            'policy_id' => $policyId,
            'metric_type' => $policy['metric_type']
        ];
    }

    /**
     * Evaluate scaling conditions for all policies
     */
    public function evaluatePolicies(): array {
        $policies = $this->db->select(
            "SELECT * FROM mod_autoscale_policies WHERE enabled = 1"
        );

        $actionsTaken = [];

        foreach ($policies as $policy) {
            $action = $this->evaluatePolicy($policy);
            if ($action) {
                $actionsTaken[] = $action;
                $this->recordScaleAction($action);
            }
        }

        return [
            'evaluated' => count($policies),
            'actions' => $actionsTaken
        ];
    }

    /**
     * Evaluate a single policy
     */
    private function evaluatePolicy($policy): ?array {
        // Check if in cooldown period
        if ($this->isInCooldown($policy->id)) {
            return null;
        }

        // Get current metric value
        $currentValue = $this->metricsCollector->getCurrentValue(
            $policy->service_id,
            $policy->metric_type
        );

        $threshold = $policy->scale_up_threshold;
        $scaleUpTriggered = $currentValue >= $threshold;
        $scaleDownTriggered = $currentValue <= $policy->scale_down_threshold;

        if (!$scaleUpTriggered && !$scaleDownTriggered) {
            return null;
        }

        // Get current instance count
        $currentInstances = $this->getCurrentInstanceCount($policy->service_id);

        // Determine action
        if ($scaleUpTriggered) {
            // Check max instances limit
            if ($currentInstances >= $policy->max_instances) {
                return null;
            }

            return $this->executeScaleUp($policy, $currentValue);
        } else {
            // Check min instances limit
            if ($currentInstances <= $policy->min_instances) {
                return null;
            }

            return $this->executeScaleDown($policy, $currentValue);
        }
    }

    /**
     * Execute scale up action
     */
    private function executeScaleUp($policy, float $currentValue): array {
        $serviceId = $policy->service_id;
        $increment = $policy->scale_increment;

        // Get current resources
        $currentResources = $this->getServiceResources($serviceId);

        // Calculate new resources
        $newResources = $this->calculateScaledResources(
            $currentResources,
            $increment,
            'up'
        );

        // Perform upgrade
        $this->performResourceUpgrade($serviceId, $newResources);

        // Log scaling event
        $this->logScaleEvent([
            'policy_id' => $policy->id,
            'service_id' => $serviceId,
            'action' => 'scale_up',
            'metric_value' => $currentValue,
            'threshold' => $policy->scale_up_threshold,
            'previous_resources' => $currentResources,
            'new_resources' => $newResources
        ]);

        return [
            'policy_id' => $policy->id,
            'service_id' => $serviceId,
            'action' => 'scale_up',
            'metric_value' => $currentValue,
            'new_resources' => $newResources,
            'timestamp' => date('Y-m-d H:i:s')
        ];
    }

    /**
     * Execute scale down action
     */
    private function executeScaleDown($policy, float $currentValue): array {
        $serviceId = $policy->service_id;
        $increment = $policy->scale_increment;

        $currentResources = $this->getServiceResources($serviceId);

        $newResources = $this->calculateScaledResources(
            $currentResources,
            $increment,
            'down'
        );

        $this->performResourceDowngrade($serviceId, $newResources);

        $this->logScaleEvent([
            'policy_id' => $policy->id,
            'service_id' => $serviceId,
            'action' => 'scale_down',
            'metric_value' => $currentValue,
            'threshold' => $policy->scale_down_threshold,
            'previous_resources' => $currentResources,
            'new_resources' => $newResources
        ]);

        return [
            'policy_id' => $policy->id,
            'service_id' => $serviceId,
            'action' => 'scale_down',
            'metric_value' => $currentValue,
            'new_resources' => $newResources,
            'timestamp' => date('Y-m-d H:i:s')
        ];
    }

    /**
     * Calculate scaled resources
     */
    private function calculateScaledResources(array $current, int $increment, string $direction): array {
        $multiplier = $direction === 'up' ? (1 + ($increment * 0.25)) : (1 - ($increment * 0.25));

        return [
            'cpu_cores' => max(1, (int) ceil($current['cpu_cores'] * $multiplier)),
            'ram_mb' => max(512, (int) ceil($current['ram_mb'] * $multiplier)),
            'disk_gb' => max(10, (int) ceil($current['disk_gb'] * $multiplier))
        ];
    }

    /**
     * Perform resource upgrade
     */
    private function performResourceUpgrade(int $serviceId, array $newResources): void {
        $vmId = $this->getVmId($serviceId);
        $hypervisor = $this->getHypervisorType($serviceId);

        // Call hypervisor API to resize
        $this->resizeVM($vmId, $newResources, $hypervisor);

        // Update WHMCS service configuration
        $this->updateServiceResources($serviceId, $newResources);

        // Record billing adjustment if applicable
        $this->recordResourceUpgradeBilling($serviceId, $newResources);
    }

    private function performResourceDowngrade(int $serviceId, array $newResources): void {
        $vmId = $this->getVmId($serviceId);
        $hypervisor = $this->getHypervisorType($serviceId);

        $this->resizeVM($vmId, $newResources, $hypervisor);
        $this->updateServiceResources($serviceId, $newResources);
    }

    /**
     * Get scaling history
     */
    public function getScalingHistory(int $serviceId, int $limit = 50): array {
        $history = $this->db->select(
            "SELECT * FROM mod_autoscale_history
             WHERE service_id = ? ORDER BY created_at DESC LIMIT ?",
            [$serviceId, $limit]
        );

        return array_map(function($event) {
            return [
                'id' => $event->id,
                'policy_id' => $event->policy_id,
                'action' => $event->action,
                'metric_value' => $event->metric_value,
                'previous_resources' => json_decode($event->previous_resources, true),
                'new_resources' => json_decode($event->new_resources, true),
                'created_at' => $event->created_at
            ];
        }, $history);
    }

    /**
     * Get scaling metrics
     */
    public function getScalingMetrics(int $serviceId, \DateTime $from, \DateTime $to): array {
        $history = $this->db->select(
            "SELECT DATE(created_at) as date, action, COUNT(*) as count
             FROM mod_autoscale_history
             WHERE service_id = ? AND created_at BETWEEN ? AND ?
             GROUP BY DATE(created_at), action",
            [$serviceId, $from->format('Y-m-d'), $to->format('Y-m-d')]
        );

        return $history;
    }

    // Helper methods

    private function getCurrentInstanceCount(int $serviceId): int {
        // Get current instance count from hypervisor or config
        return 1;
    }

    private function isInCooldown(string $policyId): bool {
        $lastAction = $this->db->select(
            "SELECT created_at FROM mod_autoscale_history
             WHERE policy_id = ? ORDER BY created_at DESC LIMIT 1",
            [$policyId]
        );

        if (empty($lastAction)) {
            return false;
        }

        $policy = $this->db->select(
            "SELECT cooldown_minutes FROM mod_autoscale_policies WHERE id = ?",
            [$policyId]
        )[0];

        $cooldownEnd = strtotime($lastAction[0]->created_at) + ($policy->cooldown_minutes * 60);

        return time() < $cooldownEnd;
    }

    private function recordScaleAction(array $action): void {
        $this->db->insert('mod_autoscale_history', [
            'policy_id' => $action['policy_id'],
            'service_id' => $action['service_id'],
            'action' => $action['action'],
            'metric_value' => $action['metric_value'],
            'previous_resources' => json_encode($action['previous_resources']),
            'new_resources' => json_encode($action['new_resources']),
            'created_at' => date('Y-m-d H:i:s')
        ]);
    }

    private function getServiceResources(int $serviceId): array {
        $service = $this->db->select(
            "SELECT configoption3 as cpu, configoption4 as ram, configoption5 as disk
             FROM tblhosting WHERE id = ?",
            [$serviceId]
        )[0];

        return [
            'cpu_cores' => (int) ($service->cpu ?? 1),
            'ram_mb' => (int) ($service->ram ?? 1024),
            'disk_gb' => (int) ($service->disk ?? 20)
        ];
    }

    private function resizeVM(string $vmId, array $resources, string $hypervisor): void {
        // Hypervisor-specific resize implementation
        switch ($hypervisor) {
            case 'proxmox':
                $this->resizeProxmoxVM($vmId, $resources);
                break;
            case 'vmware':
                $this->resizeVMwareVM($vmId, $resources);
                break;
        }
    }

    private function resizeProxmoxVM(string $vmId, array $resources): void {
        $command = "qm resize {$vmId} scsi0 {$resources['disk_gb']}G";
        exec($command);

        $command = "qm set {$vmId} --cores {$resources['cpu_cores']} --memory {$resources['ram_mb']}";
        exec($command);
    }
}

/**
 * Metrics Collector for auto-scaling decisions
 */
class MetricsCollector {
    public function getCurrentValue(int $serviceId, string $metricType): float {
        // Get current metric value from monitoring system
        switch ($metricType) {
            case 'cpu':
                return $this->getCpuUsage($serviceId);
            case 'memory':
                return $this->getMemoryUsage($serviceId);
            case 'disk':
                return $this->getDiskUsage($serviceId);
            case 'network':
                return $this->getNetworkUsage($serviceId);
            default:
                return 0;
        }
    }

    private function getCpuUsage(int $serviceId): float {
        // Implement CPU usage retrieval from monitoring agent
        $metrics = $this->getServiceMetrics($serviceId);
        return $metrics['cpu_percent'] ?? 0;
    }

    private function getMemoryUsage(int $serviceId): float {
        $metrics = $this->getServiceMetrics($serviceId);
        return $metrics['memory_percent'] ?? 0;
    }

    private function getDiskUsage(int $serviceId): float {
        $metrics = $this->getServiceMetrics($serviceId);
        return $metrics['disk_percent'] ?? 0;
    }

    private function getNetworkUsage(int $serviceId): float {
        $metrics = $this->getServiceMetrics($serviceId);
        return $metrics['network_percent'] ?? 0;
    }

    private function getServiceMetrics(int $serviceId): array {
        // Retrieve from monitoring database or API
        return \WHMCS\Database\Capsule::connection()
            ->select("SELECT * FROM mod_monitoring_metrics WHERE service_id = ? ORDER BY created_at DESC LIMIT 1", [$serviceId])[0] ?? [];
    }
}
```

## Database Schema
```sql
CREATE TABLE `mod_autoscale_policies` (
  `id` VARCHAR(50) PRIMARY KEY,
  `service_id` INT NOT NULL,
  `metric_type` ENUM('cpu', 'memory', 'disk', 'network') NOT NULL,
  `scale_up_threshold` INT DEFAULT 80,
  `scale_down_threshold` INT DEFAULT 20,
  `scale_up_action` VARCHAR(50) DEFAULT 'add_resource',
  `scale_down_action` VARCHAR(50) DEFAULT 'remove_resource',
  `scale_increment` INT DEFAULT 1,
  `cooldown_minutes` INT DEFAULT 5,
  `min_instances` INT DEFAULT 1,
  `max_instances` INT DEFAULT 10,
  `enabled` TINYINT(1) DEFAULT 1,
  `created_at` DATETIME NOT NULL,
  UNIQUE KEY `unique_service` (`service_id`)
);

CREATE TABLE `mod_autoscale_history` (
  `id` INT AUTO_INCREMENT PRIMARY KEY,
  `policy_id` VARCHAR(50) NOT NULL,
  `service_id` INT NOT NULL,
  `action` ENUM('scale_up', 'scale_down') NOT NULL,
  `metric_value` FLOAT NOT NULL,
  `previous_resources` TEXT,
  `new_resources` TEXT,
  `created_at` DATETIME NOT NULL,
  INDEX `idx_service_id` (`service_id`)
);
```

## Best Practices

1. **Conservative Thresholds**: Set thresholds to trigger scaling before peak load
2. **Cooldown Periods**: Prevent rapid scaling oscillations
3. **Resource Limits**: Set min/max bounds to control costs
4. **Monitoring**: Track scaling events and outcomes
5. **Testing**: Regularly test scaling behavior under load

## Related Skills

- whmcs-load-balancer-config
- whmcs-monitoring-agent
- whmcs-resource-quotas
- whmcs-cost-tracking