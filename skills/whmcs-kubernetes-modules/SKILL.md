---
name: whmcs-kubernetes-modules
description: K8s module deployment for WHMCS
category: Automation & DevOps
version: 1.0.0
---

# WHMCS Kubernetes Modules Skill

## Overview
This skill provides patterns and implementations for deploying and managing Kubernetes modules (Helm charts, operators) in WHMCS.

## Implementation Patterns

### Kubernetes Module Manager
```php
<?php
/**
 * WHMCS Kubernetes Module Deployment
 * Manages K8s module lifecycle
 */

namespace WHMCS\Module\DevOps\Kubernetes;

class KubernetesModuleManager {
    private $db;
    private $kubectl;

    public function __construct() {
        $this->db = \WHMCS\Database\Capsule::connection();
        $this->kubectl = new KubectlWrapper();
    }

    /**
     * Deploy Helm chart
     */
    public function deployHelmChart(array $params): array {
        $deploymentId = 'helm_' . bin2hex(random_bytes(8));

        $deployment = [
            'id' => $deploymentId,
            'service_id' => $params['service_id'],
            'release_name' => $params['release_name'],
            'chart' => $params['chart'],
            'version' => $params['version'] ?? 'latest',
            'namespace' => $params['namespace'] ?? 'default',
            'values_file' => $params['values_file'] ?? null,
            'status' => 'deploying',
            'created_at' => date('Y-m-d H:i:s')
        ];

        $this->db->insert('mod_k8s_helm_deployments', $deployment);

        // Execute helm install
        $result = $this->executeHelmInstall($deployment);

        $this->db->update('mod_k8s_helm_deployments', [
            'status' => $result['success'] ? 'deployed' : 'failed',
            'deployed_at' => date('Y-m-d H:i:s')
        ], ['id' => $deploymentId]);

        return [
            'success' => $result['success'],
            'deployment_id' => $deploymentId,
            'release' => $params['release_name']
        ];
    }

    /**
     * Install Kubernetes operator
     */
    public function installOperator(array $params): array {
        $operatorId = 'kop_' . bin2hex(random_bytes(8));

        $operator = [
            'id' => $operatorId,
            'service_id' => $params['service_id'],
            'name' => $params['name'],
            'crd_file' => $params['crd_file'],
            'namespace' => $params['namespace'] ?? 'operators',
            'created_at' => date('Y-m-d H:i:s')
        ];

        $this->db->insert('mod_k8s_operators', $operator);

        // Apply CRD
        $this->kubectl->apply($this->getClusterConfig($params['service_id']), [
            'apiVersion' => 'apiextensions.k8s.io/v1',
            'kind' => 'CustomResourceDefinition',
            'metadata' => ['name' => $params['name']],
            'spec' => json_decode($params['crd_spec'], true)
        ]);

        return [
            'success' => true,
            'operator_id' => $operatorId
        ];
    }

    /**
     * Manage Helm releases
     */
    public function listReleases(int $serviceId): array {
        $cluster = $this->getClusterForService($serviceId);

        $output = shell_exec("helm list --all --namespace default --kubeconfig /path/to/kubeconfig.json");

        return $this->parseHelmList($output);
    }

    /**
     * Upgrade Helm release
     */
    public function upgradeRelease(string $deploymentId, array $params): array {
        $deployment = $this->getHelmDeployment($deploymentId);

        if (!$deployment) {
            throw new \Exception("Deployment not found");
        }

        $result = $this->executeHelmUpgrade($deployment, $params);

        $this->db->update('mod_k8s_helm_deployments', [
            'version' => $params['version'] ?? $deployment['version'],
            'upgraded_at' => date('Y-m-d H:i:s')
        ], ['id' => $deploymentId]);

        return [
            'success' => $result['success'],
            'deployment_id' => $deploymentId
        ];
    }

    /**
     * Rollback Helm release
     */
    public function rollbackRelease(string $deploymentId, int $revision = null): array {
        $deployment = $this->getHelmDeployment($deploymentId);

        $command = $revision
            ? "helm rollback {$deployment['release_name']} {$revision}"
            : "helm rollback {$deployment['release_name']}";

        exec($command, $output, $return);

        return [
            'success' => $return === 0,
            'deployment_id' => $deploymentId
        ];
    }

    // Private helper methods

    private function executeHelmInstall(array $deployment): array {
        $command = "helm install {$deployment['release_name']} {$deployment['chart']}";
        $command .= " --namespace {$deployment['namespace']}";

        if ($deployment['values_file']) {
            $command .= " --values {$deployment['values_file']}";
        }

        $output = shell_exec($command . ' 2>&1');

        return [
            'success' => strpos($output, 'deployed') !== false,
            'output' => $output
        ];
    }

    private function executeHelmUpgrade(array $deployment, array $params): array {
        $command = "helm upgrade {$deployment['release_name']} {$params['chart'] ?? $deployment['chart']}";

        if ($params['values']) {
            foreach ($params['values'] as $key => $value) {
                $command .= " --set {$key}={$value}";
            }
        }

        $output = shell_exec($command . ' 2>&1');

        return [
            'success' => strpos($output, 'upgraded') !== false,
            'output' => $output
        ];
    }

    private function parseHelmList(string $output): array {
        $releases = [];
        $lines = explode("\n", trim($output));
        array_shift($lines); // Remove header

        foreach ($lines as $line) {
            $parts = preg_split('/\s{2,}/', trim($line));
            if (count($parts) >= 6) {
                $releases[] = [
                    'name' => $parts[0],
                    'namespace' => $parts[1],
                    'revision' => $parts[2],
                    'status' => $parts[3],
                    'chart' => $parts[4],
                    'app_version' => $parts[5] ?? ''
                ];
            }
        }

        return $releases;
    }
}
```

## Database Schema
```sql
CREATE TABLE `mod_k8s_helm_deployments` (
  `id` VARCHAR(50) PRIMARY KEY,
  `service_id` INT NOT NULL,
  `release_name` VARCHAR(255) NOT NULL,
  `chart` VARCHAR(500) NOT NULL,
  `version` VARCHAR(50),
  `namespace` VARCHAR(100) DEFAULT 'default',
  `values_file` VARCHAR(500),
  `status` ENUM('deploying', 'deployed', 'failed', 'upgrading') DEFAULT 'deploying',
  `deployed_at` DATETIME,
  `upgraded_at` DATETIME,
  `created_at' DATETIME NOT NULL
);

CREATE TABLE `mod_k8s_operators` (
  `id` VARCHAR(50) PRIMARY KEY,
  `service_id` INT NOT NULL,
  `name` VARCHAR(255) NOT NULL,
  `crd_file` TEXT,
  `namespace` VARCHAR(100) DEFAULT 'operators',
  `created_at' DATETIME NOT NULL
);
```

## Best Practices

1. **Use Helm Repositories**: Use official Helm charts when available
2. **Version Pinning**: Pin chart versions for reproducibility
3. **Namespace Isolation**: Deploy to dedicated namespaces
4. **RBAC**: Configure proper RBAC for Helm
5. **Value Management**: Use values files for configuration

## Related Skills

- whmcs-helm-charts
- whmcs-kubernetes-cluster
- whmcs-docker-compose
- whmcs-canary-releases