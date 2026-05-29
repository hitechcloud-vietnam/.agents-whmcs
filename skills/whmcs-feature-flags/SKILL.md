---
name: whmcs-feature-flags
description: Feature toggle system for WHMCS
category: Automation & DevOps
version: 1.0.0
---

# WHMCS Feature Flags Skill

## Overview
This skill provides patterns and implementations for managing feature flags in WHMCS to control feature rollouts and A/B testing.

## Implementation Patterns

### Feature Flag Manager
```php
<?php
/**
 * WHMCS Feature Flags
 * Manages feature toggles
 */

namespace WHMCS\Module\DevOps\Features;

class FeatureFlagManager {
    private $db;
    private $cache = [];

    public function __construct() {
        $this->db = \WHMCS\Database\Capsule::connection();
    }

    /**
     * Create feature flag
     */
    public function createFlag(array $params): array {
        $flagId = 'flag_' . bin2hex(random_bytes(8));

        $flag = [
            'id' => $flagId,
            'name' => $params['name'],
            'description' => $params['description'] ?? '',
            'default_value' => $params['enabled'] ?? false,
            'rollout_percentage' => $params['rollout'] ?? 0,
            'target_groups' => json_encode($params['target_groups'] ?? []),
            'created_at' => date('Y-m-d H:i:s')
        ];

        $this->db->insert('mod_feature_flags', $flag);

        return [
            'success' => true,
            'flag_id' => $flagId
        ];
    }

    /**
     * Check if feature is enabled
     */
    public function isEnabled(string $flagName, int $userId = null): bool {
        // Check cache first
        if (isset($this->cache[$flagName])) {
            return $this->cache[$flagName];
        }

        $flag = $this->db->select(
            "SELECT * FROM mod_feature_flags WHERE name = ?",
            [$flagName]
        )[0];

        if (!$flag) {
            return false;
        }

        // Check user-specific override
        if ($userId) {
            $override = $this->getUserOverride($flag->id, $userId);
            if ($override !== null) {
                return (bool) $override;
            }
        }

        // Check rollout percentage
        if ($flag->rollout_percentage < 100) {
            $bucket = crc32($userId ?? uniqid()) % 100;
            if ($bucket >= $flag->rollout_percentage) {
                return false;
            }
        }

        return (bool) $flag->default_value;
    }

    /**
     * Enable feature for user
     */
    public function enableForUser(string $flagId, int $userId): bool {
        $this->db->delete('mod_feature_flag_overrides', [
            'flag_id' => $flagId,
            'user_id' => $userId
        ]);

        $this->db->insert('mod_feature_flag_overrides', [
            'flag_id' => $flagId,
            'user_id' => $userId,
            'enabled' => 1,
            'created_at' => date('Y-m-d H:i:s')
        ]);

        // Clear cache
        unset($this->cache[$flagId]);

        return true;
    }

    /**
     * Gradual rollout
     */
    public function gradualRollout(string $flagId, int $percentage): array {
        $this->db->update('mod_feature_flags', [
            'rollout_percentage' => $percentage,
            'updated_at' => date('Y-m-d H:i:s')
        ], ['id' => $flagId]);

        return [
            'success' => true,
            'flag_id' => $flagId,
            'rollout_percentage' => $percentage
        ];
    }
}
```

## Database Schema
```sql
CREATE TABLE `mod_feature_flags` (
  `id` VARCHAR(50) PRIMARY KEY,
  `name` VARCHAR(100) NOT NULL UNIQUE,
  `description' TEXT,
  `default_value` TINYINT(1) DEFAULT 0,
  `rollout_percentage` INT DEFAULT 0,
  `target_groups` TEXT,
  `created_at' DATETIME NOT NULL,
  `updated_at' DATETIME
);

CREATE TABLE `mod_feature_flag_overrides` (
  `id` INT AUTO_INCREMENT PRIMARY KEY,
  `flag_id` VARCHAR(50) NOT NULL,
  `user_id` INT NOT NULL,
  `enabled` TINYINT(1) NOT NULL,
  `created_at' DATETIME NOT NULL,
  UNIQUE KEY `unique_user_flag` (`flag_id`, `user_id`),
  FOREIGN KEY (`flag_id`) REFERENCES `mod_feature_flags`(`id`)
);
```

## Best Practices

1. **Clear Naming**: Use descriptive flag names
2. **Documentation**: Document purpose and expected duration
3. **Cleanup**: Remove flags when features are stable
4. **Gradual Rollout**: Start with small percentage
5. **Override Capability**: Allow enabling for specific users

## Related Skills

- whmcs-canary-releases
- whmcs-gitops-workflow
- whmcs-cicd-integration
- whmcs-blue-green-deploy