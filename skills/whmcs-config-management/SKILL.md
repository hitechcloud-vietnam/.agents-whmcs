---
name: whmcs-config-management
description: Config management for WHMCS
category: Automation & DevOps
version: 1.0.0
---

# WHMCS Configuration Management Skill

## Overview
This skill provides patterns for managing configuration in WHMCS.

## Implementation Patterns

### Configuration Manager
```php
<?php
/**
 * WHMCS Configuration Management
 * Manages system configuration
 */

namespace WHMCS\Module\DevOps\Config;

class ConfigurationManager {
    private $db;

    public function __construct() {
        $this->db = \WHMCS\Database\Capsule::connection();
    }

    /**
     * Store configuration
     */
    public function setConfig(string $key, $value, array $options = []): void {
        $this->db->update('mod_config', [
            'value' => json_encode($value),
            'encrypted' => $options['encrypted'] ?? false,
            'updated_at' => date('Y-m-d H:i:s')
        ], ['key' => $key]);
    }

    /**
     * Get configuration
     */
    public function getConfig(string $key, $default = null) {
        $config = $this->db->select(
            "SELECT * FROM mod_config WHERE `key` = ?",
            [$key]
        )[0];

        if (!$config) {
            return $default;
        }

        return $config->encrypted
            ? $this->decrypt($config->value)
            : json_decode($config->value, true);
    }

    /**
     * Export configuration
     */
    public function exportConfig(string $format = 'json'): string {
        $configs = $this->db->select(
            "SELECT `key`, value FROM mod_config WHERE encrypted = 0"
        );

        $data = [];
        foreach ($configs as $config) {
            $data[$config->key] = json_decode($config->value, true);
        }

        return match($format) {
            'yaml' => yaml_emit($data),
            default => json_encode($data, JSON_PRETTY_PRINT)
        };
    }
}
```

## Database Schema
```sql
CREATE TABLE `mod_config` (
  `id` INT AUTO_INCREMENT PRIMARY KEY,
  `key` VARCHAR(100) NOT NULL UNIQUE,
  `value` TEXT NOT NULL,
  `encrypted` TINYINT(1) DEFAULT 0,
  `updated_at` DATETIME,
  `created_at' DATETIME NOT NULL
);
```

## Best Practices

1. **Version Control**: Store config in version control
2. **Encryption**: Encrypt sensitive values
3. **Environment Separation**: Different configs per env
4. **Validation**: Validate config values
5. **Documentation**: Document config options

## Related Skills

- whmcs-gitops-workflow
- whmcs-secrets-management
- whmcs-infrastructure-code
- whmcs-ansible-playbooks