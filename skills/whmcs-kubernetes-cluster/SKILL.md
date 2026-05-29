---
name: whmcs-kubernetes-cluster
description: K8s cluster management for WHMCS
category: Provisioning & Cloud
version: 1.0.0
---

# WHMCS Kubernetes Cluster Management Skill

## Overview
This skill provides patterns and implementations for managing Kubernetes clusters in WHMCS, including cluster provisioning, workload deployment, scaling, and monitoring.

## Implementation Patterns

### Kubernetes Cluster Manager
```php
<?php
/**
 * WHMCS Kubernetes Cluster Management
 * Handles K8s cluster lifecycle for hosted services
 */

namespace WHMCS\Module\Server\Kubernetes;

class KubernetesClusterManager {
    private $db;
    private $kubectl;

    public function __construct() {
        $this->db = \WHMCS\Database\Capsule::connection();
        $this->kubectl = new KubectlWrapper();
    }

    /**
     * Create Kubernetes cluster
     */
    public function createCluster(array $params): array {
        $clusterId = 'k8s_' . bin2hex(random_bytes(12));

        $cluster = [
            'id' => $clusterId,
            'service_id' => $params['service_id'],
            'name' => $params['name'],
            'version' => $params['version'] ?? '1.28',
            'pod_cidr' => $params['pod_cidr'] ?? '10.244.0.0/16',
            'service_cidr' => $params['service_cidr'] ?? '10.96.0.0/12',
            'node_count' => $params['node_count'] ?? 3,
            'master_type' => $params['master_type'] ?? 'managed',
            'node_type' => $params['node_type'] ?? 'standard',
            'status' => 'provisioning',
            'created_at' => date('Y-m-d H:i:s')
        ];

        $this->db->insert('mod_k8s_clusters', $cluster);

        // Provision cluster based on type
        $provisionResult = $this->provisionCluster($cluster);

        // Update status
        $this->db->update('mod_k8s_clusters', [
            'endpoint' => $provisionResult['endpoint'],
            'ca_cert' => $provisionResult['ca_cert'],
            'status' => 'active',
            'ready_at' => date('Y-m-d H:i:s')
        ], ['id' => $clusterId]);

        // Configure kubectl access
        $this->configureKubectl($clusterId);

        return [
            'success' => true,
            'cluster_id' => $clusterId,
            'endpoint' => $provisionResult['endpoint'],
            'kubeconfig' => $this->generateKubeconfig($clusterId)
        ];
    }

    /**
     * Deploy workload to cluster
     */
    public function deployWorkload(array $params): array {
        $cluster = $this->getCluster($params['cluster_id']);

        if (!$cluster) {
            throw new \Exception("Cluster not found: {$params['cluster_id']}");
        }

        $deploymentId = 'dep_' . bin2hex(random_bytes(8));

        $deployment = [
            'id' => $deploymentId,
            'cluster_id' => $params['cluster_id'],
            'namespace' => $params['namespace'] ?? 'default',
            'name' => $params['name'],
            'image' => $params['image'],
            'replicas' => $params['replicas'] ?? 1,
            'cpu_request' => $params['cpu_request'] ?? '100m',
            'cpu_limit' => $params['cpu_limit'] ?? '500m',
            'memory_request' => $params['memory_request'] ?? '128Mi',
            'memory_limit' => $params['memory_limit'] ?? '512Mi',
            'status' => 'deploying',
            'created_at' => date('Y-m-d H:i:s')
        ];

        $this->db->insert('mod_k8s_deployments', $deployment);

        // Apply to cluster
        $this->applyDeployment($cluster, $deployment);

        $this->db->update('mod_k8s_deployments', [
            'status' => 'running'
        ], ['id' => $deploymentId]);

        return [
            'success' => true,
            'deployment_id' => $deploymentId,
            'namespace' => $params['namespace'] ?? 'default',
            'name' => $params['name']
        ];
    }

    /**
     * Scale deployment
     */
    public function scaleDeployment(string $deploymentId, int $replicas): array {
        $deployment = $this->getDeployment($deploymentId);

        if (!$deployment) {
            throw new \Exception("Deployment not found: {$deploymentId}");
        }

        $cluster = $this->getCluster($deployment['cluster_id']);

        // Scale via kubectl
        $this->kubectl->scale($cluster, $deployment['namespace'], $deployment['name'], $replicas);

        // Update database
        $this->db->update('mod_k8s_deployments', [
            'replicas' => $replicas,
            'scaled_at' => date('Y-m-d H:i:s')
        ], ['id' => $deploymentId]);

        return [
            'success' => true,
            'deployment_id' => $deploymentId,
            'replicas' => $replicas
        ];
    }

    /**
     * Create service
     */
    public function createService(array $params): array {
        $cluster = $this->getCluster($params['cluster_id']);

        $service = [
            'id' => 'svc_' . bin2hex(random_bytes(8)),
            'cluster_id' => $params['cluster_id'],
            'namespace' => $params['namespace'] ?? 'default',
            'name' => $params['name'],
            'type' => $params['type'] ?? 'ClusterIP', // ClusterIP, NodePort, LoadBalancer
            'selector' => json_encode($params['selector'] ?? []),
            'ports' => json_encode($params['ports'] ?? []),
            'created_at' => date('Y-m-d H:i:s')
        ];

        $this->db->insert('mod_k8s_services', $service);

        // Apply to cluster
        $this->applyService($cluster, $service);

        return [
            'success' => true,
            'service_id' => $service['id'],
            'name' => $params['name'],
            'type' => $params['type'] ?? 'ClusterIP'
        ];
    }

    /**
     * Create ingress
     */
    public function createIngress(array $params): array {
        $cluster = $this->getCluster($params['cluster_id']);

        $ingress = [
            'id' => 'ing_' . bin2hex(random_bytes(8)),
            'cluster_id' => $params['cluster_id'],
            'namespace' => $params['namespace'] ?? 'default',
            'name' => $params['name'],
            'host' => $params['host'],
            'tls_enabled' => $params['tls'] ?? true,
            'tls_secret' => $params['tls_secret'] ?? null,
            'rules' => json_encode($params['rules'] ?? []),
            'annotations' => json_encode($params['annotations'] ?? []),
            'created_at' => date('Y-m-d H:i:s')
        ];

        $this->db->insert('mod_k8s_ingresses', $ingress);

        // Apply to cluster
        $this->applyIngress($cluster, $ingress);

        return [
            'success' => true,
            'ingress_id' => $ingress['id'],
            'host' => $params['host']
        ];
    }

    /**
     * Get cluster status
     */
    public function getClusterStatus(string $clusterId): array {
        $cluster = $this->getCluster($clusterId);

        if (!$cluster) {
            throw new \Exception("Cluster not found: {$clusterId}");
        }

        // Get node status
        $nodes = $this->kubectl->getNodes($cluster);
        $pods = $this->kubectl->getPods($cluster, 'kube-system');

        return [
            'cluster_id' => $clusterId,
            'name' => $cluster['name'],
            'version' => $cluster['version'],
            'status' => $cluster['status'],
            'nodes' => array_map(fn($n) => [
                'name' => $n['name'],
                'status' => $n['status'],
                'roles' => $n['roles'],
                'cpu' => $n['cpu'],
                'memory' => $n['memory']
            ], $nodes),
            'system_pods_running' => count(array_filter($pods, fn($p) => $p['phase'] === 'Running')),
            'endpoint' => $cluster['endpoint']
        ];
    }

    /**
     * Delete cluster
     */
    public function deleteCluster(string $clusterId): bool {
        $cluster = $this->getCluster($clusterId);

        if (!$cluster) {
            return false;
        }

        // Delete cluster resources
        $this->deleteClusterResources($cluster);

        // Update status
        $this->db->update('mod_k8s_clusters', [
            'status' => 'deleted',
            'deleted_at' => date('Y-m-d H:i:s')
        ], ['id' => $clusterId]);

        return true;
    }

    // Private helper methods

    private function provisionCluster(array $cluster): array {
        // Provision based on cluster type
        switch ($cluster['master_type']) {
            case 'managed':
                return $this->provisionManagedCluster($cluster);
            case 'selfmanaged':
                return $this->provisionSelfManagedCluster($cluster);
            default:
                throw new \Exception("Unknown cluster type");
        }
    }

    private function provisionManagedCluster(array $cluster): array {
        // For managed K8s (EKS, GKE, AKS)
        $endpoint = "https://{$cluster['name']}.k8s.region.provider.com";
        $caCert = base64_encode($this->generateCACert());

        return [
            'endpoint' => $endpoint,
            'ca_cert' => $caCert
        ];
    }

    private function provisionSelfManagedCluster(array $cluster): array {
        // For self-managed clusters using kubeadm
        $vmId = $this->getVmId($cluster['service_id']);

        // Initialize master node
        $initScript = <<<SCRIPT
#!/bin/bash
set -e

# Install Kubernetes
apt-get update
apt-get install -y apt-transport-https curl

curl -fsSL https://packages.cloud.google.com/apt/doc/apt-key.gpg | gpg --dearmor -o /etc/apt/keyrings/kubernetes-archive-keyring.gpg
echo "deb [signed-by=/etc/apt/keyrings/kubernetes-archive-keyring.gpg] https://apt.kubernetes.io kubernetes-xenial main" > /etc/apt/sources.list.d/kubernetes.list

apt-get update
apt-get install -y kubelet kubeadm kubectl
apt-mark hold kubelet kubeadm kubectl

# Initialize cluster
kubeadm init --pod-network-cidr={$cluster['pod_cidr']} --service-cidr={$cluster['service_cidr']}

# Setup kubectl
mkdir -p $HOME/.kube
cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
chown $(id -u):$(id -g) $HOME/.kube/config

# Install network plugin (Calico)
kubectl apply -f https://docs.projectcalico.org/manifests/calico.yaml
SCRIPT;

        $this->executeOnVM($vmId, $initScript);

        return [
            'endpoint' => 'https://localhost:6443',
            'ca_cert' => base64_encode(file_get_contents('/etc/kubernetes/pki/ca.crt'))
        ];
    }

    private function applyDeployment(array $cluster, array $deployment): void {
        $manifest = [
            'apiVersion' => 'apps/v1',
            'kind' => 'Deployment',
            'metadata' => [
                'name' => $deployment['name'],
                'namespace' => $deployment['namespace']
            ],
            'spec' => [
                'replicas' => $deployment['replicas'],
                'selector' => [
                    'matchLabels' => ['app' => $deployment['name']]
                ],
                'template' => [
                    'metadata' => [
                        'labels' => ['app' => $deployment['name']]
                    ],
                    'spec' => [
                        'containers' => [
                            [
                                'name' => $deployment['name'],
                                'image' => $deployment['image'],
                                'ports' => [['containerPort' => 8080]],
                                'resources' => [
                                    'requests' => [
                                        'cpu' => $deployment['cpu_request'],
                                        'memory' => $deployment['memory_request']
                                    ],
                                    'limits' => [
                                        'cpu' => $deployment['cpu_limit'],
                                        'memory' => $deployment['memory_limit']
                                    ]
                                ]
                            ]
                        ]
                    ]
                ]
            ]
        ];

        $this->kubectl->apply($cluster, $manifest);
    }
}

/**
 * Kubectl Wrapper
 */
class KubectlWrapper {
    public function apply(array $cluster, array $manifest): void {
        $yaml = yaml_emit($manifest);
        $file = tempnam(sys_get_temp_dir(), 'k8s_');
        file_put_contents($file, $yaml);

        $cmd = "kubectl apply -f {$file} --server={$cluster['endpoint']} --certificate-authority=" . base64_decode($cluster['ca_cert']);
        exec($cmd, $output, $return);

        unlink($file);

        if ($return !== 0) {
            throw new \Exception("kubectl apply failed: " . implode("\n", $output));
        }
    }

    public function scale(array $cluster, string $namespace, string $name, int $replicas): void {
        $cmd = "kubectl scale deployment {$name} -n {$namespace} --replicas={$replicas}";
        exec($cmd);
    }

    public function getNodes(array $cluster): array {
        $output = shell_exec("kubectl get nodes -o json");
        $data = json_decode($output, true);

        return array_map(function($node) {
            return [
                'name' => $node['metadata']['name'],
                'status' => $node['status']['conditions'][0]['status'] === 'True' ? 'Ready' : 'NotReady',
                'roles' => array_keys(array_filter($node['metadata']['labels'] ?? [], fn($v, $k) => strpos($k, 'node-role') !== false)),
                'cpu' => $node['status']['capacity']['cpu'],
                'memory' => $node['status']['capacity']['memory']
            ];
        }, $data['items'] ?? []);
    }

    public function getPods(array $cluster, string $namespace): array {
        $output = shell_exec("kubectl get pods -n {$namespace} -o json");
        $data = json_decode($output, true);

        return array_map(fn($pod) => [
            'name' => $pod['metadata']['name'],
            'phase' => $pod['status']['phase']
        ], $data['items'] ?? []);
    }
}
```

## Database Schema
```sql
CREATE TABLE `mod_k8s_clusters` (
  `id` VARCHAR(50) PRIMARY KEY,
  `service_id` INT NOT NULL,
  `name` VARCHAR(255) NOT NULL,
  `version` VARCHAR(20) DEFAULT '1.28',
  `endpoint` VARCHAR(500),
  `ca_cert` TEXT,
  `pod_cidr` VARCHAR(50),
  `service_cidr` VARCHAR(50),
  `node_count` INT DEFAULT 3,
  `master_type` VARCHAR(50) DEFAULT 'managed',
  `node_type` VARCHAR(50),
  `kubeconfig' TEXT,
  `status` ENUM('provisioning', 'active', 'error', 'deleted') DEFAULT 'provisioning',
  'ready_at' DATETIME,
  'created_at' DATETIME NOT NULL,
  INDEX `idx_service_id` (`service_id`)
);

CREATE TABLE `mod_k8s_deployments` (
  `id` VARCHAR(50) PRIMARY KEY,
  `cluster_id` VARCHAR(50) NOT NULL,
  `namespace` VARCHAR(100) DEFAULT 'default',
  `name` VARCHAR(255) NOT NULL,
  `image` VARCHAR(500) NOT NULL,
  `replicas` INT DEFAULT 1,
  `cpu_request` VARCHAR(20),
  `cpu_limit` VARCHAR(20),
  `memory_request` VARCHAR(20),
  `memory_limit` VARCHAR(20),
  `status` ENUM('deploying', 'running', 'scaling', 'error') DEFAULT 'deploying',
  `scaled_at` DATETIME,
  `created_at` DATETIME NOT NULL
);

CREATE TABLE `mod_k8s_services` (
  `id` VARCHAR(50) PRIMARY KEY,
  `cluster_id` VARCHAR(50) NOT NULL,
  `namespace` VARCHAR(100) DEFAULT 'default',
  `name` VARCHAR(255) NOT NULL,
  `type` ENUM('ClusterIP', 'NodePort', 'LoadBalancer') DEFAULT 'ClusterIP',
  `selector` TEXT,
  `ports` TEXT,
  `created_at` DATETIME NOT NULL
);

CREATE TABLE `mod_k8s_ingresses` (
  `id` VARCHAR(50) PRIMARY KEY,
  `cluster_id` VARCHAR(50) NOT NULL,
  `namespace` VARCHAR(100) DEFAULT 'default',
  `name` VARCHAR(255) NOT NULL,
  `host` VARCHAR(255) NOT NULL,
  `tls_enabled` TINYINT(1) DEFAULT 1,
  `tls_secret` VARCHAR(100),
  `rules` TEXT,
  `annotations` TEXT,
  `created_at` DATETIME NOT NULL
);
```

## Best Practices

1. **Namespace Isolation**: Use namespaces for service separation
2. **Resource Limits**: Always set CPU and memory requests/limits
3. **High Availability**: Use multiple replicas and poddisruptionbudgets
4. **Health Checks**: Configure readiness and liveness probes
5. **Logging**: Aggregate logs from all pods to central logging

## Related Skills

- whmcs-container-runtime
- whmcs-helm-charts
- whmcs-monitoring-agent
- whmcs-auto-scaling