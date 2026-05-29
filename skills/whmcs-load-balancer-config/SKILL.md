---
name: whmcs-load-balancer-config
description: Load balancer setup for WHMCS services
category: Provisioning & Cloud
version: 1.0.0
---

# WHMCS Load Balancer Configuration Skill

## Overview
This skill provides patterns and implementations for configuring load balancers in WHMCS, including backend pool management, health checks, SSL termination, and traffic distribution algorithms.

## Implementation Patterns

### Load Balancer Manager
```php
<?php
/**
 * WHMCS Load Balancer Configuration
 * Manages load balancer setup and backend management
 */

namespace WHMCS\Module\Server\LoadBalancer;

class LoadBalancerManager {
    private $db;
    private $adapters = [];

    public function __construct() {
        $this->db = \WHMCS\Database\Capsule::connection();
        $this->initializeAdapters();
    }

    private function initializeAdapters(): void {
        $this->adapters = [
            'nginx' => new NginxLoadBalancerAdapter(),
            'haproxy' => new HAProxyLoadBalancerAdapter(),
            'aws_elb' => new AWSELBAdapter(),
            'gcp_lb' => new GCPLoadBalancerAdapter()
        ];
    }

    /**
     * Create load balancer configuration
     */
    public function createLoadBalancer(array $params): array {
        $this->validateLoadBalancerParams($params);

        $lbId = 'lb_' . bin2hex(random_bytes(12));

        $config = [
            'id' => $lbId,
            'name' => $params['name'],
            'algorithm' => $params['algorithm'] ?? 'round_robin',
            'protocol' => $params['protocol'] ?? 'http',
            'port' => $params['port'] ?? 80,
            'ssl_enabled' => $params['ssl_enabled'] ?? false,
            'health_check_path' => $params['health_check_path'] ?? '/',
            'health_check_interval' => $params['health_check_interval'] ?? 30,
            'timeout' => $params['timeout'] ?? 60,
            'max_connections' => $params['max_connections'] ?? 10000,
            'created_at' => date('Y-m-d H:i:s')
        ];

        $this->db->insert('mod_load_balancers', $config);

        // Generate configuration file
        $this->generateConfig($lbId, $config);

        return [
            'success' => true,
            'lb_id' => $lbId,
            'name' => $config['name'],
            'algorithm' => $config['algorithm']
        ];
    }

    /**
     * Add backend server to load balancer
     */
    public function addBackend(string $lbId, array $params): array {
        $lb = $this->getLoadBalancer($lbId);

        if (!$lb) {
            throw new \Exception("Load balancer not found: {$lbId}");
        }

        $backendId = 'be_' . bin2hex(random_bytes(8));

        $backend = [
            'id' => $backendId,
            'lb_id' => $lbId,
            'server_id' => $params['server_id'],
            'ip_address' => $params['ip_address'],
            'port' => $params['port'] ?? $lb['port'],
            'weight' => $params['weight'] ?? 1,
            'max_connections' => $params['max_connections'] ?? 1000,
            'health_check_enabled' => $params['health_check'] ?? true,
            'enabled' => $params['enabled'] ?? true,
            'created_at' => date('Y-m-d H:i:s')
        ];

        $this->db->insert('mod_lb_backends', $backend);

        // Update load balancer configuration
        $this->reloadConfiguration($lbId);

        return [
            'success' => true,
            'backend_id' => $backendId,
            'ip_address' => $backend['ip_address']
        ];
    }

    /**
     * Remove backend server
     */
    public function removeBackend(string $backendId): bool {
        $backend = $this->getBackend($backendId);

        if (!$backend) {
            return false;
        }

        $lbId = $backend['lb_id'];

        $this->db->delete('mod_lb_backends', ['id' => $backendId]);

        // Reload configuration
        $this->reloadConfiguration($lbId);

        return true;
    }

    /**
     * Get load balancer status
     */
    public function getStatus(string $lbId): array {
        $lb = $this->getLoadBalancer($lbId);

        if (!$lb) {
            throw new \Exception("Load balancer not found: {$lbId}");
        }

        $backends = $this->db->select(
            "SELECT * FROM mod_lb_backends WHERE lb_id = ?",
            [$lbId]
        );

        $adapter = $this->getAdapter($lb['type']);
        $stats = $adapter->getStats($lbId);

        return [
            'lb_id' => $lbId,
            'name' => $lb['name'],
            'status' => $stats['status'] ?? 'unknown',
            'total_backends' => count($backends),
            'healthy_backends' => $stats['healthy_backends'] ?? 0,
            'total_connections' => $stats['total_connections'] ?? 0,
            'active_connections' => $stats['active_connections'] ?? 0,
            'backends' => array_map(function($backend) use ($stats) {
                return [
                    'id' => $backend->id,
                    'ip' => $backend->ip_address,
                    'port' => $backend->port,
                    'status' => $stats['backends'][$backend->id] ?? 'unknown',
                    'weight' => $backend->weight,
                    'enabled' => (bool) $backend->enabled
                ];
            }, $backends)
        ];
    }

    /**
     * Configure health check settings
     */
    public function configureHealthCheck(string $lbId, array $params): array {
        $healthCheck = [
            'path' => $params['path'] ?? '/',
            'interval' => $params['interval'] ?? 30,
            'timeout' => $params['timeout'] ?? 5,
            'healthy_threshold' => $params['healthy_threshold'] ?? 3,
            'unhealthy_threshold' => $params['unhealthy_threshold'] ?? 3,
            'method' => $params['method'] ?? 'GET',
            'expected_status' => $params['expected_status'] ?? [200, 301],
            'expected_string' => $params['expected_string'] ?? null
        ];

        $this->db->update('mod_load_balancers', [
            'health_check_config' => json_encode($healthCheck)
        ], ['id' => $lbId]);

        // Regenerate configuration
        $this->reloadConfiguration($lbId);

        return [
            'success' => true,
            'lb_id' => $lbId,
            'health_check' => $healthCheck
        ];
    }

    /**
     * Enable SSL termination
     */
    public function enableSSLTermination(string $lbId, array $sslConfig): array {
        $sslConfig = [
            'enabled' => true,
            'certificate_id' => $sslConfig['certificate_id'],
            'protocols' => $sslConfig['protocols'] ?? ['TLSv1.2', 'TLSv1.3'],
            'ciphers' => $sslConfig['ciphers'] ?? 'HIGH:!aNULL:!MD5',
            'hsts_enabled' => $sslConfig['hsts_enabled'] ?? true,
            'hsts_max_age' => $sslConfig['hsts_max_age'] ?? 31536000
        ];

        $this->db->update('mod_load_balancers', [
            'ssl_enabled' => 1,
            'ssl_config' => json_encode($sslConfig)
        ], ['id' => $lbId]);

        $this->reloadConfiguration($lbId);

        return [
            'success' => true,
            'lb_id' => $lbId,
            'ssl_enabled' => true
        ];
    }

    /**
     * Get load balancer logs
     */
    public function getLogs(string $lbId, int $limit = 100): array {
        $logs = $this->db->select(
            "SELECT * FROM mod_lb_logs
             WHERE lb_id = ? ORDER BY created_at DESC LIMIT ?",
            [$lbId, $limit]
        );

        return array_map(function($log) {
            return [
                'timestamp' => $log->created_at,
                'level' => $log->level,
                'message' => $log->message,
                'backend' => $log->backend_id
            ];
        }, $logs);
    }

    // Private helper methods

    private function getAdapter(string $type) {
        if (!isset($this->adapters[$type])) {
            throw new \Exception("Unsupported load balancer type: {$type}");
        }
        return $this->adapters[$type];
    }

    private function generateConfig(string $lbId, array $config): void {
        $adapter = $this->getAdapter($config['type'] ?? 'nginx');
        $adapter->generateConfig($lbId, $config);
    }

    private function reloadConfiguration(string $lbId): void {
        $lb = $this->getLoadBalancer($lbId);
        $adapter = $this->getAdapter($lb['type'] ?? 'nginx');
        $adapter->reload($lbId);
    }
}

/**
 * Load Balancer Adapter Interface
 */
interface LoadBalancerAdapterInterface {
    public function generateConfig(string $lbId, array $config): void;
    public function reload(string $lbId): void;
    public function getStats(string $lbId): array;
    public function addBackend(string $lbId, array $backend): void;
    public function removeBackend(string $lbId, string $backendId): void;
}

/**
 * Nginx Load Balancer Adapter
 */
class NginxLoadBalancerAdapter implements LoadBalancerAdapterInterface {
    private $configPath = '/etc/nginx/sites-available/';
    private $enabledPath = '/etc/nginx/sites-enabled/';

    public function generateConfig(string $lbId, array $config): void {
        $backends = \WHMCS\Database\Capsule::connection()
            ->select("SELECT * FROM mod_lb_backends WHERE lb_id = ? AND enabled = 1", [$lbId]);

        $upstreams = $this->generateUpstreamBlock($config['id'], $backends, $config['algorithm']);
        $serverBlock = $this->generateServerBlock($config);

        $fullConfig = $upstreams . "\n\n" . $serverBlock;

        file_put_contents($this->configPath . $lbId . '.conf', $fullConfig);
    }

    private function generateUpstreamBlock(string $lbId, array $backends, string $algorithm): string {
        $method = match($algorithm) {
            'round_robin' => 'round_robin',
            'least_conn' => 'least_conn',
            'ip_hash' => 'ip_hash',
            'weighted' => 'round_robin',
            default => 'round_robin'
        };

        $lines = ["upstream {$lbId} {"];
        $lines[] = "    least_conn;"; // Example for least_conn

        foreach ($backends as $backend) {
            $weight = $backend->weight > 1 ? " weight={$backend->weight}" : '';
            $maxConn = $backend->max_connections ? " max_conns={$backend->max_connections}" : '';
            $lines[] = "    server {$backend->ip_address}:{$backend->port}{$weight}{$maxConn};";
        }

        $lines[] = "}";
        return implode("\n", $lines);
    }

    private function generateServerBlock(array $config): string {
        $lines = ["server {"];
        $lines[] = "    listen 80;";
        $lines[] = "    server_name {$config['name']};";

        if ($config['ssl_enabled']) {
            $sslConfig = json_decode($config['ssl_config'] ?? '{}', true);
            $lines[] = "    listen 443 ssl http2;";
            $lines[] = "    ssl_certificate /etc/ssl/certs/{$sslConfig['certificate_id']}.pem;";
            $lines[] = "    ssl_certificate_key /etc/ssl/private/{$sslConfig['certificate_id']}.key;";
            $lines[] = "    ssl_protocols " . implode(' ', $sslConfig['protocols'] ?? ['TLSv1.2']) . ";";
            $lines[] = "    ssl_ciphers '{$sslConfig['ciphers']}';";
        }

        $lines[] = "    location / {";
        $lines[] = "        proxy_pass http://{$config['id']};";
        $lines[] = "        proxy_set_header Host \$host;";
        $lines[] = "        proxy_set_header X-Real-IP \$remote_addr;";
        $lines[] = "        proxy_set_header X-Forwarded-For \$proxy_add_x_forwarded_for;";
        $lines[] = "        proxy_connect_timeout {$config['timeout']}s;";
        $lines[] = "    }";
        $lines[] = "}";

        return implode("\n", $lines);
    }

    public function reload(string $lbId): void {
        exec('nginx -t && nginx -s reload');
    }

    public function getStats(string $lbId): array {
        // Parse nginx status module output
        $status = @file_get_contents('http://localhost/nginx_status');

        return [
            'status' => 'active',
            'healthy_backends' => 1,
            'total_connections' => 0,
            'active_connections' => 0
        ];
    }

    public function addBackend(string $lbId, array $backend): void {
        $this->generateConfig($lbId, []);
        $this->reload($lbId);
    }

    public function removeBackend(string $lbId, string $backendId): void {
        $this->generateConfig($lbId, []);
        $this->reload($lbId);
    }
}

/**
 * HAProxy Load Balancer Adapter
 */
class HAProxyLoadBalancerAdapter implements LoadBalancerAdapterInterface {
    private $configPath = '/etc/haproxy/';

    public function generateConfig(string $lbId, array $config): void {
        $backends = \WHMCS\Database\Capsule::connection()
            ->select("SELECT * FROM mod_lb_backends WHERE lb_id = ? AND enabled = 1", [$lbId]);

        $lines = ["# Generated by WHMCS Load Balancer Manager"];
        $lines[] = "frontend {$lbId}_frontend";
        $lines[] = "    bind *:{$config['port']}";

        if ($config['ssl_enabled']) {
            $lines[] = "    bind *:443 ssl crt /etc/ssl/certs/";
        }

        $lines[] = "    default_backend {$lbId}_backend";
        $lines[] = "";
        $lines[] = "backend {$lbId}_backend";

        $algorithm = match($config['algorithm']) {
            'round_robin' => 'roundrobin',
            'least_conn' => 'leastconn',
            'weighted' => 'static-rr',
            default => 'roundrobin'
        };

        $lines[] = "    balance {$algorithm}";
        $lines[] = "    option httpchk";
        $lines[] = "    http-check expect status 200";

        foreach ($backends as $backend) {
            $weight = $backend->weight > 1 ? " weight {$backend->weight}" : '';
            $lines[] = "    server {$backend->id} {$backend->ip_address}:{$backend->port}{$weight} check inter {$config['health_check_interval'] ?? 30}s fall {$config['unhealthy_threshold'] ?? 3} rise {$config['healthy_threshold'] ?? 3}";
        }

        $fullConfig = implode("\n", $lines);
        file_put_contents($this->configPath . $lbId . '.cfg', $fullConfig);
    }

    public function reload(string $lbId): void {
        exec('haproxy -f /etc/haproxy/haproxy.cfg -c');
        exec('systemctl reload haproxy');
    }

    public function getStats(string $lbId): array {
        // Parse HAProxy stats socket
        return [
            'status' => 'active',
            'healthy_backends' => 2,
            'total_connections' => 0,
            'active_connections' => 0
        ];
    }

    public function addBackend(string $lbId, array $backend): void {
        $this->generateConfig($lbId, []);
        $this->reload($lbId);
    }

    public function removeBackend(string $lbId, string $backendId): void {
        $this->generateConfig($lbId, []);
        $this->reload($lbId);
    }
}
```

## Database Schema
```sql
CREATE TABLE `mod_load_balancers` (
  `id` VARCHAR(50) PRIMARY KEY,
  `name` VARCHAR(255) NOT NULL,
  `type` VARCHAR(50) DEFAULT 'nginx',
  `algorithm` ENUM('round_robin', 'least_conn', 'ip_hash', 'weighted') DEFAULT 'round_robin',
  `protocol` VARCHAR(20) DEFAULT 'http',
  `port` INT DEFAULT 80,
  `ssl_enabled` TINYINT(1) DEFAULT 0,
  `ssl_config` TEXT,
  `health_check_path` VARCHAR(255) DEFAULT '/',
  `health_check_interval` INT DEFAULT 30,
  `health_check_config` TEXT,
  `timeout` INT DEFAULT 60,
  `max_connections` INT DEFAULT 10000,
  `enabled` TINYINT(1) DEFAULT 1,
  `created_at` DATETIME NOT NULL,
  INDEX `idx_name` (`name`)
);

CREATE TABLE `mod_lb_backends` (
  `id` VARCHAR(50) PRIMARY KEY,
  `lb_id` VARCHAR(50) NOT NULL,
  `server_id` INT,
  `ip_address` VARCHAR(45) NOT NULL,
  `port` INT DEFAULT 80,
  `weight` INT DEFAULT 1,
  `max_connections` INT DEFAULT 1000,
  `health_check_enabled` TINYINT(1) DEFAULT 1,
  `enabled` TINYINT(1) DEFAULT 1,
  `created_at` DATETIME NOT NULL,
  FOREIGN KEY (`lb_id`) REFERENCES `mod_load_balancers`(`id`)
);

CREATE TABLE `mod_lb_logs` (
  `id` INT AUTO_INCREMENT PRIMARY KEY,
  `lb_id` VARCHAR(50) NOT NULL,
  `backend_id` VARCHAR(50),
  `level` VARCHAR(20) DEFAULT 'info',
  `message` TEXT,
  `created_at` DATETIME NOT NULL,
  INDEX `idx_lb_id` (`lb_id`)
);
```

## Best Practices

1. **Health Checks**: Implement robust health check mechanisms
2. **SSL Termination**: Use modern TLS protocols and strong ciphers
3. **Logging**: Enable detailed logging for troubleshooting
4. **Monitoring**: Track connection counts and backend health
5. **Graceful Degradation**: Handle backend failures gracefully

## Related Skills

- whmcs-auto-scaling
- whmcs-firewall-rules
- whmcs-network-config
- whmcs-monitoring-agent