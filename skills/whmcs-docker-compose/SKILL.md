---
name: whmcs-docker-compose
description: Docker deployment for WHMCS
category: Automation & DevOps
version: 1.0.0
---

# WHMCS Docker Compose Skill

## Overview
This skill provides patterns and implementations for managing Docker Compose deployments in WHMCS for containerized applications.

## Implementation Patterns

### Docker Compose Manager
```php
<?php
/**
 * WHMCS Docker Compose Management
 * Handles Docker Compose deployments
 */

namespace WHMCS\Module\DevOps\Docker;

class DockerComposeManager {
    private $db;

    public function __construct() {
        $this->db = \WHMCS\Database\Capsule::connection();
    }

    /**
     * Create Docker Compose configuration
     */
    public function createConfiguration(array $params): array {
        $configId = 'dc_' . bin2hex(random_bytes(12));

        $config = [
            'id' => $configId,
            'service_id' => $params['service_id'],
            'name' => $params['name'],
            'compose_version' => $params['version'] ?? '3.8',
            'services' => json_encode($params['services']),
            'networks' => json_encode($params['networks'] ?? []),
            'volumes' => json_encode($params['volumes'] ?? []),
            'created_at' => date('Y-m-d H:i:s')
        ];

        $this->db->insert('mod_docker_compose_configs', $config);

        // Generate compose file
        $composeFile = $this->generateComposeFile($config);

        return [
            'success' => true,
            'config_id' => $configId,
            'compose_file' => $composeFile
        ];
    }

    /**
     * Generate WHMCS Docker Compose template
     */
    public function generateWHMCSCompose(): array {
        $compose = <<<YAML
version: '3.8'

services:
  whmcs:
    image: whmcs/whmcs:latest
    container_name: whmcs_app
    depends_on:
      - db
      - redis
    environment:
      - DB_HOST=db
      - DB_PORT=3306
      - DB_DATABASE=whmcs
      - DB_USERNAME=whmcs
      - DB_PASSWORD=\${DB_PASSWORD}
    volumes:
      - whmcs_data:/var/www/html
      - ./config:/var/www/html/config
    networks:
      - whmcs_network
    ports:
      - "8080:80"
    restart: unless-stopped

  db:
    image: mysql:8.0
    container_name: whmcs_db
    environment:
      - MYSQL_ROOT_PASSWORD=\${MYSQL_ROOT_PASSWORD}
      - MYSQL_DATABASE=whmcs
      - MYSQL_USER=whmcs
      - MYSQL_PASSWORD=\${DB_PASSWORD}
    volumes:
      - db_data:/var/lib/mysql
    networks:
      - whmcs_network
    restart: unless-stopped

  redis:
    image: redis:7-alpine
    container_name: whmcs_redis
    networks:
      - whmcs_network
    restart: unless-stopped

networks:
  whmcs_network:
    driver: bridge

volumes:
  whmcs_data:
  db_data:
YAML;

        return [
            'compose_file' => $compose,
            'services' => ['whmcs', 'db', 'redis']
        ];
    }

    /**
     * Deploy Docker Compose stack
     */
    public function deployStack(string $configId, array $options = []): array {
        $config = $this->getConfig($configId);

        if (!$config) {
            throw new \Exception("Config not found");
        }

        $composeFile = $this->generateComposeFile($config);
        $projectDir = $this->getProjectDir($config['service_id']);

        file_put_contents($projectDir . '/docker-compose.yml', $composeFile);

        // Execute docker-compose up
        $command = "cd {$projectDir} && docker-compose up -d";
        if ($options['rebuild'] ?? false) {
            $command .= ' --build';
        }

        $output = shell_exec($command . ' 2>&1');

        $this->logDeployment($configId, $output);

        return [
            'success' => strpos($output, 'done') !== false,
            'containers' => $this->getContainerStatus($projectDir)
        ];
    }

    /**
     * Scale service
     */
    public function scaleService(string $configId, string $service, int $replicas): array {
        $config = $this->getConfig($configId);
        $projectDir = $this->getProjectDir($config['service_id']);

        $command = "cd {$projectDir} && docker-compose up -d --scale {$service}={$replicas}";
        shell_exec($command);

        return [
            'success' => true,
            'service' => $service,
            'replicas' => $replicas
        ];
    }

    /**
     * Get stack status
     */
    public function getStackStatus(int $serviceId): array {
        $projectDir = $this->getProjectDir($serviceId);

        $output = shell_exec("cd {$projectDir} && docker-compose ps 2>&1");

        $containers = [];
        $lines = explode("\n", trim($output));
        array_shift($lines); // Remove header

        foreach ($lines as $line) {
            if (empty(trim($line))) continue;

            $parts = preg_split('/\s{2,}/', trim($line));
            if (count($parts) >= 4) {
                $containers[] = [
                    'name' => $parts[0],
                    'command' => $parts[1],
                    'status' => $parts[2],
                    'ports' => $parts[3] ?? ''
                ];
            }
        }

        return [
            'service_id' => $serviceId,
            'containers' => $containers,
            'total' => count($containers)
        ];
    }

    // Private helper methods

    private function generateComposeFile(array $config): string {
        $version = $config['compose_version'];
        $services = json_decode($config['services'], true);
        $networks = json_decode($config['networks'] ?? '[]', true);
        $volumes = json_decode($config['volumes'] ?? '[]', true);

        $content = "version: '{$version}'\n\nservices:\n";

        foreach ($services as $service) {
            $content .= "  {$service['name']}:\n";
            $content .= "    image: {$service['image']}\n";

            if (!empty($service['ports'])) {
                foreach ($service['ports'] as $port) {
                    $content .= "    ports:\n      - \"{$port}\"\n";
                }
            }

            if (!empty($service['environment'])) {
                $content .= "    environment:\n";
                foreach ($service['environment'] as $env) {
                    $content .= "      - {$env}\n";
                }
            }

            if (!empty($service['volumes'])) {
                $content .= "    volumes:\n";
                foreach ($service['volumes'] as $vol) {
                    $content .= "      - {$vol}\n";
                }
            }

            if (!empty($service['depends_on'])) {
                $content .= "    depends_on:\n";
                foreach ($service['depends_on'] as $dep) {
                    $content .= "      - {$dep}\n";
                }
            }

            $content .= "    restart: unless-stopped\n\n";
        }

        if (!empty($networks)) {
            $content .= "networks:\n";
            foreach ($networks as $network) {
                $content .= "  {$network['name']}:\n";
                if (!empty($network['driver'])) {
                    $content .= "    driver: {$network['driver']}\n";
                }
            }
        }

        if (!empty($volumes)) {
            $content .= "volumes:\n";
            foreach ($volumes as $volume) {
                $content .= "  {$volume['name']}:\n";
            }
        }

        return $content;
    }

    private function getProjectDir(int $serviceId): string {
        return "/var/lib/whmcs/docker/projects/{$serviceId}";
    }

    private function getContainerStatus(string $projectDir): array {
        $output = shell_exec("cd {$projectDir} && docker-compose ps --format json 2>&1");
        return json_decode($output, true) ?? [];
    }
}
```

## Docker Compose Template for WHMCS

```yaml
version: '3.8'

services:
  whmcs:
    image: whmcs/whmcs:8.0
    container_name: whmcs
    environment:
      - DB_HOST=db
      - DB_PASSWORD=${DB_PASSWORD}
    volumes:
      - whmcs_storage:/var/www/html
    networks:
      - whmcs_net

  db:
    image: mysql:8.0
    environment:
      - MYSQL_ROOT_PASSWORD=${MYSQL_ROOT_PASSWORD}
      - MYSQL_DATABASE=whmcs
      - MYSQL_USER=whmcs
      - MYSQL_PASSWORD=${DB_PASSWORD}
    volumes:
      - db_storage:/var/lib/mysql

  redis:
    image: redis:7-alpine

networks:
  whmcs_net:
    driver: bridge

volumes:
  whmcs_storage:
  db_storage:
```

## Best Practices

1. **Use Named Volumes**: Prefer named volumes over bind mounts
2. **Health Checks**: Add health checks for critical services
3. **Resource Limits**: Set CPU and memory limits
4. **Restart Policies**: Configure appropriate restart policies
5. **Secrets Management**: Use Docker secrets for sensitive data

## Related Skills

- whmcs-container-runtime
- whmcs-kubernetes-modules
- whmcs-helm-charts
- whmcs-cicd-integration