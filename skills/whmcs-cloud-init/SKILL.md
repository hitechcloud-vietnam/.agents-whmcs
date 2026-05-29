---
name: whmcs-cloud-init
description: Cloud-init configuration for WHMCS provisioning
category: Provisioning & Cloud
version: 1.0.0
---

# WHMCS Cloud-Init Configuration Skill

## Overview
This skill provides patterns and implementations for configuring cloud-init in WHMCS provisioning modules to automate VM initialization, configuration, and post-provisioning setup.

## Implementation Patterns

### Basic Cloud-Init Configuration
```php
<?php
/**
 * WHMCS Cloud-Init Configuration
 * Automates VM initialization during provisioning
 */

class WHMCSCloudInit {
    private $config;
    private $logPath = '/var/log/whmcs/cloud-init.log';

    /**
     * Generate cloud-init user-data for VM provisioning
     */
    public function generateUserData(array $params): string {
        $userData = [
            '#cloud-config' => true,
            'hostname' => $params['hostname'] ?? $this->generateHostname($params),
            'manage_etc_hosts' => true,
            'timezone' => $params['timezone'] ?? 'UTC',
            'ssh_pwauth' => false,
            'ssh_keys' => [
                'ssh_rsa' => $params['ssh_key'] ?? ''
            ],
            'packages' => $this->getPackages($params),
            'runcmd' => $this->generateRunCommands($params),
            'write_files' => $this->generateConfigFiles($params)
        ];

        return $this->yamlEncode($userData);
    }

    /**
     * Generate cloud-init network config (network-config v1)
     */
    public function generateNetworkConfig(array $params): string {
        $networkConfig = [
            'version' => 1,
            'config' => [
                [
                    'type' => 'physical',
                    'name' => 'eth0',
                    'subnets' => [
                        [
                            'type' => 'dhcp'
                        ]
                    ]
                ]
            ]
        ];

        if (!empty($params['static_ip'])) {
            $networkConfig['config'][0]['subnets'] = [
                [
                    'type' => 'static',
                    'address' => $params['static_ip'],
                    'gateway' => $params['gateway'],
                    'dns_nameservers' => explode(',', $params['dns_servers'] ?? '8.8.8.8,8.8.4.4')
                ]
            ];
        }

        return $this->yamlEncode($networkConfig);
    }

    /**
     * Generate meta-data for instance
     */
    public function generateMetaData(array $params): string {
        return $this->yamlEncode([
            'instance-id' => $params['server_id'] . '-' . $params['service_id'],
            'local-hostname' => $params['hostname'] ?? $this->generateHostname($params),
            'availability-zone' => $params['region'] ?? 'default'
        ]);
    }

    /**
     * Configure cloud-init based on OS template
     */
    private function getPackages(array $params): array {
        $osFamily = $params['os_family'] ?? 'ubuntu';

        switch ($osFamily) {
            case 'ubuntu':
            case 'debian':
                return ['curl', 'wget', 'unzip', 'python3', 'jq'];
            case 'centos':
            case 'rhel':
            case 'almalinux':
                return ['curl', 'wget', 'unzip', 'python3', 'jq', 'yum-utils'];
            case 'windows':
                return [];
            default:
                return ['curl', 'wget', 'unzip'];
        }
    }

    /**
     * Generate post-installation commands
     */
    private function generateRunCommands(array $params): array {
        $commands = [];

        // WHMCS agent installation
        $commands[] = 'curl -s -o /tmp/whmcs-agent.sh ' .
            '"' . $params['whmcs_url'] . '/includes/whmcs-agent.sh"';
        $commands[] = 'chmod +x /tmp/whmcs-agent.sh';
        $commands[] = '/tmp/whmcs-agent.sh ' . $params['service_id'] . ' ' . $params['api_key'];

        // Additional customization based on product
        if (!empty($params['custom_script'])) {
            $commands[] = 'curl -s "' . $params['custom_script'] . '" | bash';
        }

        return $commands;
    }

    /**
     * Generate configuration files to be written
     */
    private function generateConfigFiles(array $params): array {
        $files = [];

        // WHMCS provisioning configuration
        $files[] = [
            'path' => '/etc/whmcs/provision.conf',
            'content' => json_encode([
                'service_id' => $params['service_id'],
                'api_url' => $params['whmcs_url'],
                'api_key' => $params['api_key'],
                'server_id' => $params['server_id'],
                'monitoring_enabled' => $params['monitoring'] ?? true
            ], JSON_PRETTY_PRINT),
            'permissions' => '0640'
        ];

        // Monitoring agent config
        if ($params['monitoring'] ?? true) {
            $files[] = [
                'path' => '/etc/whmcs/monitoring.conf',
                'content' => json_encode([
                    'endpoint' => $params['monitoring_endpoint'] ?? $params['whmcs_url'] . '/modules/servers/monitoring/api',
                    'interval' => 60,
                    'metrics' => ['cpu', 'memory', 'disk', 'network']
                ], JSON_PRETTY_PRINT),
                'permissions' => '0640'
            ];
        }

        return $files;
    }

    /**
     * YAML encoding helper
     */
    private function yamlEncode(array $data): string {
        $lines = [];
        $this->yamlRecursiveEncode($data, $lines, 0);
        return implode("\n", $lines);
    }

    private function yamlRecursiveEncode(array $data, array &$lines, int $indent): void {
        foreach ($data as $key => $value) {
            $prefix = str_repeat('  ', $indent);
            if (is_array($value) && $this->isAssociative($value)) {
                $lines[] = "{$prefix}{$key}:";
                $this->yamlRecursiveEncode($value, $lines, $indent + 1);
            } elseif (is_array($value)) {
                $lines[] = "{$prefix}{$key}:";
                foreach ($value as $item) {
                    $lines[] = "{$prefix}  - {$item}";
                }
            } elseif ($value === true) {
                $lines[] = "{$prefix}{$key}: true";
            } elseif ($value === false) {
                $lines[] = "{$prefix}{$key}: false";
            } else {
                $lines[] = "{$prefix}{$key}: " . (strpos($value, ':') !== false ? "\"$value\"" : $value);
            }
        }
    }

    private function isAssociative(array $arr): bool {
        if (empty($arr)) return false;
        return array_keys($arr) !== range(0, count($arr) - 1);
    }

    private function generateHostname(array $params): string {
        return sprintf(
            'vm-%s-%s',
            $params['service_id'],
            substr(md5($params['service_id']), 0, 6)
        );
    }

    /**
     * Upload cloud-init configuration to hypervisor
     */
    public function uploadConfig(string $hypervisor, array $config): bool {
        $this->log("Uploading cloud-init config for hypervisor: {$hypervisor}");

        switch ($hypervisor) {
            case 'proxmox':
                return $this->uploadToProxmox($config);
            case 'vmware':
                return $this->uploadToVMware($config);
            case 'libvirt':
                return $this->uploadToLibvirt($config);
            case 'openstack':
                return $this->uploadToOpenStack($config);
            case 'aws':
                return $this->uploadToAWS($config);
            default:
                throw new \Exception("Unsupported hypervisor: {$hypervisor}");
        }
    }

    private function uploadToProxmox(array $config): bool {
        // Proxmox cloud-init configuration via API
        $apiEndpoint = $config['api_endpoint'];
        $node = $config['node'];
        $vmid = $config['vmid'];

        $userData = $this->generateUserData($config['params']);
        $networkConfig = $this->generateNetworkConfig($config['params']);
        $metaData = $this->generateMetaData($config['params']);

        // Upload to Proxmox
        $ch = curl_init("https://{$apiEndpoint}/api2/json/nodes/{$node}/qemu/{$vmid}/config");
        curl_setopt_array($ch, [
            CURLOPT_CUSTOMREQUEST => 'PUT',
            CURLOPT_POSTFIELDS => http_build_query([
                'cicustom' => 1,
                'searchdomain' => 'local'
            ]),
            CURLOPT_HTTPHEADER => ['Authorization: Bearer ' . $config['api_token']],
            CURLOPT_RETURNTRANSFER => true
        ]);
        curl_exec($ch);
        curl_close($ch);

        // Upload cloud-init files to storage
        $this->uploadCloudInitFiles($apiEndpoint, $config['api_token'], $node, $vmid, [
            'user-data' => $userData,
            'network-config' => $networkConfig,
            'meta-data' => $metaData
        ]);

        return true;
    }

    private function uploadToAWS(array $config): bool {
        // AWS EC2 instance launch with user-data
        $instanceId = $config['instance_id'];
        $userData = base64_encode($this->generateUserData($config['params']));

        $command = "aws ec2 run-instances --instance-id {$instanceId} --user-data '{$userData}'";
        exec($command, $output, $return);

        return $return === 0;
    }

    private function uploadCloudInitFiles(string $api, string $token, string $node, int $vmid, array $files): void {
        foreach ($files as $filename => $content) {
            $ch = curl_init("https://{$api}/api2/json/nodes/{$node}/qemu/{$vmid}/cloudinit");
            curl_setopt_array($ch, [
                CURLOPT_CUSTOMREQUEST => 'POST',
                CURLOPT_POSTFIELDS => ['filename' => $filename, 'content' => $content],
                CURLOPT_HTTPHEADER => ['Authorization: Bearer ' . $token],
                CURLOPT_RETURNTRANSFER => true
            ]);
            curl_exec($ch);
            curl_close($ch);
        }
    }

    private function log(string $message): void {
        error_log("[WHMCS Cloud-Init] " . date('Y-m-d H:i:s') . " - " . $message);
    }
}
```

### WHMCS Hook Integration
```php
/**
 * Cloud-init configuration hooks for WHMCS service provisioning
 */

use WHMCS\Database\Capsule;

// Hook: After service provision
add_hook('ServiceProvision', 1, function($vars) {
    $cloudInit = new WHMCSCloudInit();

    $service = Capsule::table('tblhosting')
        ->where('id', $vars['serviceid'])
        ->first();

    $product = Capsule::table('tblproducts')
        ->where('id', $service->packageid)
        ->first();

    $config = [
        'hostname' => $service->domain,
        'service_id' => $service->id,
        'server_id' => $service->server,
        'timezone' => $product->configoption3 ?? 'UTC',
        'whmcs_url' => 'https://' . $_SERVER['HTTP_HOST'],
        'api_key' => get_query_var('api_key'),
        'monitoring' => $product->configoption4 ?? true
    ];

    $userData = $cloudInit->generateUserData($config);
    $networkConfig = $cloudInit->generateNetworkConfig($config);

    // Store cloud-init data for API usage
    Capsule::table('mod_cloudinit')->insert([
        'service_id' => $service->id,
        'user_data' => $userData,
        'network_config' => $networkConfig,
        'created_at' => date('Y-m-d H:i:s')
    ]);
});
```

## Cloud-Init Configuration Templates

### Ubuntu Cloud-Init Template
```yaml
#cloud-config
hostname: ${hostname}
manage_etc_hosts: true
timezone: ${timezone}
ssh_pwauth: false
ssh_keys:
  rsa_private: |
    ${ssh_private_key}
  rsa_public: ${ssh_public_key}

package_update: true
packages:
  - curl
  - wget
  - unzip
  - python3
  - jq
  - fail2ban
  - ufw

write_files:
  - path: /etc/whmcs/provision.conf
    content: |
      {
        "service_id": "${service_id}",
        "api_url": "${whmcs_url}",
        "api_key": "${api_key}"
      }
    permissions: '0640'
    owner: root:root

runcmd:
  - curl -s -o /tmp/whmcs-agent.sh "${whmcs_url}/includes/whmcs-agent.sh"
  - chmod +x /tmp/whmcs-agent.sh
  - bash /tmp/whmcs-agent.sh "${service_id}" "${api_key}"
  - systemctl enable whmcs-monitoring
  - systemctl start whmcs-monitoring

final_message: "WHMCS Cloud-Init completed successfully"
```

## Best Practices

1. **Secure SSH Keys**: Store SSH keys securely and never in plain text
2. **Timeout Configuration**: Set appropriate timeouts for cloud-init operations
3. **Logging**: Enable verbose logging for troubleshooting
4. **Idempotency**: Ensure cloud-init scripts are idempotent
5. **Network Configuration**: Use consistent network interface naming

## Troubleshooting

- Check `/var/log/cloud-init.log` on provisioned instances
- Verify cloud-init version compatibility
- Validate YAML syntax for configuration files
- Ensure network connectivity during initialization

## Related Skills

- whmcs-provisioning-master
- whmcs-monitoring-agent
- whmcs-terraform-modules
- whmcs-docker-setup