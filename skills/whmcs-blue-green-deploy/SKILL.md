---
name: whmcs-blue-green-deploy
description: Blue-green deployment for WHMCS
category: Automation & DevOps
version: 1.0.0
---

# WHMCS Blue-Green Deployment Skill

## Overview
This skill provides patterns and implementations for managing blue-green deployments in WHMCS for zero-downtime releases.

## Implementation Patterns

### Blue-Green Deployment Manager
```php
<?php
/**
 * WHMCS Blue-Green Deployment
 * Handles blue-green deployment strategy
 */

namespace WHMCS\Module\DevOps\Deployment;

class BlueGreenDeploymentManager {
    private $db;

    public function __construct() {
        $this->db = \WHMCS\Database\Capsule::connection();
    }

    /**
     * Setup blue-green environment
     */
    public function setupEnvironment(array $params): array {
        $envId = 'bg_' . bin2hex(random_bytes(8));

        $environment = [
            'id' => $envId,
            'service_id' => $params['service_id'],
            'blue_version' => $params['blue_version'],
            'green_version' => $params['green_version'] ?? null,
            'active_color' => 'blue',
            'created_at' => date('Y-m-d H:i:s')
        ];

        $this->db->insert('mod_bluegreen_environments', $environment);

        return [
            'success' => true,
            'environment_id' => $envId
        ];
    }

    /**
     * Switch traffic
     */
    public function switchTraffic(string $envId, string $toColor): array {
        $env = $this->getEnvironment($envId);

        $fromColor = $env['active_color'];

        if ($toColor === $fromColor) {
            return ['success' => true, 'message' => 'Already on this color'];
        }

        // Deploy to new environment
        $this->deployToEnvironment($envId, $toColor);

        // Switch load balancer
        $this->switchLoadBalancer($envId, $toColor);

        // Update active color
        $this->db->update('mod_bluegreen_environments', [
            'active_color' => $toColor,
            'last_switch_at' => date('Y-m-d H:i:s'),
            'last_switch_from' => $fromColor
        ], ['id' => $envId]);

        return [
            'success' => true,
            'from_color' => $fromColor,
            'to_color' => $toColor
        ];
    }

    /**
     * Rollback to previous version
     */
    public function rollback(string $envId): array {
        $env = $this->getEnvironment($envId);

        $currentColor = $env['active_color'];
        $previousColor = $currentColor === 'blue' ? 'green' : 'blue';

        return $this->switchTraffic($envId, $previousColor);
    }
}
```

## Database Schema
```sql
CREATE TABLE `mod_bluegreen_environments` (
  `id` VARCHAR(50) PRIMARY KEY,
  `service_id` INT NOT NULL,
  `blue_version` VARCHAR(50),
  `green_version` VARCHAR(50),
  `active_color` ENUM('blue', 'green') DEFAULT 'blue',
  `last_switch_at` DATETIME,
  `last_switch_from` VARCHAR(10),
  `created_at' DATETIME NOT NULL
);
```

## Best Practices

1. **Identical Environments**: Keep blue and green identical
2. **Test Before Switch**: Test new environment before switching
3. **Database Compatibility**: Ensure no DB migration issues
4. **Quick Rollback**: Always have quick rollback path
5. **Health Checks**: Verify health after traffic switch

## Related Skills

- whmcs-canary-releases
- whmcs-load-balancer-config
- whmcs-gitops-workflow
- whmcs-cicd-integration