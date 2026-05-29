---
name: whmcs-firewall-rules
description: Firewall configuration for WHMCS services
category: Provisioning & Cloud
version: 1.0.0
---

# WHMCS Firewall Configuration Skill

## Overview
This skill provides patterns and implementations for configuring firewall rules in WHMCS, including network security policies, port management, IP whitelisting/blacklisting, and DDoS protection integration.

## Implementation Patterns

### Firewall Manager Class
```php
<?php
/**
 * WHMCS Firewall Configuration
 * Manages firewall rules for hosted services
 */

namespace WHMCS\Module\Server\Firewall;

class FirewallManager {
    private $db;
    private $providers = [];

    public function __construct() {
        $this->db = \WHMCS\Database\Capsule::connection();
        $this->initializeProviders();
    }

    private function initializeProviders(): void {
        $this->providers = [
            'iptables' => new IPTablesProvider(),
            'ufw' => new UFWProvider(),
            'cloudflare' => new CloudflareProvider(),
            'aws_security_group' => new AWS SecurityGroupProvider()
        ];
    }

    /**
     * Create firewall rule
     */
    public function createRule(array $params): array {
        $this->validateRuleParams($params);

        $ruleId = 'fw_' . bin2hex(random_bytes(12));

        $rule = [
            'id' => $ruleId,
            'service_id' => $params['service_id'] ?? null,
            'name' => $params['name'],
            'direction' => $params['direction'] ?? 'ingress', // ingress, egress
            'protocol' => $params['protocol'] ?? 'tcp',
            'port' => $params['port'],
            'source_ip' => $params['source_ip'] ?? '0.0.0.0/0',
            'dest_ip' => $params['dest_ip'] ?? null,
            'action' => $params['action'] ?? 'allow', // allow, deny, log
            'priority' => $params['priority'] ?? 100,
            'enabled' => $params['enabled'] ?? true,
            'description' => $params['description'] ?? '',
            'expires_at' => $params['expires_at'] ?? null,
            'created_at' => date('Y-m-d H:i:s')
        ];

        $this->db->insert('mod_firewall_rules', $rule);

        // Apply rule to server/firewall
        $this->applyRule($rule);

        return [
            'success' => true,
            'rule_id' => $ruleId,
            'name' => $rule['name']
        ];
    }

    /**
     * Create security group
     */
    public function createSecurityGroup(array $params): array {
        $groupId = 'sg_' . bin2hex(random_bytes(12));

        $group = [
            'id' => $groupId,
            'name' => $params['name'],
            'description' => $params['description'] ?? '',
            'service_id' => $params['service_id'] ?? null,
            'created_at' => date('Y-m-d H:i:s')
        ];

        $this->db->insert('mod_firewall_groups', $group);

        // Add initial rules if provided
        if (!empty($params['rules'])) {
            foreach ($params['rules'] as $rule) {
                $rule['service_id'] = $params['service_id'];
                $rule['group_id'] = $groupId;
                $this->createRule($rule);
            }
        }

        return [
            'success' => true,
            'group_id' => $groupId,
            'name' => $params['name']
        ];
    }

    /**
     * List firewall rules
     */
    public function listRules(array $filters = []): array {
        $query = "SELECT * FROM mod_firewall_rules WHERE 1=1";
        $bindings = [];

        if (!empty($filters['service_id'])) {
            $query .= " AND service_id = ?";
            $bindings[] = $filters['service_id'];
        }

        if (!empty($filters['enabled'])) {
            $query .= " AND enabled = ?";
            $bindings[] = $filters['enabled'];
        }

        $query .= " ORDER BY priority ASC, created_at DESC";

        $rules = $this->db->select($query, $bindings);

        return array_map(function($rule) {
            return [
                'id' => $rule->id,
                'name' => $rule->name,
                'direction' => $rule->direction,
                'protocol' => $rule->protocol,
                'port' => $rule->port,
                'source_ip' => $rule->source_ip,
                'action' => $rule->action,
                'enabled' => (bool) $rule->enabled,
                'created_at' => $rule->created_at
            ];
        }, $rules);
    }

    /**
     * Enable/disable rule
     */
    public function toggleRule(string $ruleId, bool $enabled): bool {
        $rule = $this->getRule($ruleId);

        if (!$rule) {
            return false;
        }

        $this->db->update('mod_firewall_rules', [
            'enabled' => $enabled ? 1 : 0
        ], ['id' => $ruleId]);

        // Update firewall
        if ($enabled) {
            $this->applyRule($rule);
        } else {
            $this->removeRule($rule);
        }

        return true;
    }

    /**
     * Delete rule
     */
    public function deleteRule(string $ruleId): bool {
        $rule = $this->getRule($ruleId);

        if (!$rule) {
            return false;
        }

        $this->removeRule($rule);
        $this->db->delete('mod_firewall_rules', ['id' => $ruleId]);

        return true;
    }

    /**
     * Create temporary whitelist entry
     */
    public function createWhitelistEntry(array $params): array {
        $entryId = 'wl_' . bin2hex(random_bytes(8));

        $entry = [
            'id' => $entryId,
            'service_id' => $params['service_id'],
            'ip_address' => $params['ip_address'],
            'port' => $params['port'] ?? 'all',
            'reason' => $params['reason'] ?? 'Manual whitelist',
            'expires_at' => $params['expires_at'] ?? null,
            'created_by' => $params['admin_id'],
            'created_at' => date('Y-m-d H:i:s')
        ];

        $this->db->insert('mod_firewall_whitelist', $entry);

        // Create rule for whitelist
        $this->createRule([
            'service_id' => $params['service_id'],
            'name' => "whitelist-{$params['ip_address']}",
            'direction' => 'ingress',
            'protocol' => $params['protocol'] ?? 'tcp',
            'port' => $params['port'] ?? '1:65535',
            'source_ip' => $params['ip_address'],
            'action' => 'allow',
            'priority' => 10,
            'expires_at' => $params['expires_at']
        ]);

        return [
            'success' => true,
            'entry_id' => $entryId,
            'ip_address' => $params['ip_address']
        ];
    }

    /**
     * Create temporary blacklist entry
     */
    public function createBlacklistEntry(array $params): array {
        $entryId = 'bl_' . bin2hex(random_bytes(8));

        $entry = [
            'id' => $entryId,
            'service_id' => $params['service_id'] ?? null,
            'ip_address' => $params['ip_address'],
            'reason' => $params['reason'] ?? 'Manual blacklist',
            'expires_at' => $params['expires_at'] ?? null,
            'created_by' => $params['admin_id'],
            'created_at' => date('Y-m-d H:i:s')
        ];

        $this->db->insert('mod_firewall_blacklist', $entry);

        // Create deny rule
        $this->createRule([
            'name' => "blacklist-{$params['ip_address']}",
            'direction' => 'ingress',
            'protocol' => 'all',
            'port' => 'all',
            'source_ip' => $params['ip_address'],
            'action' => 'deny',
            'priority' => 1
        ]);

        return [
            'success' => true,
            'entry_id' => $entryId,
            'ip_address' => $params['ip_address']
        ];
    }

    /**
     * Get firewall logs
     */
    public function getLogs(int $serviceId, int $limit = 100): array {
        $logs = $this->db->select(
            "SELECT * FROM mod_firewall_logs
             WHERE service_id = ? ORDER BY created_at DESC LIMIT ?",
            [$serviceId, $limit]
        );

        return array_map(function($log) {
            return [
                'timestamp' => $log->created_at,
                'action' => $log->action,
                'source_ip' => $log->source_ip,
                'dest_ip' => $log->dest_ip,
                'port' => $log->port,
                'protocol' => $log->protocol,
                'bytes' => $log->bytes_count,
                'packets' => $log->packets_count
            ];
        }, $logs);
    }

    /**
     * Block IP across all services
     */
    public function globalBlock(string $ipAddress, string $reason, ?string $expiresAt = null): array {
        // Add to global blacklist
        $this->db->insert('mod_firewall_global_blacklist', [
            'ip_address' => $ipAddress,
            'reason' => $reason,
            'expires_at' => $expiresAt,
            'created_at' => date('Y-m-d H:i:s')
        ]);

        // Apply to all active services
        $services = $this->db->select("SELECT id FROM tblhosting WHERE domainstatus = 'Active'");

        foreach ($services as $service) {
            $this->createBlacklistEntry([
                'service_id' => $service->id,
                'ip_address' => $ipAddress,
                'reason' => $reason,
                'expires_at' => $expiresAt
            ]);
        }

        return [
            'success' => true,
            'ip_address' => $ipAddress,
            'services_affected' => count($services)
        ];
    }

    // Private helper methods

    private function applyRule(array $rule): void {
        $provider = $this->getProvider('iptables');

        if ($rule['action'] === 'allow') {
            $provider->allow($rule);
        } else {
            $provider->deny($rule);
        }

        // Log rule application
        $this->logRuleApplication($rule, 'applied');
    }

    private function removeRule(array $rule): void {
        $provider = $this->getProvider('iptables');
        $provider->remove($rule);

        $this->logRuleApplication($rule, 'removed');
    }

    private function getProvider(string $type) {
        if (!isset($this->providers[$type])) {
            throw new \Exception("Unknown firewall provider: {$type}");
        }
        return $this->providers[$type];
    }
}

/**
 * IPTables Firewall Provider
 */
class IPTablesProvider {
    public function allow(array $rule): void {
        $direction = $rule['direction'] === 'ingress' ? 'INPUT' : 'OUTPUT';
        $port = $this->parsePort($rule['port']);
        $protocol = $rule['protocol'];

        $command = "iptables -A {$direction} -p {$protocol} --dport {$port} -s {$rule['source_ip']} -j ACCEPT";
        exec($command);
    }

    public function deny(array $rule): void {
        $direction = $rule['direction'] === 'ingress' ? 'INPUT' : 'OUTPUT';
        $port = $this->parsePort($rule['port']);
        $protocol = $rule['protocol'];

        $command = "iptables -A {$direction} -p {$protocol} --dport {$port} -s {$rule['source_ip']} -j DROP";
        exec($command);
    }

    public function remove(array $rule): void {
        $direction = $rule['direction'] === 'ingress' ? 'INPUT' : 'OUTPUT';
        $port = $this->parsePort($rule['port']);
        $protocol = $rule['protocol'];

        $command = "iptables -D {$direction} -p {$protocol} --dport {$port} -s {$rule['source_ip']} -j ACCEPT";
        exec($command);
    }

    private function parsePort(string $port): string {
        if (strpos($port, ':') !== false) {
            return $port;
        }
        return (int) $port;
    }
}

/**
 * Cloudflare Firewall Provider
 */
class CloudflareProvider {
    private $apiToken;

    public function __construct() {
        $config = $this->loadConfig();
        $this->apiToken = $config['cloudflare_token'];
    }

    public function blockIP(string $ip): void {
        $ch = curl_init("https://api.cloudflare.com/client/v4/user/firewall/access_rules/rules");
        curl_setopt_array($ch, [
            CURLOPT_POST => true,
            CURLOPT_POSTFIELDS => json_encode([
                'filter' => ['expression' => "ip.src eq {$ip}"],
                'action' => 'block',
                'description' => 'WHMCS blocked'
            ]),
            CURLOPT_HTTPHEADER => [
                'Authorization: Bearer ' . $this->apiToken,
                'Content-Type: application/json'
            ],
            CURLOPT_RETURNTRANSFER => true
        ]);
        curl_exec($ch);
        curl_close($ch);
    }

    public function allowIP(string $ip): void {
        $ch = curl_init("https://api.cloudflare.com/client/v4/user/firewall/access_rules/rules");
        curl_setopt_array($ch, [
            CURLOPT_POST => true,
            CURLOPT_POSTFIELDS => json_encode([
                'filter' => ['expression' => "ip.src eq {$ip}"],
                'action' => 'allow',
                'description' => 'WHMCS whitelisted'
            ]),
            CURLOPT_HTTPHEADER => [
                'Authorization: Bearer ' . $this->apiToken,
                'Content-Type: application/json'
            ],
            CURLOPT_RETURNTRANSFER => true
        ]);
        curl_exec($ch);
        curl_close($ch);
    }
}
```

## Database Schema
```sql
CREATE TABLE `mod_firewall_rules` (
  `id` VARCHAR(50) PRIMARY KEY,
  `service_id` INT,
  `group_id` VARCHAR(50),
  `name` VARCHAR(255) NOT NULL,
  `direction` ENUM('ingress', 'egress') DEFAULT 'ingress',
  `protocol` VARCHAR(20) DEFAULT 'tcp',
  `port` VARCHAR(50) NOT NULL,
  `source_ip` VARCHAR(50) DEFAULT '0.0.0.0/0',
  `dest_ip` VARCHAR(50),
  `action` ENUM('allow', 'deny', 'log') DEFAULT 'allow',
  `priority` INT DEFAULT 100,
  `enabled` TINYINT(1) DEFAULT 1,
  `description` TEXT,
  `expires_at` DATETIME,
  `created_at` DATETIME NOT NULL
);

CREATE TABLE `mod_firewall_groups` (
  `id` VARCHAR(50) PRIMARY KEY,
  `name` VARCHAR(255) NOT NULL,
  `description` TEXT,
  `service_id` INT,
  `created_at` DATETIME NOT NULL
);

CREATE TABLE `mod_firewall_whitelist` (
  `id` VARCHAR(50) PRIMARY KEY,
  `service_id` INT NOT NULL,
  `ip_address` VARCHAR(45) NOT NULL,
  `port` VARCHAR(50) DEFAULT 'all',
  `reason` VARCHAR(255),
  `expires_at` DATETIME,
  `created_by` INT,
  `created_at` DATETIME NOT NULL
);

CREATE TABLE `mod_firewall_blacklist` (
  `id` VARCHAR(50) PRIMARY KEY,
  `service_id` INT,
  `ip_address` VARCHAR(45) NOT NULL,
  `reason` VARCHAR(255),
  `expires_at` DATETIME,
  `created_by` INT,
  `created_at` DATETIME NOT NULL
);

CREATE TABLE `mod_firewall_logs` (
  `id` BIGINT AUTO_INCREMENT PRIMARY KEY,
  `service_id` INT NOT NULL,
  `action` VARCHAR(20),
  `source_ip` VARCHAR(45),
  `dest_ip` VARCHAR(45),
  `port` INT,
  `protocol` VARCHAR(20),
  `bytes_count` BIGINT,
  `packets_count` BIGINT,
  `created_at` DATETIME NOT NULL,
  INDEX `idx_service_id` (`service_id`)
);
```

## Common Firewall Rules Template

| Service Type | Port | Protocol | Action | Description |
|--------------|------|----------|--------|-------------|
| Web Server | 80, 443 | TCP | Allow | HTTP/HTTPS |
| SSH | 22 | TCP | Allow | Secure Shell |
| FTP | 20, 21 | TCP | Allow | FTP Access |
| SMTP | 25, 465, 587 | TCP | Allow | Email |
| DNS | 53 | UDP | Allow | DNS Queries |
| MySQL | 3306 | TCP | Allow | MySQL (local only) |
| PostgreSQL | 5432 | TCP | Allow | PostgreSQL |
| RDP | 3389 | TCP | Allow | Windows RDP |

## Best Practices

1. **Default Deny**: Start with default deny policy
2. **Least Privilege**: Only open required ports
3. **Logging**: Enable logging for security events
4. **Regular Review**: Audit rules periodically
5. **Timeout Rules**: Set expiration for temporary access

## Related Skills

- whmcs-network-config
- whmcs-load-balancer-config
- whmcs-security-headers
- whmcs-monitoring-agent