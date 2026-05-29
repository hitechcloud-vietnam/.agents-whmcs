---
name: whmcs-container-runtime
description: Container setup for WHMCS services
category: Provisioning & Cloud
version: 1.0.0
---

# WHMCS Container Runtime Setup Skill

## Overview
This skill provides patterns and implementations for setting up container runtimes in WHMCS, including Docker, containerd, and orchestration integration.

## Implementation Patterns

### Container Runtime Manager
```php
<?php
/**
 * WHMCS Container Runtime Setup
 * Manages container infrastructure for hosted services
 */

namespace WHMCS\Module\Server\Containers;

class ContainerRuntimeManager {
    private $db;

    public function __construct() {
        $this->db = \WHMCS\Database\Capsule::connection();
    }

    /**
     * Setup container runtime on service
     */
    public function setupRuntime(array $params): array {
        $runtimeId = 'ctr_' . bin2hex(random_bytes(12));

        $runtime = [
            'id' => $runtimeId,
            'service_id' => $params['service_id'],
            'runtime_type' => $params['runtime_type'] ?? 'docker', // docker, containerd, podman
            'version' => $params['version'] ?? 'latest',
            'storage_driver' => $params['storage_driver'] ?? 'overlay2',
            'log_driver' => $params['log_driver'] ?? 'json-file',
            'registry_mirrors' => json_encode($params['registry_mirrors'] ?? []),
            'insecure_registries' => json_encode($params['insecure_registries'] ?? []),
            'status' => 'installing',
            'created_at' => date('Y-m-d H:i:s')
        ];

        $this->db->insert('mod_container_runtimes', $runtime);

        // Install runtime on VM
        $vmId = $this->getVmId($params['service_id']);
        $this->installRuntimeOnVM($vmId, $runtime);

        // Configure storage and networking
        $this->configureRuntime($vmId, $runtime);

        return [
            'success' => true,
            'runtime_id' => $runtimeId,
            'runtime_type' => $params['runtime_type']
        ];
    }

    /**
     * Create container from image
     */
    public function createContainer(array $params): array {
        $containerId = 'ctr_' . bin2hex(random_bytes(12));

        $container = [
            'id' => $containerId,
            'runtime_id' => $params['runtime_id'],
            'name' => $params['name'],
            'image' => $params['image'],
            'tag' => $params['tag'] ?? 'latest',
            'command' => $params['command'] ?? null,
            'environment' => json_encode($params['environment'] ?? []),
            'ports' => json_encode($params['ports'] ?? []),
            'volumes' => json_encode($params['volumes'] ?? []),
            'network' => $params['network'] ?? 'bridge',
            'status' => 'created',
            'created_at' => date('Y-m-d H:i:s')
        ];

        $this->db->insert('mod_containers', $container);

        // Create container on VM
        $this->createContainerOnHost($container);

        return [
            'success' => true,
            'container_id' => $containerId,
            'name' => $params['name']
        ];
    }

    /**
     * Manage container lifecycle
     */
    public function manageContainer(string $containerId, string $action): array {
        $container = $this->getContainer($containerId);

        if (!$container) {
            throw new \Exception("Container not found: {$containerId}");
        }

        $vmId = $this->getVmIdFromRuntime($container['runtime_id']);

        switch ($action) {
            case 'start':
                $this->startContainer($vmId, $container);
                break;
            case 'stop':
                $this->stopContainer($vmId, $container);
                break;
            case 'restart':
                $this->restartContainer($vmId, $container);
                break;
            case 'pause':
                $this->pauseContainer($vmId, $container);
                break;
            case 'unpause':
                $this->unpauseContainer($vmId, $container);
                break;
            case 'remove':
                $this->removeContainer($vmId, $container);
                break;
        }

        // Update status
        $status = match($action) {
            'start' => 'running',
            'stop' => 'exited',
            'remove' => 'deleted',
            'pause' => 'paused',
            default => $this->getContainerStatus($vmId, $container)
        };

        if ($action !== 'remove') {
            $this->db->update('mod_containers', ['status' => $status], ['id' => $containerId]);
        }

        return [
            'success' => true,
            'container_id' => $containerId,
            'action' => $action,
            'status' => $status
        ];
    }

    /**
     * Setup container registry
     */
    public function configureRegistry(array $params): array {
        $registryId = 'reg_' . bin2hex(random_bytes(8));

        $registry = [
            'id' => $registryId,
            'runtime_id' => $params['runtime_id'],
            'name' => $params['name'],
            'url' => $params['url'],
            'username' => $params['username'] ?? null,
            'password_env' => $params['password_env'] ?? 'REGISTRY_PASSWORD',
            'insecure' => $params['insecure'] ?? false,
            'created_at' => date('Y-m-d H:i:s')
        ];

        $this->db->insert('mod_container_registries', $registry);

        // Configure on VM
        $runtime = $this->getRuntime($params['runtime_id']);
        $vmId = $this->getVmIdFromRuntime($params['runtime_id']);
        $this->configureRegistryOnHost($vmId, $registry);

        return [
            'success' => true,
            'registry_id' => $registryId,
            'name' => $params['name']
        ];
    }

    /**
     * Setup container networking
     */
    public function createNetwork(array $params): array {
        $networkId = 'net_' . bin2hex(random_bytes(8));

        $network = [
            'id' => $networkId,
            'runtime_id' => $params['runtime_id'],
            'name' => $params['name'],
            'driver' => $params['driver'] ?? 'bridge',
            'subnet' => $params['subnet'] ?? null,
            'gateway' => $params['gateway'] ?? null,
            'created_at' => date('Y-m-d H:i:s')
        ];

        $this->db->insert('mod_container_networks', $network);

        // Create network on host
        $vmId = $this->getVmIdFromRuntime($params['runtime_id']);
        $this->createNetworkOnHost($vmId, $network);

        return [
            'success' => true,
            'network_id' => $networkId,
            'name' => $params['name']
        ];
    }

    /**
     * Create Docker Compose stack
     */
    public function createComposeStack(array $params): array {
        $stackId = 'stk_' . bin2hex(random_bytes(8));

        $stack = [
            'id' => $stackId,
            'runtime_id' => $params['runtime_id'],
            'name' => $params['name'],
            'compose_file' => $params['compose_file'],
            'environment_file' => $params['environment_file'] ?? null,
            'status' => 'deploying',
            'created_at' => date('Y-m-d H:i:s')
        ];

        $this->db->insert('mod_compose_stacks', $stack);

        // Deploy stack
        $vmId = $this->getVmIdFromRuntime($params['runtime_id']);
        $this->deployComposeStack($vmId, $stack);

        return [
            'success' => true,
            'stack_id' => $stackId,
            'name' => $params['name']
        ];
    }

    /**
     * Get container logs
     */
    public function getContainerLogs(string $containerId, int $lines = 100): array {
        $container = $this->getContainer($containerId);
        $vmId = $this->getVmIdFromRuntime($container['runtime_id']);

        $command = "docker logs --tail {$lines} {$container['name']} 2>&1";
        $output = $this->executeOnVM($vmId, $command);

        return [
            'container_id' => $containerId,
            'logs' => $output
        ];
    }

    /**
     * Get container stats
     */
    public function getContainerStats(string $containerId): array {
        $container = $this->getContainer($containerId);
        $vmId = $this->getVmIdFromRuntime($container['runtime_id']);

        $command = "docker stats --no-stream --format json {$container['name']}";
        $output = $this->executeOnVM($vmId, $command);

        $stats = json_decode($output, true);

        return [
            'container_id' => $containerId,
            'cpu_percent' => $stats['CPUPerc'] ?? '0%',
            'memory_usage' => $stats['MemUsage'] ?? '0B / 0B',
            'memory_percent' => $stats['MemPerc'] ?? '0%',
            'network_io' => $stats['NetIO'] ?? '0B / 0B',
            'block_io' => $stats['BlockIO'] ?? '0B / 0B'
        ];
    }

    // Private helper methods

    private function installRuntimeOnVM(string $vmId, array $runtime): void {
        $installScript = match($runtime['runtime_type']) {
            'docker' => $this->getDockerInstallScript($runtime),
            'containerd' => $this->getContainerdInstallScript($runtime),
            'podman' => $this->getPodmanInstallScript($runtime),
            default => throw new \Exception("Unsupported runtime: {$runtime['runtime_type']}")
        };

        $this->executeOnVM($vmId, $installScript);
    }

    private function getDockerInstallScript(array $runtime): string {
        return <<<SCRIPT
#!/bin/bash
set -e

# Install prerequisites
apt-get update
apt-get install -y apt-transport-https ca-certificates curl gnupg lsb-release

# Add Docker GPG key
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg

# Add Docker repository
echo "deb [arch=amd64 signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable" | tee /etc/apt/sources.list.d/docker.list > /dev/null

# Install Docker
apt-get update
apt-get install -y docker-ce docker-ce-cli containerd.io docker-compose-plugin

# Configure Docker daemon
mkdir -p /etc/docker
cat > /etc/docker/daemon.json <<EOF
{
  "storage-driver": "{$runtime['storage_driver']}",
  "log-driver": "{$runtime['log_driver']}",
  "log-opts": {
    "max-size": "10m",
    "max-file": "3"
  },
  "registry-mirrors": [],
  "insecure-registries": []
}
EOF

# Restart Docker
systemctl enable docker
systemctl start docker

# Add current user to docker group
usermod -aG docker ubuntu
SCRIPT;
    }

    private function createContainerOnHost(array $container): void {
        $ports = json_decode($container['ports'], true) ?: [];
        $volumes = json_decode($container['volumes'], true) ?: [];
        $env = json_decode($container['environment'], true) ?: [];

        $portMapping = '';
        foreach ($ports as $port) {
            $portMapping .= " -p {$port}";
        }

        $volumeMapping = '';
        foreach ($volumes as $volume) {
            $volumeMapping .= " -v {$volume}";
        }

        $envVars = '';
        foreach ($env as $key => $value) {
            $envVars .= " -e {$key}='{$value}'";
        }

        $command = $container['command'] ? " -c '{$container['command']}'" : '';

        $createCmd = "docker create{$portMapping}{$volumeMapping}{$envVars} {$container['image']}:{$container['tag']}{$command} {$container['name']}";

        $vmId = $this->getVmIdFromRuntime($container['runtime_id']);
        $this->executeOnVM($vmId, $createCmd);
    }
}
```

## Database Schema
```sql
CREATE TABLE `mod_container_runtimes` (
  `id` VARCHAR(50) PRIMARY KEY,
  `service_id` INT NOT NULL,
  `runtime_type` ENUM('docker', 'containerd', 'podman') DEFAULT 'docker',
  `version` VARCHAR(20),
  `storage_driver` VARCHAR(50) DEFAULT 'overlay2',
  `log_driver` VARCHAR(50) DEFAULT 'json-file',
  `registry_mirrors` TEXT,
  `insecure_registries` TEXT,
  `status` ENUM('installing', 'ready', 'error') DEFAULT 'installing',
  `created_at` DATETIME NOT NULL
);

CREATE TABLE `mod_containers` (
  `id` VARCHAR(50) PRIMARY KEY,
  `runtime_id` VARCHAR(50) NOT NULL,
  `name` VARCHAR(255) NOT NULL,
  `image` VARCHAR(500) NOT NULL,
  `tag` VARCHAR(100) DEFAULT 'latest',
  `command` TEXT,
  `environment` TEXT,
  `ports` TEXT,
  `volumes` TEXT,
  `network` VARCHAR(100) DEFAULT 'bridge',
  `status` ENUM('created', 'running', 'paused', 'exited', 'deleted') DEFAULT 'created',
  `created_at` DATETIME NOT NULL,
  FOREIGN KEY (`runtime_id`) REFERENCES `mod_container_runtimes`(`id`)
);

CREATE TABLE `mod_container_registries` (
  `id` VARCHAR(50) PRIMARY KEY,
  `runtime_id` VARCHAR(50) NOT NULL,
  `name` VARCHAR(255) NOT NULL,
  `url` VARCHAR(500) NOT NULL,
  `username` VARCHAR(255),
  `password_env` VARCHAR(100),
  `insecure` TINYINT(1) DEFAULT 0,
  `created_at` DATETIME NOT NULL
);

CREATE TABLE `mod_container_networks` (
  `id` VARCHAR(50) PRIMARY KEY,
  `runtime_id` VARCHAR(50) NOT NULL,
  `name` VARCHAR(255) NOT NULL,
  `driver` VARCHAR(50) DEFAULT 'bridge',
  `subnet` VARCHAR(50),
  `gateway` VARCHAR(50),
  `created_at` DATETIME NOT NULL
);

CREATE TABLE `mod_compose_stacks` (
  `id` VARCHAR(50) PRIMARY KEY,
  `runtime_id` VARCHAR(50) NOT NULL,
  `name` VARCHAR(255) NOT NULL,
  `compose_file` TEXT NOT NULL,
  `environment_file` TEXT,
  `status` ENUM('deploying', 'running', 'error', 'stopped') DEFAULT 'deploying',
  `created_at` DATETIME NOT NULL
);
```

## Docker Compose Template Example
```yaml
version: '3.8'
services:
  web:
    image: nginx:latest
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./data:/var/www/html
    networks:
      - frontend
    restart: unless-stopped

  app:
    build: ./app
    depends_on:
      - db
    environment:
      - DB_HOST=db
    networks:
      - frontend
      - backend

  db:
    image: mysql:8.0
    volumes:
      - db_data:/var/lib/mysql
    environment:
      - MYSQL_ROOT_PASSWORD=${DB_PASSWORD}
    networks:
      - backend

networks:
  frontend:
    driver: bridge
  backend:
    driver: bridge

volumes:
  db_data:
```

## Best Practices

1. **Image Management**: Use specific tags, not 'latest'
2. **Resource Limits**: Set CPU and memory limits for containers
3. **Networking**: Use custom networks for service isolation
4. **Logging**: Configure appropriate log drivers and rotation
5. **Security**: Scan images for vulnerabilities regularly

## Related Skills

- whmcs-docker-compose
- whmcs-kubernetes-cluster
- whmcs-helm-charts
- whmcs-monitoring-agent