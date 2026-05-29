---
name: whmcs-network-config
description: Network setup for WHMCS services
category: Provisioning & Cloud
version: 1.0.0
---

# WHMCS Network Configuration Skill

## Overview
This skill provides patterns and implementations for configuring network settings in WHMCS, including VLAN setup, IP address management, DNS configuration, and network topology management.

## Implementation Patterns

### Network Manager Class
```php
<?php
/**
 * WHMCS Network Configuration
 * Manages network settings for hosted services
 */

namespace WHMCS\Module\Server\Network;

class NetworkManager {
    private $db;

    public function __construct() {
        $this->db = \WHMCS\Database\Capsule::connection();
    }

    /**
     * Create network configuration
     */
    public function createNetwork(array $params): array {
        $networkId = 'net_' . bin2hex(random_bytes(12));

        $network = [
            'id' => $networkId,
            'name' => $params['name'],
            'type' => $params['type'] ?? 'vlan', // vlan, vpc, bridge
            'cidr' => $params['cidr'],
            'gateway' => $params['gateway'],
            'vlan_id' => $params['vlan_id'] ?? null,
            'dns_servers' => implode(',', $params['dns_servers'] ?? ['8.8.8.8', '8.8.4.4']),
            'domain' => $params['domain'] ?? '',
            'created_at' => date('Y-m-d H:i:s')
        ];

        $this->db->insert('mod_networks', $network);

        // Configure on hypervisor
        $this->configureNetwork($network);

        return [
            'success' => true,
            'network_id' => $networkId,
            'cidr' => $network['cidr']
        ];
    }

    /**
     * Assign IP to service
     */
    public function assignIP(array $params): array {
        $serviceId = $params['service_id'];
        $ipType = $params['type'] ?? 'ipv4'; // ipv4, ipv6

        // Get or allocate IP
        $ip = $this->allocateIP($params['network_id'] ?? null, $ipType);

        // Assign to service
        $assignmentId = $this->saveIPAssignment($serviceId, $ip, $ipType);

        // Configure on VM
        $this->configureVMIP($serviceId, $ip, $params);

        return [
            'success' => true,
            'assignment_id' => $assignmentId,
            'ip_address' => $ip,
            'type' => $ipType
        ];
    }

    /**
     * Configure DNS for service
     */
    public function configureDNS(int $serviceId, array $params): array {
        $domain = $params['domain'];
        $records = $params['records'] ?? [];

        // Create A record
        if (!empty($params['ipv4'])) {
            $this->createDNSRecord($domain, 'A', $params['ipv4'], $params['ttl'] ?? 3600);
        }

        // Create AAAA record
        if (!empty($params['ipv6'])) {
            $this->createDNSRecord($domain, 'AAAA', $params['ipv6'], $params['ttl'] ?? 3600);
        }

        // Create CNAME
        if (!empty($params['cname'])) {
            $this->createDNSRecord($domain, 'CNAME', $params['cname'], $params['ttl'] ?? 3600);
        }

        // Create MX record
        if (!empty($params['mx'])) {
            $this->createDNSRecord($domain, 'MX', $params['mx'], $params['ttl'] ?? 3600, $params['mx_priority'] ?? 10);
        }

        return [
            'success' => true,
            'domain' => $domain,
            'records_created' => count($records)
        ];
    }

    /**
     * Create reverse DNS (PTR) record
     */
    public function createReverseDNS(string $ip, string $hostname): array {
        $this->db->insert('mod_reverse_dns', [
            'ip_address' => $ip,
            'hostname' => $hostname,
            'created_at' => date('Y-m-d H:i:s')
        ]);

        // Configure on DNS server
        $this->configurePtrRecord($ip, $hostname);

        return [
            'success' => true,
            'ip' => $ip,
            'hostname' => $hostname
        ];
    }

    /**
     * List IP allocations
     */
    public function listAllocations(array $filters = []): array {
        $query = "SELECT a.*, h.domain as service_name
                  FROM mod_ip_allocations a
                  LEFT JOIN tblhosting h ON a.service_id = h.id
                  WHERE 1=1";

        $bindings = [];

        if (!empty($filters['network_id'])) {
            $query .= " AND a.network_id = ?";
            $bindings[] = $filters['network_id'];
        }

        if (!empty($filters['service_id'])) {
            $query .= " AND a.service_id = ?";
            $bindings[] = $filters['service_id'];
        }

        if (!empty($filters['type'])) {
            $query .= " AND a.type = ?";
            $bindings[] = $filters['type'];
        }

        $allocations = $this->db->select($query, $bindings);

        return array_map(function($alloc) {
            return [
                'id' => $alloc->id,
                'ip_address' => $alloc->ip_address,
                'service_id' => $alloc->service_id,
                'service_name' => $alloc->service_name,
                'network_id' => $alloc->network_id,
                'type' => $alloc->type,
                'assigned_at' => $alloc->created_at
            ];
        }, $allocations);
    }

    /**
     * Configure VPC peering
     */
    public function createPeering(array $params): array {
        $peeringId = 'peer_' . bin2hex(random_bytes(12));

        $peering = [
            'id' => $peeringId,
            'source_network_id' => $params['source_network_id'],
            'dest_network_id' => $params['dest_network_id'],
            'status' => 'pending',
            'created_at' => date('Y-m-d H:i:s')
        ];

        $this->db->insert('mod_network_peerings', $peering);

        // Initiate peering on hypervisor
        $this->initiatePeering($peering);

        return [
            'success' => true,
            'peering_id' => $peeringId
        ];
    }

    /**
     * Get network statistics
     */
    public function getNetworkStats(string $networkId): array {
        $network = $this->db->select(
            "SELECT * FROM mod_networks WHERE id = ?",
            [$networkId]
        )[0];

        $allocations = $this->db->select(
            "SELECT COUNT(*) as total, SUM(CASE WHEN service_id IS NOT NULL THEN 1 ELSE 0 END) as assigned
             FROM mod_ip_allocations WHERE network_id = ?",
            [$networkId]
        )[0];

        $cidrParts = explode('/', $network->cidr);
        $totalIPs = $this->calculateIPCount($cidrParts[1]);

        return [
            'network_id' => $networkId,
            'cidr' => $network->cidr,
            'total_ips' => $totalIPs,
            'allocated_ips' => $allocations->total,
            'assigned_ips' => $allocations->assigned,
            'available_ips' => $totalIPs - $allocations->total
        ];
    }

    /**
     * Release IP address
     */
    public function releaseIP(string $ipAddress): bool {
        $allocation = $this->db->select(
            "SELECT * FROM mod_ip_allocations WHERE ip_address = ?",
            [$ipAddress]
        )[0];

        if (!$allocation) {
            return false;
        }

        // Configure on VM to remove
        $this->removeVMIP($allocation->service_id, $ipAddress);

        // Delete allocation record
        $this->db->delete('mod_ip_allocations', ['ip_address' => $ipAddress]);

        return true;
    }

    // Private helper methods

    private function allocateIP(?string $networkId, string $type): string {
        if ($networkId) {
            // Get from specific network
            $network = $this->db->select(
                "SELECT * FROM mod_networks WHERE id = ?",
                [$networkId]
            )[0];

            $cidr = $network->cidr;
        } else {
            // Get default network
            $network = $this->db->select(
                "SELECT * FROM mod_networks WHERE is_default = 1 LIMIT 1"
            )[0];

            if (!$network) {
                throw new \Exception("No default network configured");
            }

            $cidr = $network->cidr;
        }

        // Find unallocated IP
        $cidrParts = explode('/', $cidr);
        $baseIP = $cidrParts[0];
        $numIPs = $this->calculateIPCount($cidrParts[1]);

        $gatewayIP = $this->ipToLong($baseIP) + 1;
        $dnsIPs = $gatewayIP + 1;
        $startIP = $dnsIPs + 10;

        for ($i = $startIP; $i < $gatewayIP + $numIPs - 1; $i++) {
            $ip = $this->longToIP($i);

            $exists = $this->db->select(
                "SELECT id FROM mod_ip_allocations WHERE ip_address = ?",
                [$ip]
            );

            if (empty($exists)) {
                return $ip;
            }
        }

        throw new \Exception("No available IP addresses in network");
    }

    private function calculateIPCount(int $cidrPrefix): int {
        return pow(2, 32 - $cidrPrefix);
    }

    private function ipToLong(string $ip): int {
        return sprintf('%u', ip2long($ip));
    }

    private function longToIP(int $long): string {
        return long2ip($long);
    }

    private function configureVMIP(int $serviceId, string $ip, array $params): void {
        $vmId = $this->getVmId($serviceId);
        $hypervisor = $this->getHypervisorType($serviceId);

        // Configure via hypervisor API
        $this->setVMIP($vmId, $ip, $params['netmask'] ?? '255.255.255.0', $params['gateway'] ?? '');
    }
}

/**
 * DNS Configuration Handler
 */
class DNSConfigManager {
    public function createDNSRecord(string $name, string $type, string $value, int $ttl = 3600, int $priority = null): bool {
        $recordId = 'dns_' . bin2hex(random_bytes(8));

        $data = [
            'id' => $recordId,
            'name' => $name,
            'type' => $type,
            'value' => $value,
            'ttl' => $ttl,
            'priority' => $priority
        ];

        if ($type === 'MX') {
            $data['priority'] = $priority;
        }

        \WHMCS\Database\Capsule::connection()->insert('mod_dns_records', $data);

        // Push to DNS server
        $this->pushToDNSServer($data);

        return true;
    }

    public function deleteDNSRecord(string $recordId): bool {
        $record = \WHMCS\Database\Capsule::connection()->select(
            "SELECT * FROM mod_dns_records WHERE id = ?",
            [$recordId]
        )[0];

        if (!$record) {
            return false;
        }

        \WHMCS\Database\Capsule::connection()->delete('mod_dns_records', ['id' => $recordId]);
        $this->removeFromDNSServer($record);

        return true;
    }

    public function updateDNSZone(string $domain): bool {
        // Regenerate zone file and reload DNS
        $records = \WHMCS\Database\Capsule::connection()->select(
            "SELECT * FROM mod_dns_records WHERE name LIKE ?",
            ['%' . $domain]
        );

        $zoneContent = $this->generateZoneFile($domain, $records);

        file_put_contents("/etc/bind/zones/{$domain}.db", $zoneContent);
        exec('rndc reload ' . $domain);

        return true;
    }

    private function generateZoneFile(string $domain, array $records): string {
        $lines = [];
        $lines[] = "\$TTL 3600";
        $lines[] = "@ IN SOA ns1.{$domain}. admin.{$domain}. (";
        $lines[] = "    " . time() . " ; Serial";
        $lines[] = "    3600 ; Refresh";
        $lines[] = "    1800 ; Retry";
        $lines[] = "    604800 ; Expire";
        $lines[] = "    3600 ; Minimum TTL";
        $lines[] = ")";
        $lines[] = "";
        $lines[] = "@ IN NS ns1.{$domain}.";
        $lines[] = "@ IN NS ns2.{$domain}.";

        foreach ($records as $record) {
            switch ($record->type) {
                case 'A':
                    $lines[] = "{$record->name} IN A {$record->value}";
                    break;
                case 'AAAA':
                    $lines[] = "{$record->name} IN AAAA {$record->value}";
                    break;
                case 'CNAME':
                    $lines[] = "{$record->name} IN CNAME {$record->value}";
                    break;
                case 'MX':
                    $lines[] = "@ IN MX {$record->priority} {$record->value}";
                    break;
                case 'TXT':
                    $lines[] = "{$record->name} IN TXT \"{$record->value}\"";
                    break;
            }
        }

        return implode("\n", $lines);
    }

    private function pushToDNSServer(array $record): void {
        // Update bind/nsupdate
    }

    private function removeFromDNSServer($record): void {
        // Update bind/nsupdate
    }
}
```

## Database Schema
```sql
CREATE TABLE `mod_networks` (
  `id` VARCHAR(50) PRIMARY KEY,
  `name` VARCHAR(255) NOT NULL,
  `type` ENUM('vlan', 'vpc', 'bridge') DEFAULT 'vlan',
  `cidr` VARCHAR(50) NOT NULL,
  `gateway` VARCHAR(45),
  `vlan_id` INT,
  `dns_servers` TEXT,
  `domain` VARCHAR(255),
  `is_default` TINYINT(1) DEFAULT 0,
  `created_at` DATETIME NOT NULL
);

CREATE TABLE `mod_ip_allocations` (
  `id` VARCHAR(50) PRIMARY KEY,
  `network_id` VARCHAR(50) NOT NULL,
  `ip_address` VARCHAR(45) NOT NULL,
  `service_id` INT,
  `type` ENUM('ipv4', 'ipv6') DEFAULT 'ipv4',
  `created_at` DATETIME NOT NULL,
  UNIQUE KEY `unique_ip` (`ip_address`)
);

CREATE TABLE `mod_dns_records` (
  `id` VARCHAR(50) PRIMARY KEY,
  `name` VARCHAR(255) NOT NULL,
  `type` ENUM('A', 'AAAA', 'CNAME', 'MX', 'TXT', 'NS', 'PTR') NOT NULL,
  `value` TEXT NOT NULL,
  `ttl` INT DEFAULT 3600,
  `priority` INT,
  `created_at` DATETIME NOT NULL
);

CREATE TABLE `mod_reverse_dns` (
  `id` INT AUTO_INCREMENT PRIMARY KEY,
  `ip_address` VARCHAR(45) NOT NULL,
  `hostname` VARCHAR(255) NOT NULL,
  `created_at` DATETIME NOT NULL,
  UNIQUE KEY `unique_ptr` (`ip_address`)
);

CREATE TABLE `mod_network_peerings` (
  `id` VARCHAR(50) PRIMARY KEY,
  `source_network_id` VARCHAR(50) NOT NULL,
  `dest_network_id` VARCHAR(50) NOT NULL,
  `status` ENUM('pending', 'active', 'failed') DEFAULT 'pending',
  `created_at` DATETIME NOT NULL
);
```

## Best Practices

1. **IP Management**: Use DHCP or systematic IP allocation
2. **DNS Redundancy**: Configure multiple DNS servers
3. **Network Segmentation**: Separate services by VLAN
4. **Monitoring**: Track IP utilization and availability
5. **Documentation**: Keep network maps updated

## Related Skills

- whmcs-firewall-rules
- whmcs-storage-provisioning
- whmcs-dns-management
- whmcs-provisioning-master