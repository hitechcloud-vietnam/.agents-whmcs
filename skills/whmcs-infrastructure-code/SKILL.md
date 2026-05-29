---
name: whmcs-infrastructure-code
description: IaC patterns for WHMCS
category: Automation & DevOps
version: 1.0.0
---

# WHMCS Infrastructure as Code Skill

## Overview
This skill provides patterns and implementations for managing infrastructure as code (IaC) in WHMCS, including declarative configuration, state management, and version control.

## Implementation Patterns

### Infrastructure Code Manager
```php
<?php
/**
 * WHMCS Infrastructure as Code
 * Manages declarative infrastructure configuration
 */

namespace WHMCS\Module\DevOps\IaC;

class InfrastructureCodeManager {
    private $db;
    private $stateManager;
    private $validator;

    public function __construct() {
        $this->db = \WHMCS\Database\Capsule::connection();
        $this->stateManager = new StateManager();
        $this->validator = new IaCValidator();
    }

    /**
     * Create infrastructure definition
     */
    public function createDefinition(array $params): array {
        $definitionId = 'iad_' . bin2hex(random_bytes(12));

        $definition = [
            'id' => $definitionId,
            'service_id' => $params['service_id'],
            'name' => $params['name'],
            'version' => $params['version'] ?? '1.0',
            'format' => $params['format'] ?? 'yaml', // yaml, json, hcl
            'content' => $params['content'],
            'variables' => json_encode($params['variables'] ?? []),
            'created_at' => date('Y-m-d H:i:s')
        ];

        $this->db->insert('mod_iac_definitions', $definition);

        // Validate definition
        $validation = $this->validator->validate($definition['content'], $definition['format']);

        if (!$validation['valid']) {
            $this->db->update('mod_iac_definitions', [
                'validation_errors' => json_encode($validation['errors'])
            ], ['id' => $definitionId]);
        }

        return [
            'success' => true,
            'definition_id' => $definitionId,
            'validation' => $validation
        ];
    }

    /**
     * Generate infrastructure from template
     */
    public function generateFromTemplate(array $params): array {
        $templateId = $params['template_id'];
        $template = $this->getTemplate($templateId);

        if (!$template) {
            throw new \Exception("Template not found: {$templateId}");
        }

        // Process template with variables
        $content = $this->processTemplate($template['content'], $params['variables'] ?? []);

        // Validate generated content
        $validation = $this->validator->validate($content, $template['format']);

        return [
            'success' => true,
            'content' => $content,
            'validation' => $validation
        ];
    }

    /**
     * Plan infrastructure changes
     */
    public function planChanges(string $definitionId): array {
        $definition = $this->getDefinition($definitionId);

        if (!$definition) {
            throw new \Exception("Definition not found");
        }

        // Get current state
        $currentState = $this->stateManager->getCurrentState($definition['service_id']);

        // Parse new definition
        $newConfig = $this->parseDefinition($definition['content'], $definition['format']);

        // Calculate changes
        $changes = $this->calculateChanges($currentState, $newConfig);

        // Generate plan
        $plan = [
            'definition_id' => $definitionId,
            'service_id' => $definition['service_id'],
            'changes' => $changes,
            'additions' => count($changes['add']),
            'modifications' => count($changes['update']),
            'deletions' => count($changes['delete']),
            'planned_at' => date('Y-m-d H:i:s')
        ];

        // Store plan
        $this->storePlan($plan);

        return $plan;
    }

    /**
     * Apply infrastructure definition
     */
    public function applyDefinition(string $definitionId, bool $dryRun = false): array {
        $definition = $this->getDefinition($definitionId);

        if (!$definition) {
            throw new \Exception("Definition not found");
        }

        // Parse definition
        $config = $this->parseDefinition($definition['content'], $definition['format']);

        if ($dryRun) {
            return [
                'would_create' => count($config['resources']),
                'would_modify' => 0,
                'would_delete' => 0
            ];
        }

        // Apply changes
        $results = [];

        foreach ($config['resources'] as $resource) {
            $result = $this->applyResource($definition['service_id'], $resource);
            $results[] = $result;
        }

        // Update state
        $this->stateManager->updateState($definition['service_id'], $config);

        // Log execution
        $this->logExecution($definitionId, $results);

        return [
            'success' => true,
            'applied' => count($results),
            'results' => $results
        ];
    }

    /**
     * Generate IaC for existing service
     */
    public function generateFromExisting(int $serviceId): array {
        // Get current service configuration
        $serviceConfig = $this->getServiceConfiguration($serviceId);

        // Convert to IaC format
        $iacContent = $this->convertToIaC($serviceConfig);

        $definitionId = 'iad_' . bin2hex(random_bytes(12));

        $this->db->insert('mod_iac_definitions', [
            'id' => $definitionId,
            'service_id' => $serviceId,
            'name' => 'exported-' . date('Ymd'),
            'version' => '1.0',
            'format' => 'yaml',
            'content' => $iacContent,
            'created_at' => date('Y-m-d H:i:s')
        ]);

        return [
            'success' => true,
            'definition_id' => $definitionId,
            'content' => $iacContent
        ];
    }

    // Private helper methods

    private function parseDefinition(string $content, string $format): array {
        switch ($format) {
            case 'yaml':
                return yaml_parse($content);
            case 'json':
                return json_decode($content, true);
            case 'hcl':
                return $this->parseHCL($content);
            default:
                throw new \Exception("Unknown format: {$format}");
        }
    }

    private function calculateChanges(array $currentState, array $newConfig): array {
        $changes = [
            'add' => [],
            'update' => [],
            'delete' => []
        ];

        // Find additions and modifications
        foreach ($newConfig['resources'] as $resource) {
            $id = $resource['id'];

            if (!isset($currentState['resources'][$id])) {
                $changes['add'][] = $resource;
            } elseif ($this->hasChanges($currentState['resources'][$id], $resource)) {
                $changes['update'][] = $resource;
            }
        }

        // Find deletions
        foreach ($currentState['resources'] as $id => $resource) {
            if (!isset($newConfig['resources'][$id])) {
                $changes['delete'][] = $resource;
            }
        }

        return $changes;
    }

    private function hasChanges(array $old, array $new): bool {
        foreach ($new as $key => $value) {
            if (!isset($old[$key]) || $old[$key] !== $value) {
                return true;
            }
        }
        return false;
    }

    private function convertToIaC(array $config): string {
        // Convert service config to IaC format
        return yaml_emit([
            'version' => '1.0',
            'service' => [
                'id' => $config['id'],
                'name' => $config['name'],
                'resources' => $config['resources']
            ]
        ]);
    }
}

/**
 * State Manager
 */
class StateManager {
    public function getCurrentState(int $serviceId): array {
        $state = \WHMCS\Database\Capsule::connection()->select(
            "SELECT state FROM mod_iac_state WHERE service_id = ?",
            [$serviceId]
        );

        if (empty($state)) {
            return ['resources' => []];
        }

        return json_decode($state[0]->state, true) ?? ['resources' => []];
    }

    public function updateState(int $serviceId, array $config): void {
        \WHMCS\Database\Capsule::connection()->update(
            'mod_iac_state',
            ['state' => json_encode($config), 'updated_at' => date('Y-m-d H:i:s')],
            ['service_id' => $serviceId]
        );
    }
}
```

## Database Schema
```sql
CREATE TABLE `mod_iac_definitions` (
  `id` VARCHAR(50) PRIMARY KEY,
  `service_id` INT NOT NULL,
  `name` VARCHAR(255) NOT NULL,
  `version` VARCHAR(20) DEFAULT '1.0',
  `format` ENUM('yaml', 'json', 'hcl') DEFAULT 'yaml',
  `content` TEXT NOT NULL,
  `variables` TEXT,
  `validation_errors` TEXT,
  `created_at' DATETIME NOT NULL,
  INDEX `idx_service_id` (`service_id`)
);

CREATE TABLE `mod_iac_state` (
  `service_id` INT PRIMARY KEY,
  `state` TEXT NOT NULL,
  `updated_at' DATETIME NOT NULL
);

CREATE TABLE `mod_iac_templates` (
  `id` VARCHAR(50) PRIMARY KEY,
  `name` VARCHAR(255) NOT NULL,
  `description' TEXT,
  `format` VARCHAR(20) DEFAULT 'yaml',
  `content' TEXT NOT NULL,
  `variables_schema' TEXT,
  `created_at' DATETIME NOT NULL
);
```

## IaC YAML Template Example
```yaml
version: '1.0'
service:
  name: example-service
  region: us-east-1

resources:
  - type: vm
    name: web-server
    count: 2
    specs:
      cpu: 4
      memory: 8192
      disk: 100

  - type: load_balancer
    name: main-lb
    targets:
      - web-server

  - type: firewall
    name: web-firewall
    rules:
      - port: 443
        protocol: tcp
        source: 0.0.0.0/0
```

## Best Practices

1. **Version Control**: Keep all IaC in version control
2. **Drift Detection**: Regularly check for configuration drift
3. **Validation**: Always validate before applying
4. **Modular Templates**: Create reusable templates
5. **State Management**: Track state to detect changes

## Related Skills

- whmcs-terraform-modules
- whmcs-gitops-workflow
- whmcs-ansible-playbooks
- whmcs-config-management