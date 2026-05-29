---
name: whmcs-disaster-recovery-vm
description: VM disaster recovery planning for WHMCS
category: Provisioning & Cloud
version: 1.0.0
---

# WHMCS Disaster Recovery VM Skill

## Overview
This skill provides patterns and implementations for disaster recovery planning in WHMCS, including recovery procedures, failover systems, backup verification, and business continuity workflows.

## Implementation Patterns

### Disaster Recovery Manager
```php
<?php
/**
 * WHMCS Disaster Recovery Management
 * Handles VM failover and disaster recovery operations
 */

namespace WHMCS\Module\Server\DR;

class DisasterRecoveryManager {
    private $db;
    private $alertManager;
    private $recoveryProcedures = [];

    public function __construct() {
        $this->db = \WHMCS\Database\Capsule::connection();
        $this->alertManager = new AlertManager();
        $this->initializeRecoveryProcedures();
    }

    /**
     * Initialize disaster recovery procedures
     */
    private function initializeRecoveryProcedures(): void {
        $this->recoveryProcedures = [
            'vm_failure' => new VMFailureRecovery(),
            'hypervisor_failure' => new HypervisorFailureRecovery(),
            'network_failure' => new NetworkFailureRecovery(),
            'storage_failure' => new StorageFailureRecovery(),
            'data_corruption' => new DataCorruptionRecovery()
        ];
    }

    /**
     * Create disaster recovery plan for a service
     */
    public function createRecoveryPlan(int $serviceId, array $params): array {
        $this->validateRecoveryPlanParams($params);

        $planId = 'dr_' . bin2hex(random_bytes(12));

        $plan = [
            'id' => $planId,
            'service_id' => $serviceId,
            'rto_minutes' => $params['rto_minutes'] ?? 60,
            'rpo_minutes' => $params['rpo_minutes'] ?? 15,
            'recovery_strategy' => $params['strategy'] ?? 'backup_restore',
            'failover_region' => $params['failover_region'] ?? null,
            'backup_frequency' => $params['backup_frequency'] ?? 'daily',
            'health_check_interval' => $params['health_check_interval'] ?? 300,
            'auto_failover_enabled' => $params['auto_failover'] ?? false,
            'created_at' => date('Y-m-d H:i:s')
        ];

        $this->db->insert('mod_dr_plans', $plan);

        // Create associated backup schedule
        $this->createBackupScheduleForPlan($planId, $serviceId, $params);

        // Setup health monitoring
        $this->setupHealthMonitoring($planId, $serviceId);

        return [
            'success' => true,
            'plan_id' => $planId,
            'rto_minutes' => $plan['rto_minutes'],
            'rpo_minutes' => $plan['rpo_minutes']
        ];
    }

    /**
     * Execute disaster recovery failover
     */
    public function executeFailover(string $planId, array $options = []): array {
        $plan = $this->getRecoveryPlan($planId);

        if (!$plan) {
            throw new \Exception("Recovery plan not found: {$planId}");
        }

        $startTime = microtime(true);

        // Log start of recovery
        $this->logRecoveryEvent($planId, 'failover_started', [
            'reason' => $options['reason'] ?? 'manual_trigger',
            'triggered_by' => $options['triggered_by'] ?? 'system'
        ]);

        // Notify stakeholders
        $this->notifyRecoveryStart($plan);

        // Execute recovery procedure based on strategy
        try {
            $recoveryResult = match ($plan['recovery_strategy']) {
                'backup_restore' => $this->executeBackupRestoreRecovery($plan),
                'vm_replication' => $this->executeVMReplicationRecovery($plan),
                'snapshot_restore' => $this->executeSnapshotRestoreRecovery($plan),
                default => throw new \Exception("Unknown recovery strategy")
            };

            $success = true;
            $errorMessage = null;
        } catch (\Exception $e) {
            $success = false;
            $errorMessage = $e->getMessage();
            $recoveryResult = null;
        }

        $endTime = microtime(true);
        $durationSeconds = round($endTime - $startTime);

        // Update plan with recovery results
        $this->db->update('mod_dr_plans', [
            'last_recovery_at' => date('Y-m-d H:i:s'),
            'last_recovery_duration' => $durationSeconds,
            'last_recovery_status' => $success ? 'success' : 'failed'
        ], ['id' => $planId]);

        // Log completion
        $this->logRecoveryEvent($planId, 'failover_completed', [
            'success' => $success,
            'duration_seconds' => $durationSeconds,
            'error' => $errorMessage
        ]);

        // Notify stakeholders of completion
        $this->notifyRecoveryComplete($plan, $success, $durationSeconds);

        return [
            'success' => $success,
            'plan_id' => $planId,
            'duration_seconds' => $durationSeconds,
            'result' => $recoveryResult,
            'error' => $errorMessage
        ];
    }

    /**
     * Backup restore recovery strategy
     */
    private function executeBackupRestoreRecovery(array $plan): array {
        $serviceId = $plan['service_id'];

        // Get latest successful backup
        $backup = $this->getLatestBackup($serviceId);

        if (!$backup) {
            throw new \Exception("No valid backup found for recovery");
        }

        // Get target hypervisor
        $targetHypervisor = $this->getFailoverHypervisor($plan);

        // Create new VM on target
        $vmId = $this->createFailoverVM($plan, $targetHypervisor);

        // Download and restore backup
        $localPath = $this->downloadBackup($backup);
        $this->restoreBackupToVM($vmId, $localPath);

        // Configure networking
        $this->configureFailoverNetworking($vmId, $plan);

        // Start VM
        $this->startVM($vmId);

        // Update DNS if auto-failover
        if ($plan['auto_failover_enabled']) {
            $this->updateDNSForFailover($plan);
        }

        return [
            'vm_id' => $vmId,
            'backup_id' => $backup['id'],
            'hypervisor' => $targetHypervisor
        ];
    }

    /**
     * Health check and monitoring
     */
    public function performHealthCheck(int $serviceId): array {
        $checks = [];

        // VM connectivity check
        $checks['vm_reachable'] = $this->checkVMReachability($serviceId);

        // Disk health
        $checks['disk_healthy'] = $this->checkDiskHealth($serviceId);

        // Memory usage
        $checks['memory_normal'] = $this->checkMemoryUsage($serviceId);

        // Backup availability
        $checks['backup_available'] = $this->checkBackupAvailability($serviceId);

        // Recent backups verified
        $checks['recent_backup_verified'] = $this->checkRecentBackupVerified($serviceId);

        $overallHealth = array_reduce($checks, fn($carry, $item) => $carry && $item, true);

        $result = [
            'service_id' => $serviceId,
            'healthy' => $overallHealth,
            'checks' => $checks,
            'checked_at' => date('Y-m-d H:i:s')
        ];

        // Update health status in database
        $this->db->update('mod_dr_plans', [
            'health_status' => $overallHealth ? 'healthy' : 'degraded',
            'last_health_check' => date('Y-m-d H:i:s')
        ], ['service_id' => $serviceId]);

        // Alert if unhealthy
        if (!$overallHealth) {
            $this->alertManager->sendHealthAlert($serviceId, $checks);
        }

        return $result;
    }

    /**
     * List all recovery plans
     */
    public function listRecoveryPlans(array $filters = []): array {
        $query = "SELECT p.*, h.domain as service_name
                  FROM mod_dr_plans p
                  JOIN tblhosting h ON p.service_id = h.id
                  WHERE 1=1";

        $bindings = [];

        if (!empty($filters['status'])) {
            $query .= " AND p.health_status = ?";
            $bindings[] = $filters['status'];
        }

        if (!empty($filters['auto_failover'])) {
            $query .= " AND p.auto_failover_enabled = ?";
            $bindings[] = $filters['auto_failover'];
        }

        $plans = $this->db->select($query, $bindings);

        return array_map(function($plan) {
            return [
                'id' => $plan->id,
                'service_id' => $plan->service_id,
                'service_name' => $plan->service_name,
                'rto_minutes' => $plan->rto_minutes,
                'rpo_minutes' => $plan->rpo_minutes,
                'recovery_strategy' => $plan->recovery_strategy,
                'health_status' => $plan->health_status,
                'auto_failover' => (bool) $plan->auto_failover_enabled,
                'last_recovery_at' => $plan->last_recovery_at,
                'created_at' => $plan->created_at
            ];
        }, $plans);
    }

    /**
     * Get recovery timeline/report
     */
    public function getRecoveryReport(string $planId, \DateTime $from, \DateTime $to): array {
        $events = $this->db->select(
            "SELECT * FROM mod_dr_events
             WHERE plan_id = ? AND created_at BETWEEN ? AND ?
             ORDER BY created_at DESC",
            [$planId, $from->format('Y-m-d H:i:s'), $to->format('Y-m-d H:i:s')]
        );

        $plan = $this->getRecoveryPlan($planId);

        return [
            'plan_id' => $planId,
            'period' => [
                'from' => $from->format('Y-m-d'),
                'to' => $to->format('Y-m-d')
            ],
            'rto_target_minutes' => $plan['rto_minutes'],
            'rpo_target_minutes' => $plan['rpo_minutes'],
            'events' => array_map(function($event) {
                return [
                    'type' => $event->event_type,
                    'details' => json_decode($event->event_data, true),
                    'timestamp' => $event->created_at
                ];
            }, $events),
            'summary' => $this->calculateRecoveryMetrics($events, $plan)
        ];
    }

    // Private helper methods

    private function getLatestBackup(int $serviceId): ?array {
        $backup = $this->db->select(
            "SELECT * FROM mod_service_backups
             WHERE service_id = ? AND status = 'completed'
             ORDER BY created_at DESC LIMIT 1",
            [$serviceId]
        );

        return $backup[0] ?? null;
    }

    private function getFailoverHypervisor(array $plan): string {
        // Return failover hypervisor based on plan configuration
        return $plan['failover_region'] ?? 'backup-hypervisor-1';
    }

    private function checkVMReachability(int $serviceId): bool {
        $vmDetails = $this->getVmDetails($serviceId);
        $ip = $vmDetails['ip_address'] ?? '';

        $pingResult = exec("ping -c 3 -W 5 {$ip} 2>&1 | grep -c '3 packets'");
        return (int) $pingResult === 1;
    }

    private function checkDiskHealth(int $serviceId): bool {
        // Implementation for disk health check
        return true;
    }

    private function checkMemoryUsage(int $serviceId): bool {
        $vmDetails = $this->getVmDetails($serviceId);
        // Check memory usage threshold
        return true;
    }

    private function checkBackupAvailability(int $serviceId): bool {
        $backup = $this->getLatestBackup($serviceId);
        return $backup !== null;
    }

    private function checkRecentBackupVerified(int $serviceId): bool {
        $recentBackup = $this->db->select(
            "SELECT * FROM mod_service_backups
             WHERE service_id = ? AND status = 'verified'
             AND created_at > DATE_SUB(NOW(), INTERVAL 1 DAY)",
            [$serviceId]
        );

        return !empty($recentBackup);
    }

    private function logRecoveryEvent(string $planId, string $eventType, array $eventData): void {
        $this->db->insert('mod_dr_events', [
            'plan_id' => $planId,
            'event_type' => $eventType,
            'event_data' => json_encode($eventData),
            'created_at' => date('Y-m-d H:i:s')
        ]);
    }

    private function calculateRecoveryMetrics(array $events, array $plan): array {
        $failovers = array_filter($events, fn($e) => $e->event_type === 'failover_completed');

        $totalDuration = 0;
        $successCount = 0;

        foreach ($failovers as $event) {
            $data = json_decode($event->event_data, true);
            $totalDuration += $data['duration_seconds'] ?? 0;
            if ($data['success'] ?? false) {
                $successCount++;
            }
        }

        return [
            'total_recoveries' => count($failovers),
            'successful_recoveries' => $successCount,
            'failed_recoveries' => count($failovers) - $successCount,
            'average_recovery_time' => count($failovers) > 0 ? $totalDuration / count($failovers) : 0,
            'met_rto' => $successCount > 0 && ($totalDuration / $successCount) <= ($plan['rto_minutes'] * 60)
        ];
    }
}

/**
 * Alert Manager for DR events
 */
class AlertManager {
    public function sendHealthAlert(int $serviceId, array $checks): void {
        $message = "DR Health Check Failed for Service #{$serviceId}\n";
        $message .= "Failed checks: " . implode(', ', array_keys(array_filter($checks, fn($v) => !$v)));

        // Send alerts via configured channels
        $this->sendEmailAlert($serviceId, $message);
        $this->sendWebhookAlert($serviceId, $checks);
    }

    private function sendEmailAlert(int $serviceId, string $message): void {
        // Implementation for email alerts
    }

    private function sendWebhookAlert(int $serviceId, array $checks): void {
        // Implementation for webhook alerts
    }
}
```

## Database Schema
```sql
CREATE TABLE `mod_dr_plans` (
  `id` VARCHAR(50) PRIMARY KEY,
  `service_id` INT NOT NULL,
  `rto_minutes` INT DEFAULT 60,
  `rpo_minutes` INT DEFAULT 15,
  `recovery_strategy` VARCHAR(50) DEFAULT 'backup_restore',
  `failover_region` VARCHAR(100),
  `backup_frequency` VARCHAR(20) DEFAULT 'daily',
  `health_check_interval` INT DEFAULT 300,
  `auto_failover_enabled` TINYINT(1) DEFAULT 0,
  `health_status` ENUM('healthy', 'degraded', 'critical') DEFAULT 'healthy',
  `last_health_check` DATETIME,
  `last_recovery_at` DATETIME,
  `last_recovery_duration` INT,
  `last_recovery_status` VARCHAR(20),
  `created_at` DATETIME NOT NULL,
  UNIQUE KEY `unique_service` (`service_id`)
);

CREATE TABLE `mod_dr_events` (
  `id` INT AUTO_INCREMENT PRIMARY KEY,
  `plan_id` VARCHAR(50) NOT NULL,
  `event_type` VARCHAR(100) NOT NULL,
  `event_data` TEXT,
  `created_at` DATETIME NOT NULL,
  INDEX `idx_plan_id` (`plan_id`)
);
```

## Failover Workflow

```
┌─────────────────────────────────────────────────────────────────┐
│                    Disaster Recovery Workflow                     │
├─────────────────────────────────────────────────────────────────┤
│  1. Health Check (every 5 minutes)                             │
│     └─> Check VM reachability, disk health, backup availability │
│                                                                  │
│  2. Alert Trigger (if health check fails)                        │
│     └─> Send alerts to administrators                           │
│                                                                  │
│  3. Auto/Manual Failover                                        │
│     ├─> If auto_failover: Start recovery automatically          │
│     └─> If manual: Wait for admin confirmation                  │
│                                                                  │
│  4. Recovery Execution                                          │
│     ├─> Provision new VM on failover hypervisor                │
│     ├─> Restore from latest verified backup                     │
│     ├─> Configure networking                                   │
│     └─> Start services                                          │
│                                                                  │
│  5. DNS Update (if configured)                                  │
│     └─> Point domain to new VM IP                               │
│                                                                  │
│  6. Post-Recovery Verification                                   │
│     └─> Run health checks on new VM                            │
└─────────────────────────────────────────────────────────────────┘
```

## Best Practices

1. **Regular Testing**: Test failover procedures quarterly
2. **Documentation**: Keep runbooks updated with current procedures
3. **Monitoring**: Implement comprehensive health monitoring
4. **Communication**: Establish clear escalation paths
5. **RTO/RPO Planning**: Set realistic recovery objectives

## Related Skills

- whmcs-backup-scheduling
- whmcs-snapshot-management
- whmcs-monitoring-agent
- whmcs-incident-response