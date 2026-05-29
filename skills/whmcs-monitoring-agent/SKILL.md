---
name: whmcs-monitoring-agent
description: Monitoring agent deployment for WHMCS
category: Provisioning & Cloud
version: 1.0.0
---

# WHMCS Monitoring Agent Deployment Skill

## Overview
This skill provides patterns and implementations for deploying and managing monitoring agents in WHMCS, including metrics collection, alerting configuration, and dashboard integration.

## Implementation Patterns

### Monitoring Agent Manager
```php
<?php
/**
 * WHMCS Monitoring Agent Deployment
 * Manages monitoring agents for hosted services
 */

namespace WHMCS\Module\Server\Monitoring;

class MonitoringAgentManager {
    private $db;

    public function __construct() {
        $this->db = \WHMCS\Database\Capsule::connection();
    }

    /**
     * Deploy monitoring agent to service
     */
    public function deployAgent(array $params): array {
        $serviceId = $params['service_id'];
        $vmId = $this->getVmId($serviceId);
        $hypervisor = $this->getHypervisorType($serviceId);
        $vmIp = $this->getVmIP($serviceId);

        $agentId = 'ma_' . bin2hex(random_bytes(12));
        $agentToken = bin2hex(random_bytes(32));

        // Create agent configuration
        $config = [
            'id' => $agentId,
            'service_id' => $serviceId,
            'token' => $agentToken,
            'server_url' => $params['server_url'] ?? $this->getDefaultServerUrl(),
            'collectors' => $params['collectors'] ?? ['cpu', 'memory', 'disk', 'network'],
            'interval' => $params['interval'] ?? 60,
            'status' => 'installing',
            'created_at' => date('Y-m-d H:i:s')
        ];

        $this->db->insert('mod_monitoring_agents', $config);

        // Install agent on VM
        $this->installAgentOnVM($vmId, $vmIp, $config);

        // Start agent
        $this->startAgent($vmId);

        return [
            'success' => true,
            'agent_id' => $agentId,
            'token' => $agentToken
        ];
    }

    /**
     * Configure alerting for service
     */
    public function configureAlerting(string $agentId, array $alertConfig): array {
        $agent = $this->getAgent($agentId);

        if (!$agent) {
            throw new \Exception("Agent not found: {$agentId}");
        }

        $alerts = [];

        // CPU alert
        if (!empty($alertConfig['cpu_threshold'])) {
            $alerts[] = [
                'metric' => 'cpu_usage',
                'condition' => '>',
                'threshold' => $alertConfig['cpu_threshold'],
                'duration' => $alertConfig['cpu_duration'] ?? 300,
                'severity' => $alertConfig['cpu_severity'] ?? 'warning'
            ];
        }

        // Memory alert
        if (!empty($alertConfig['memory_threshold'])) {
            $alerts[] = [
                'metric' => 'memory_usage',
                'condition' => '>',
                'threshold' => $alertConfig['memory_threshold'],
                'duration' => $alertConfig['memory_duration'] ?? 300,
                'severity' => $alertConfig['memory_severity'] ?? 'warning'
            ];
        }

        // Disk alert
        if (!empty($alertConfig['disk_threshold'])) {
            $alerts[] = [
                'metric' => 'disk_usage',
                'condition' => '>',
                'threshold' => $alertConfig['disk_threshold'],
                'duration' => $alertConfig['disk_duration'] ?? 60,
                'severity' => $alertConfig['disk_severity'] ?? 'critical'
            ];
        }

        // Store alert configuration
        $this->db->update('mod_monitoring_agents', [
            'alert_config' => json_encode($alerts)
        ], ['id' => $agentId]);

        // Push configuration to agent
        $this->pushAlertConfigToAgent($agentId, $alerts);

        return [
            'success' => true,
            'agent_id' => $agentId,
            'alerts_configured' => count($alerts)
        ];
    }

    /**
     * Get monitoring metrics
     */
    public function getMetrics(int $serviceId, array $params = []): array {
        $from = $params['from'] ?? date('Y-m-d H:i:s', strtotime('-1 hour'));
        $to = $params['to'] ?? date('Y-m-d H:i:s');
        $resolution = $params['resolution'] ?? '1m';

        $metrics = $this->db->select(
            "SELECT * FROM mod_monitoring_metrics
             WHERE service_id = ? AND created_at BETWEEN ? AND ?
             ORDER BY created_at ASC",
            [$serviceId, $from, $to]
        );

        // Group by timestamp based on resolution
        return $this->aggregateMetrics($metrics, $resolution);
    }

    /**
     * Get real-time status
     */
    public function getStatus(int $serviceId): array {
        $agent = $this->db->select(
            "SELECT * FROM mod_monitoring_agents WHERE service_id = ? ORDER BY created_at DESC LIMIT 1",
            [$serviceId]
        )[0];

        if (!$agent) {
            return ['status' => 'not_deployed', 'service_id' => $serviceId];
        }

        // Get latest metrics
        $latestMetrics = $this->db->select(
            "SELECT * FROM mod_monitoring_metrics
             WHERE service_id = ? ORDER BY created_at DESC LIMIT 1",
            [$serviceId]
        )[0];

        // Check if agent is responsive
        $lastSeen = strtotime($agent->last_seen ?? $agent->created_at);
        $isOnline = (time() - $lastSeen) < 300;

        return [
            'agent_id' => $agent->id,
            'status' => $isOnline ? 'online' : 'offline',
            'last_seen' => $agent->last_seen,
            'version' => $agent->version,
            'metrics' => $latestMetrics ? [
                'cpu_percent' => $latestMetrics->cpu_percent,
                'memory_percent' => $latestMetrics->memory_percent,
                'disk_percent' => $latestMetrics->disk_percent,
                'network_in' => $latestMetrics->network_in_bps,
                'network_out' => $latestMetrics->network_out_bps
            ] : null
        ];
    }

    /**
     * Configure metric collection
     */
    public function configureMetrics(string $agentId, array $config): array {
        $collectionConfig = [
            'interval' => $config['interval'] ?? 60,
            'collectors' => $config['collectors'] ?? [],
            'retention_days' => $config['retention_days'] ?? 30,
            'aggregate' => $config['aggregate'] ?? true
        ];

        $this->db->update('mod_monitoring_agents', [
            'collection_config' => json_encode($collectionConfig)
        ], ['id' => $agentId]);

        return [
            'success' => true,
            'agent_id' => $agentId,
            'config' => $collectionConfig
        ];
    }

    /**
     * Uninstall agent
     */
    public function uninstallAgent(string $agentId): bool {
        $agent = $this->getAgent($agentId);

        if (!$agent) {
            return false;
        }

        // Run uninstall script on VM
        $vmId = $this->getVmIdByAgent($agentId);
        $this->runUninstallScript($vmId);

        // Update status
        $this->db->update('mod_monitoring_agents', [
            'status' => 'uninstalled',
            'uninstalled_at' => date('Y-m-d H:i:s')
        ], ['id' => $agentId]);

        return true;
    }

    /**
     * Get alert history
     */
    public function getAlertHistory(int $serviceId, int $limit = 100): array {
        $alerts = $this->db->select(
            "SELECT * FROM mod_monitoring_alerts
             WHERE service_id = ? ORDER BY created_at DESC LIMIT ?",
            [$serviceId, $limit]
        );

        return array_map(function($alert) {
            return [
                'id' => $alert->id,
                'metric' => $alert->metric,
                'condition' => $alert->condition,
                'threshold' => $alert->threshold,
                'actual_value' => $alert->actual_value,
                'severity' => $alert->severity,
                'status' => $alert->status,
                'created_at' => $alert->created_at,
                'resolved_at' => $alert->resolved_at
            ];
        }, $alerts);
    }

    /**
     * Acknowledge alert
     */
    public function acknowledgeAlert(string $alertId, int $adminId): bool {
        $this->db->update('mod_monitoring_alerts', [
            'status' => 'acknowledged',
            'acknowledged_by' => $adminId,
            'acknowledged_at' => date('Y-m-d H:i:s')
        ], ['id' => $alertId]);

        return true;
    }

    // Private helper methods

    private function installAgentOnVM(string $vmId, string $vmIp, array $config): void {
        $installScript = $this->generateInstallScript($config);

        // Copy and execute installation script
        $this->copyToVM($vmId, $vmIp, '/tmp/install_agent.sh', $installScript);
        $this->executeOnVM($vmId, 'bash /tmp/install_agent.sh');
    }

    private function generateInstallScript(array $config): string {
        return <<<SCRIPT
#!/bin/bash
# WHMCS Monitoring Agent Installation Script

set -e

AGENT_TOKEN="{$config['token']}"
SERVER_URL="{$config['server_url']}"
COLLECTORS="{$config['collectors']}"
INTERVAL="{$config['interval']}"

# Detect OS
if [ -f /etc/debian_version ]; then
    OS="debian"
elif [ -f /etc/redhat-release ]; then
    OS="rhel"
else
    OS="unknown"
fi

# Create agent user
useradd -r -s /bin/false whmcs-agent || true

# Download and install agent
curl -sL -o /opt/whmcs-agent.tar.gz "{$config['server_url']}/agent/download"
tar -xzf /opt/whmcs-agent.tar.gz -C /opt/
chmod +x /opt/whmcs-agent/agent

# Create configuration
cat > /etc/whmcs-agent.conf << EOF
server_url={$config['server_url']}
agent_token={$config['token']}
collectors={$config['collectors']}
interval={$config['interval']}
EOF

# Setup systemd service
cat > /etc/systemd/system/whmcs-agent.service << EOF
[Unit]
Description=WHMCS Monitoring Agent
After=network.target

[Service]
Type=simple
ExecStart=/opt/whmcs-agent/agent start
Restart=on-failure

[Install]
WantedBy=multi-user.target
EOF

systemctl enable whmcs-agent
systemctl start whmcs-agent

echo "Agent installed successfully"
SCRIPT;
    }

    private function startAgent(string $vmId): void {
        $this->executeOnVM($vmId, 'systemctl start whmcs-agent');
    }

    private function pushAlertConfigToAgent(string $agentId, array $alerts): void {
        $agent = $this->getAgent($agentId);
        $vmId = $this->getVmIdByAgent($agentId);

        $configPayload = json_encode(['alerts' => $alerts]);
        $this->executeOnVM($vmId, "curl -X POST http://localhost:9100/config -d '{$configPayload}'");
    }
}

/**
 * Metrics Aggregator
 */
class MetricsAggregator {
    public function aggregateMetrics(array $metrics, string $resolution): array {
        if (empty($metrics)) {
            return [];
        }

        $interval = $this->getIntervalSeconds($resolution);
        $aggregated = [];

        $currentBucket = null;
        $bucketData = [];

        foreach ($metrics as $metric) {
            $timestamp = strtotime($metric->created_at);
            $bucketStart = floor($timestamp / $interval) * $interval;

            if ($currentBucket !== $bucketStart) {
                if ($currentBucket !== null) {
                    $aggregated[] = $this->aggregateBucket($bucketData, $currentBucket);
                }
                $currentBucket = $bucketStart;
                $bucketData = [];
            }

            $bucketData[] = $metric;
        }

        if (!empty($bucketData)) {
            $aggregated[] = $this->aggregateBucket($bucketData, $currentBucket);
        }

        return $aggregated;
    }

    private function getIntervalSeconds(string $resolution): int {
        return match($resolution) {
            '1m' => 60,
            '5m' => 300,
            '15m' => 900,
            '1h' => 3600,
            '1d' => 86400,
            default => 60
        };
    }

    private function aggregateBucket(array $data, int $timestamp): array {
        $cpuValues = array_map(fn($m) => $m->cpu_percent, $data);
        $memValues = array_map(fn($m) => $m->memory_percent, $data);

        return [
            'timestamp' => date('Y-m-d H:i:s', $timestamp),
            'cpu_avg' => array_sum($cpuValues) / count($cpuValues),
            'cpu_max' => max($cpuValues),
            'memory_avg' => array_sum($memValues) / count($memValues),
            'memory_max' => max($memValues),
            'samples' => count($data)
        ];
    }
}
```

## Database Schema
```sql
CREATE TABLE `mod_monitoring_agents` (
  `id` VARCHAR(50) PRIMARY KEY,
  `service_id` INT NOT NULL,
  `token` VARCHAR(100) NOT NULL,
  `server_url` VARCHAR(255) NOT NULL,
  `collectors` TEXT,
  `interval` INT DEFAULT 60,
  `alert_config` TEXT,
  `collection_config` TEXT,
  `status` ENUM('installing', 'online', 'offline', 'uninstalled') DEFAULT 'installing',
  `version` VARCHAR(20),
  `last_seen` DATETIME,
  `created_at` DATETIME NOT NULL,
  UNIQUE KEY `unique_service` (`service_id`)
);

CREATE TABLE `mod_monitoring_metrics` (
  `id` BIGINT AUTO_INCREMENT PRIMARY KEY,
  `service_id` INT NOT NULL,
  `agent_id` VARCHAR(50) NOT NULL,
  `cpu_percent` FLOAT DEFAULT 0,
  `memory_percent` FLOAT DEFAULT 0,
  `disk_percent` FLOAT DEFAULT 0,
  `network_in_bps` BIGINT DEFAULT 0,
  `network_out_bps` BIGINT DEFAULT 0,
  `load_avg` FLOAT,
  `created_at` DATETIME NOT NULL,
  INDEX `idx_service_time` (`service_id`, `created_at`)
);

CREATE TABLE `mod_monitoring_alerts` (
  `id` INT AUTO_INCREMENT PRIMARY KEY,
  `service_id` INT NOT NULL,
  `agent_id` VARCHAR(50) NOT NULL,
  `metric` VARCHAR(50) NOT NULL,
  `condition` VARCHAR(10) NOT NULL,
  `threshold` FLOAT NOT NULL,
  `actual_value` FLOAT NOT NULL,
  `severity` ENUM('info', 'warning', 'critical') DEFAULT 'warning',
  `status` ENUM('firing', 'acknowledged', 'resolved') DEFAULT 'firing',
  `created_at` DATETIME NOT NULL,
  `acknowledged_by` INT,
  `acknowledged_at` DATETIME,
  `resolved_at` DATETIME,
  INDEX `idx_service_status` (`service_id`, `status`)
);
```

## Installation Script Template
```bash
#!/bin/bash
# Agent Installation Script
# Generated by WHMCS Monitoring Module

# Configuration
AGENT_TOKEN="${AGENT_TOKEN}"
SERVER_URL="${SERVER_URL}"
INTERVAL="${INTERVAL:-60}"

# Download and install
curl -sL "${SERVER_URL}/downloads/agent/latest" | tar -xz -C /opt/
chmod +x /opt/whmcs-agent/agent

# Configure
cat > /etc/whmcs-agent.json <<EOF
{
  "server_url": "${SERVER_URL}",
  "token": "${AGENT_TOKEN}",
  "interval": ${INTERVAL}
}
EOF

# Enable and start
systemctl enable whmcs-agent
systemctl start whmcs-agent
```

## Best Practices

1. **Lightweight Agents**: Keep agent resource footprint minimal
2. **Secure Communication**: Use TLS for agent-server communication
3. **Local Processing**: Aggregate metrics locally before transmission
4. **Resilient Connection**: Handle network interruptions gracefully
5. **Configurable Retention**: Balance storage costs with historical data needs

## Related Skills

- whmcs-log-collection
- whmcs-incident-response
- whmcs-metrics-analytics
- whmcs-monitoring-dashboard