---
name: whmcs-canary-releases
description: Canary deployment for WHMCS
category: Automation & DevOps
version: 1.0.0
---

# WHMCS Canary Releases Skill

## Overview
This skill provides patterns and implementations for managing canary deployments in WHMCS to safely rollout new features.

## Implementation Patterns

### Canary Release Manager
```php
<?php
/**
 * WHMCS Canary Release Management
 * Handles canary deployment strategy
 */

namespace WHMCS\Module\DevOps\Deployment;

class CanaryReleaseManager {
    private $db;

    public function __construct() {
        $this->db = \WHMCS\Database\Capsule::connection();
    }

    /**
     * Start canary release
     */
    public function startCanary(array $params): array {
        $canaryId = 'canary_' . bin2hex(random_bytes(8));

        $canary = [
            'id' => $canaryId,
            'service_id' => $params['service_id'],
            'version' => $params['version'],
            'canary_percentage' => $params['percentage'] ?? 10,
            'metrics' => json_encode($params['metrics'] ?? []),
            'status' => 'deploying',
            'created_at' => date('Y-m-d H:i:s')
        ];

        $this->db->insert('mod_canary_releases', $canary);

        // Deploy canary instances
        $this->deployCanaryInstances($canaryId, $params);

        return [
            'success' => true,
            'canary_id' => $canaryId
        ];
    }

    /**
     * Monitor canary metrics
     */
    public function monitorCanary(string $canaryId): array {
        $canary = $this->getCanary($canaryId);

        if (!$canary) {
            throw new \Exception("Canary release not found");
        }

        // Collect metrics
        $metrics = $this->collectMetrics($canaryId);

        // Evaluate health
        $health = $this->evaluateHealth($metrics);

        // Log metrics
        $this->logMetrics($canaryId, $metrics);

        return [
            'canary_id' => $canaryId,
            'metrics' => $metrics,
            'health' => $health
        ];
    }

    /**
     * Promote canary
     */
    public function promoteCanary(string $canaryId): array {
        $canary = $this->getCanary($canaryId);

        // Full deployment to all instances
        $this->fullDeploy($canary['service_id'], $canary['version']);

        $this->db->update('mod_canary_releases', [
            'status' => 'promoted',
            'promoted_at' => date('Y-m-d H:i:s')
        ], ['id' => $canaryId]);

        return [
            'success' => true,
            'canary_id' => $canaryId
        ];
    }

    /**
     * Rollback canary
     */
    public function rollbackCanary(string $canaryId): array {
        $canary = $this->getCanary($canaryId);

        // Remove canary instances
        $this->removeCanaryInstances($canaryId);

        $this->db->update('mod_canary_releases', [
            'status' => 'rolled_back',
            'rolled_back_at' => date('Y-m-d H:i:s')
        ], ['id' => $canaryId]);

        return [
            'success' => true,
            'canary_id' => $canaryId
        ];
    }
}
```

## Database Schema
```sql
CREATE TABLE `mod_canary_releases` (
  `id` VARCHAR(50) PRIMARY KEY,
  `service_id` INT NOT NULL,
  `version` VARCHAR(50) NOT NULL,
  `canary_percentage` INT DEFAULT 10,
  `metrics` TEXT,
  `status' ENUM('deploying', 'running', 'promoted', 'rolled_back') DEFAULT 'deploying',
  `promoted_at' DATETIME,
  `rolled_back_at' DATETIME,
  `created_at' DATETIME NOT NULL
);
```

## Best Practices

1. **Start Small**: Begin with 5-10% traffic
2. **Monitor Closely**: Watch error rates and latency
3. **Set Thresholds**: Define success criteria upfront
4. **Quick Rollback**: Be ready to rollback fast
5. **Gradual Increase**: Incrementally increase traffic

## Related Skills

- whmcs-blue-green-deploy
- whmcs-feature-flags
- whmcs-monitoring-agent
- whmcs-incident-response