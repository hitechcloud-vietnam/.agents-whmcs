# WHMCS Dedicated Server Provisioning Module DevKit
# Version: 1.0 | Updated: 2026-05-29

## DevKit Structure

```
devkits/whmcs-dedicated-server/
├── dedicated_server.php      # Main provisioning module
├── lib/
│   └── ApiClient.php         # Dedicated server API client
├── templates/
│   └── clientarea.tpl         # Client area template
└── DEVKIT.md                  # This file
```

## Module Code

```php
<?php
/**
 * WHMCS Dedicated Server Provisioning Module
 */

if (!defined("WHMCS")) {
    die("Direct access denied");
}

function dedicated_server_MetaData(): array {
    return [
        'DisplayName' => 'Dedicated Server',
        'APIVersion' => '1.1',
        'RequiresServer' => true,
        'DefaultNonSSLPort' => 443,
        'Parameters' => ['server_username', 'server_password', 'server_access_hash'],
    ];
}

function dedicated_server_ConfigOptions(array $params): array {
    return [
        'server_type' => [
            'Type' => 'dropdown',
            'Options' => 'basic,standard,premium,enterprise',
            'Default' => 'standard',
            'Description' => 'Server tier',
        ],
        'cpu' => [
            'Type' => 'dropdown',
            'Options' => '4-core,8-core,16-core,32-core,64-core',
            'Default' => '8-core',
            'Description' => 'CPU cores',
        ],
        'ram' => [
            'Type' => 'dropdown',
            'Options' => '16gb,32gb,64gb,128gb,256gb',
            'Default' => '32gb',
            'Description' => 'RAM',
        ],
        'storage' => [
            'Type' => 'dropdown',
            'Options' => '2x1tb-hdd,2x1tb-ssd,2x2tb-ssd,4x2tb-ssd',
            'Default' => '2x1tb-ssd',
            'Description' => 'Storage configuration',
        ],
        'bandwidth' => [
            'Type' => 'dropdown',
            'Options' => '1tb,10tb,unlimited',
            'Default' => '10tb',
            'Description' => 'Monthly bandwidth',
        ],
        'raid' => [
            'Type' => 'dropdown',
            'Options' => 'none,raid0,raid1,raid10',
            'Default' => 'raid1',
            'Description' => 'RAID level',
        ],
        'location' => [
            'Type' => 'dropdown',
            'Options' => 'us-east,us-west,eu-central,eu-west,ap-south',
            'Default' => 'us-east',
            'Description' => 'Datacenter location',
        ],
    ];
}

function dedicated_server_CreateAccount(array $params): string {
    try {
        $api = new DedicatedServer\ApiClient($params);
        
        $result = $api->createServer([
            'hostname' => $params['domain'] ?: 'dedi-' . $params['serviceid'],
            'type' => $params['configoption1'],
            'location' => $params['configoption7'],
            'raid' => $params['configoption6'],
            'ips' => $params['customfields']['Additional IPs'] ?? 1,
        ]);

        saveCustomFieldValue($params['serviceid'], 'Server ID', $result['server_id']);
        saveCustomFieldValue($params['serviceid'], 'Primary IP', $result['primary_ip']);
        saveCustomFieldValue($params['serviceid'], 'Switch Port', $result['switch_port']);
        saveCustomFieldValue($params['serviceid'], 'Root Password', encryptValue($result['root_password']));
        saveCustomFieldValue($params['serviceid'], 'KVM IP', $result['kvm_ip']);
        saveCustomFieldValue($params['serviceid'], 'KVM Port', $result['kvm_port']);

        logActivity("DedicatedServer: Provisioned server {$result['server_id']} in {$params['configoption7']}");
        return 'success';
        
    } catch (\Exception $e) {
        logActivity("DedicatedServer CreateAccount Error: " . $e->getMessage());
        return 'Error: ' . $e->getMessage();
    }
}

function dedicated_server_SuspendAccount(array $params): string {
    try {
        $serverId = getCustomFieldValue($params['serviceid'], 'Server ID');
        $api = new DedicatedServer\ApiClient($params);
        $api->suspendServer($serverId);
        return 'success';
    } catch (\Exception $e) {
        return 'Error: ' . $e->getMessage();
    }
}

function dedicated_server_UnsuspendAccount(array $params): string {
    try {
        $serverId = getCustomFieldValue($params['serviceid'], 'Server ID');
        $api = new DedicatedServer\ApiClient($params);
        $api->unsuspendServer($serverId);
        return 'success';
    } catch (\Exception $e) {
        return 'Error: ' . $e->getMessage();
    }
}

function dedicated_server_TerminateAccount(array $params): string {
    try {
        $serverId = getCustomFieldValue($params['serviceid'], 'Server ID');
        $api = new DedicatedServer\ApiClient($params);
        $api->terminateServer($serverId);
        return 'success';
    } catch (\Exception $e) {
        return 'Error: ' . $e->getMessage();
    }
}

function dedicated_server_ChangePassword(array $params): string {
    try {
        $serverId = getCustomFieldValue($params['serviceid'], 'Server ID');
        $api = new DedicatedServer\ApiClient($params);
        $newPassword = $api->resetPassword($serverId, $params['password']);
        saveCustomFieldValue($params['serviceid'], 'Root Password', encryptValue($newPassword));
        return 'success';
    } catch (\Exception $e) {
        return 'Error: ' . $e->getMessage();
    }
}

function dedicated_server_ChangePackage(array $params): string {
    try {
        $serverId = getCustomFieldValue($params['serviceid'], 'Server ID');
        $api = new DedicatedServer\ApiClient($params);
        $api->updateServer($serverId, [
            'type' => $params['configoption1'],
            'ram' => $params['configoption3'],
            'bandwidth' => $params['configoption5'],
        ]);
        return 'success';
    } catch (\Exception $e) {
        return 'Error: ' . $e->getMessage();
    }
}

function dedicated_server_TestConnection(array $params): array {
    try {
        $api = new DedicatedServer\ApiClient($params);
        $api->ping();
        return ['success' => true, 'error' => ''];
    } catch (\Exception $e) {
        return ['success' => false, 'error' => $e->getMessage()];
    }
}

function dedicated_server_AdminServices(array $params): array {
    return [
        'Server ID' => getCustomFieldValue($params['serviceid'], 'Server ID') ?: 'N/A',
        'Primary IP' => getCustomFieldValue($params['serviceid'], 'Primary IP') ?: 'N/A',
        'Location' => $params['configoption7'],
        'Server Type' => $params['configoption1'],
        'Storage' => $params['configoption4'],
        'RAID' => $params['configoption6'],
    ];
}

function dedicated_server_ClientArea(array $params): array {
    $serverId = getCustomFieldValue($params['serviceid'], 'Server ID');
    $primaryIp = getCustomFieldValue($params['serviceid'], 'Primary IP');
    $kvmIp = getCustomFieldValue($params['serviceid'], 'KVM IP');
    $kvmPort = getCustomFieldValue($params['serviceid'], 'KVM Port');
    
    $metrics = [];
    try {
        $api = new DedicatedServer\ApiClient($params);
        $metrics = $api->getServerMetrics($serverId);
    } catch (\Exception $e) {}

    return [
        'pagetitle' => 'Dedicated Server - ' . ($params['domain'] ?: 'Server'),
        'templatefile' => 'templates/clientarea',
        'vars' => [
            'server_id' => $serverId,
            'primary_ip' => $primaryIp,
            'kvm_ip' => $kvmIp,
            'kvm_port' => $kvmPort,
            'metrics' => $metrics,
            'status' => $params['status'],
            'config' => $params,
        ],
    ];
}

function dedicated_server_ClientAreaAllowedFunctions(): array {
    return [
        'RebootServer' => 'Reboot Server',
        'PowerCycle' => 'Power Cycle',
        'ReinstallOS' => 'Reinstall OS',
        'OpenKVM' => 'KVM Console',
    ];
}

function encryptValue(string $value): string {
    return base64_encode($value);
}
```

## API Client: lib/ApiClient.php

```php
<?php
namespace DedicatedServer;

class ApiClient {
    private string $baseUrl;
    private string $apiKey;
    private int $timeout = 60;

    public function __construct(array $params) {
        $this->baseUrl = rtrim($params['serverhost'] ?? '', '/');
        $this->apiKey = $params['serveraccesshash'] ?? '';
    }

    public function request(string $method, string $endpoint, array $data = []): array {
        $ch = curl_init();
        curl_setopt_array($ch, [
            CURLOPT_URL => $this->baseUrl . $endpoint,
            CURLOPT_RETURNTRANSFER => true,
            CURLOPT_TIMEOUT => $this->timeout,
            CURLOPT_HTTPHEADER => [
                'Authorization: Bearer ' . $this->apiKey,
                'Content-Type: application/json',
            ],
        ]);

        if ($method === 'POST') {
            curl_setopt($ch, CURLOPT_POST, true);
            curl_setopt($ch, CURLOPT_POSTFIELDS, json_encode($data));
        }

        $response = curl_exec($ch);
        $httpCode = curl_getinfo($ch, CURLINFO_HTTP_CODE);
        curl_close($ch);

        $result = json_decode($response, true) ?? [];
        
        if ($httpCode >= 400) {
            throw new \Exception($result['message'] ?? "HTTP {$httpCode}");
        }

        return $result['data'] ?? $result;
    }

    public function ping(): array {
        return $this->request('GET', '/api/v1/ping');
    }

    public function createServer(array $data): array {
        return $this->request('POST', '/api/v1/servers', $data);
    }

    public function getServerMetrics(string $serverId): array {
        return $this->request('GET', "/api/v1/servers/{$serverId}/metrics");
    }

    public function suspendServer(string $serverId): array {
        return $this->request('POST', "/api/v1/servers/{$serverId}/suspend");
    }

    public function unsuspendServer(string $serverId): array {
        return $this->request('POST', "/api/v1/servers/{$serverId}/unsuspend");
    }

    public function terminateServer(string $serverId): array {
        return $this->request('DELETE', "/api/v1/servers/{$serverId}");
    }

    public function resetPassword(string $serverId, string $password): string {
        $result = $this->request('POST', "/api/v1/servers/{$serverId}/password", [
            'password' => $password,
        ]);
        return $result['password'] ?? '';
    }

    public function updateServer(string $serverId, array $data): array {
        return $this->request('PUT', "/api/v1/servers/{$serverId}", $data);
    }

    public function rebootServer(string $serverId): array {
        return $this->request('POST', "/api/v1/servers/{$serverId}/reboot");
    }

    public function powerCycle(string $serverId): array {
        return $this->request('POST', "/api/v1/servers/{$serverId}/power/cycle");
    }

    public function reinstallOS(string $serverId, string $image): array {
        return $this->request('POST', "/api/v1/servers/{$serverId}/reinstall", [
            'image' => $image,
        ]);
    }
}
```

## Client Area Template

```smarty
<div class="dedicated-server-client">
    <div class="server-header">
        <h2><i class="fa fa-server"></i> Dedicated Server</h2>
        <span class="badge badge-{$status|lower}">{$status}</span>
    </div>

    <div class="server-info-grid">
        <div class="info-card">
            <h4><i class="fa fa-network-wired"></i> Network</h4>
            <div class="info-row">
                <label>Server ID:</label>
                <span class="value">{$server_id}</span>
            </div>
            <div class="info-row">
                <label>Primary IP:</label>
                <span class="value"><code>{$primary_ip}</code></span>
            </div>
            <div class="info-row">
                <label>Location:</label>
                <span class="value">{$config.configoption7}</span>
            </div>
        </div>

        <div class="info-card">
            <h4><i class="fa fa-cogs"></i> Hardware</h4>
            <div class="info-row">
                <label>Type:</label>
                <span class="value">{$config.configoption1|upper}</span>
            </div>
            <div class="info-row">
                <label>Storage:</label>
                <span class="value">{$config.configoption4}</span>
            </div>
            <div class="info-row">
                <label>RAID:</label>
                <span class="value">{if $config.configoption6 eq 'none'}No RAID{else}{$config.configoption6}{/if}</span>
            </div>
        </div>
    </div>

    {if $metrics}
    <div class="server-metrics">
        <h4><i class="fa fa-chart-line"></i> Resource Usage</h4>
        <div class="metrics-grid">
            <div class="metric-box">
                <div class="metric-label">CPU</div>
                <div class="metric-bar"><div class="metric-fill" style="width: {$metrics.cpu}%"></div></div>
                <div class="metric-value">{$metrics.cpu}%</div>
            </div>
            <div class="metric-box">
                <div class="metric-label">RAM ({$metrics.ram_used}GB / {$metrics.ram_total}GB)</div>
                <div class="metric-bar"><div class="metric-fill" style="width: {($metrics.ram_used/$metrics.ram_total)*100}%"></div></div>
            </div>
            <div class="metric-box">
                <div class="metric-label">Disk</div>
                <div class="metric-bar"><div class="metric-fill" style="width: {$metrics.disk}%"></div></div>
                <div class="metric-value">{$metrics.disk}%</div>
            </div>
        </div>
        <div class="bandwidth-info">
            Bandwidth: {$metrics.bandwidth_used} TB / {$metrics.bandwidth_limit}
        </div>
    </div>
    {/if}

    <div class="server-actions">
        <button class="btn btn-primary" onclick="serverAction('reboot')">
            <i class="fa fa-sync"></i> Reboot
        </button>
        <button class="btn" onclick="serverAction('power')">
            <i class="fa fa-power-off"></i> Power Cycle
        </button>
        <button class="btn" onclick="serverAction('kvm')">
            <i class="fa fa-desktop"></i> KVM Console
        </button>
        <button class="btn" onclick="serverAction('reinstall')">
            <i class="fa fa-redo"></i> Reinstall OS
        </button>
    </div>
</div>

<style>
.dedicated-server-client { padding: 20px; }
.server-header { display: flex; justify-content: space-between; align-items: center; margin-bottom: 20px; }
.server-info-grid { display: grid; grid-template-columns: repeat(auto-fit, minmax(300px, 1fr)); gap: 20px; margin-bottom: 20px; }
.info-card { background: #f8f9fa; border-radius: 8px; padding: 15px; }
.info-card h4 { margin: 0 0 15px 0; }
.info-row { display: flex; justify-content: space-between; padding: 8px 0; border-bottom: 1px solid #eee; }
.server-metrics { background: #f8f9fa; border-radius: 8px; padding: 15px; margin-bottom: 20px; }
.metrics-grid { display: grid; gap: 15px; }
.metric-box { }
.metric-bar { height: 8px; background: #e0e0e0; border-radius: 4px; overflow: hidden; margin-top: 5px; }
.metric-fill { height: 100%; background: linear-gradient(90deg, #28a745, #ffc107, #dc3545); }
.server-actions { display: flex; gap: 10px; flex-wrap: wrap; }
</style>
```

## Required Custom Fields

| Field Name | Type | Description |
|------------|------|-------------|
| Server ID | Text | Server identifier |
| Primary IP | Text | Primary IPv4 address |
| Switch Port | Text | Switch port |
| Root Password | Password | Encrypted root password |
| KVM IP | Text | KVM over IP address |
| KVM Port | Text | KVM port |
| Additional IPs | Text | Number of additional IPs |
