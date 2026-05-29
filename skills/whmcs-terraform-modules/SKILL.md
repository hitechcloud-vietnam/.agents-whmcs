---
name: whmcs-terraform-modules
description: Terraform setup for WHMCS
category: Automation & DevOps
version: 1.0.0
---

# WHMCS Terraform Modules Skill

## Overview
This skill provides patterns and implementations for creating and managing Terraform modules for WHMCS infrastructure provisioning.

## Implementation Patterns

### Terraform Module Manager
```php
<?php
/**
 * WHMCS Terraform Modules
 * Manages Terraform module lifecycle
 */

namespace WHMCS\Module\DevOps\Terraform;

class TerraformModuleManager {
    private $db;
    private $terraformExecutor;

    public function __construct() {
        $this->db = \WHMCS\Database\Capsule::connection();
        $this->terraformExecutor = new TerraformExecutor();
    }

    /**
     * Create Terraform module
     */
    public function createModule(array $params): array {
        $moduleId = 'tfmod_' . bin2hex(random_bytes(12));

        $module = [
            'id' => $moduleId,
            'service_id' => $params['service_id'],
            'name' => $params['name'],
            'source' => $params['source'],
            'version' => $params['version'] ?? '1.0.0',
            'variables' => json_encode($params['variables'] ?? []),
            'outputs' => json_encode($params['outputs'] ?? []),
            'created_at' => date('Y-m-d H:i:s')
        ];

        $this->db->insert('mod_terraform_modules', $module);

        return [
            'success' => true,
            'module_id' => $moduleId
        ];
    }

    /**
     * Generate Terraform configuration
     */
    public function generateConfig(int $serviceId, array $params): array {
        $service = $this->getService($serviceId);

        $config = <<<TERRAFORM
# Terraform Configuration for WHMCS Service
# Generated: {$service->domain}

terraform {
  required_version = ">= 1.0"
  backend "remote" {
    organization = "whmcs"
    workspaces {
      name = "service-{$serviceId}"
    }
  }
}

provider "aws" {
  region = var.region
}

variable "region" {
  description = "AWS region"
  type        = string
  default     = "us-east-1"
}

variable "instance_type" {
  description = "EC2 instance type"
  type        = string
  default     = "t3.medium"
}

variable "ami_id" {
  description = "AMI ID for instances"
  type        = string
  default     = ""
}

# Main configuration
resource "aws_instance" "whmcs_server" {
  count         = var.instance_count
  ami           = var.ami_id
  instance_type = var.instance_type

  tags = {
    Name        = "whmcs-service-{$serviceId}"
    Environment = var.environment
    ServiceId   = "{$serviceId}"
  }
}

output "instance_ips" {
  value = aws_instance.whmcs_server[*].private_ip
}
TERRAFORM;

        return [
            'success' => true,
            'config' => $config,
            'required_variables' => ['region', 'instance_type', 'ami_id', 'instance_count']
        ];
    }

    /**
     * Initialize Terraform workspace
     */
    public function initializeWorkspace(array $params): array {
        $workspaceId = 'ws_' . bin2hex(random_bytes(8));

        $this->db->insert('mod_terraform_workspaces', [
            'id' => $workspaceId,
            'service_id' => $params['service_id'],
            'name' => $params['name'],
            'config' => $params['config'],
            'status' => 'initializing',
            'created_at' => date('Y-m-d H:i:s')
        ]);

        // Initialize terraform
        $result = $this->terraformExecutor->init($params['config']);

        $this->db->update('mod_terraform_workspaces', [
            'status' => 'ready',
            'initialized_at' => date('Y-m-d H:i:s')
        ], ['id' => $workspaceId]);

        return [
            'success' => true,
            'workspace_id' => $workspaceId,
            'init_result' => $result
        ];
    }

    /**
     * Plan Terraform changes
     */
    public function plan(string $workspaceId): array {
        $workspace = $this->getWorkspace($workspaceId);

        if (!$workspace) {
            throw new \Exception("Workspace not found");
        }

        $result = $this->terraformExecutor->plan($workspace['config']);

        return [
            'success' => true,
            'changes' => $result['changes'],
            'add' => $result['add'],
            'change' => $result['change'],
            'destroy' => $result['destroy']
        ];
    }

    /**
     * Apply Terraform configuration
     */
    public function apply(string $workspaceId): array {
        $workspace = $this->getWorkspace($workspaceId);

        $result = $this->terraformExecutor->apply($workspace['config']);

        $this->db->update('mod_terraform_workspaces', [
            'last_applied_at' => date('Y-m-d H:i:s'),
            'last_apply_hash' => $result['hash']
        ], ['id' => $workspaceId]);

        // Log the apply
        $this->logApply($workspaceId, $result);

        return [
            'success' => true,
            'resources_created' => $result['add'],
            'resources_modified' => $result['change']
        ];
    }

    /**
     * Destroy Terraform resources
     */
    public function destroy(string $workspaceId): array {
        $workspace = $this->getWorkspace($workspaceId);

        $result = $this->terraformExecutor->destroy($workspace['config']);

        $this->db->update('mod_terraform_workspaces', [
            'status' => 'destroyed',
            'destroyed_at' => date('Y-m-d H:i:s')
        ], ['id' => $workspaceId]);

        return [
            'success' => true,
            'resources_destroyed' => $result['destroy']
        ];
    }
}

/**
 * Terraform Executor
 */
class TerraformExecutor {
    private $binaryPath = '/usr/local/bin/terraform';
    private $workDir = '/var/lib/whmcs/terraform';

    public function init(array $config): array {
        $workDir = $this->workDir . '/' . uniqid();
        mkdir($workDir, 0755, true);

        // Write configuration
        file_put_contents($workDir . '/main.tf', $config['main']);

        // Run terraform init
        $output = shell_exec("cd {$workDir} && {$this->binaryPath} init -upgrade 2>&1");

        return ['success' => true, 'output' => $output];
    }

    public function plan(array $config): array {
        $workDir = $this->findWorkDir($config['workspace_id']);

        $output = shell_exec("cd {$workDir} && {$this->binaryPath} plan -out=tfplan 2>&1");

        return $this->parsePlanOutput($output);
    }

    public function apply(array $config): array {
        $workDir = $this->findWorkDir($config['workspace_id']);

        $output = shell_exec("cd {$workDir} && {$this->binaryPath} apply -auto-approve tfplan 2>&1");

        return [
            'add' => 1,
            'change' => 0,
            'destroy' => 0,
            'hash' => md5($output)
        ];
    }

    public function destroy(array $config): array {
        $workDir = $this->findWorkDir($config['workspace_id']);

        $output = shell_exec("cd {$workDir} && {$this->binaryPath} destroy -auto-approve 2>&1");

        return ['destroy' => 1, 'output' => $output];
    }
}
```

## Database Schema
```sql
CREATE TABLE `mod_terraform_modules` (
  `id` VARCHAR(50) PRIMARY KEY,
  `service_id` INT NOT NULL,
  `name` VARCHAR(255) NOT NULL,
  `source` VARCHAR(500) NOT NULL,
  `version` VARCHAR(50) DEFAULT '1.0.0',
  `variables` TEXT,
  `outputs` TEXT,
  `created_at' DATETIME NOT NULL
);

CREATE TABLE `mod_terraform_workspaces` (
  `id` VARCHAR(50) PRIMARY KEY,
  `service_id` INT NOT NULL,
  `name` VARCHAR(255) NOT NULL,
  `config' TEXT,
  `status' ENUM('initializing', 'ready', 'applied', 'destroyed', 'error') DEFAULT 'initializing',
  `last_applied_at' DATETIME,
  'last_apply_hash' VARCHAR(64),
  'destroyed_at' DATETIME,
  'created_at' DATETIME NOT NULL
);
```

## Best Practices

1. **Remote State**: Use remote backend for state management
2. **Workspace Isolation**: Separate workspaces per environment
3. **Variable Files**: Use .tfvars for sensitive data
4. **Module Versioning**: Pin module versions for stability
5. **State Locking**: Enable state locking to prevent conflicts

## Related Skills

- whmcs-infrastructure-code
- whmcs-ansible-playbooks
- whmcs-config-management
- whmcs-gitops-workflow