# WHMCS VPN Provisioning Module DevKit
# Version: 1.0 | Updated: 2026-05-29

## DevKit Structure

```
devkits/whmcs-vpn-provisioning/
├── vpn_provisioning.php       # Main provisioning module
├── lib/
│   └── ApiClient.php         # VPN provider API client
├── templates/
│   └── clientarea.tpl         # Client area template
└── DEVKIT.md                  # This file
```

## Module Code

```php
<?php
/**
 * WHMCS VPN Provisioning Module
 */

if (!defined("WHMCS")) {
    die("Direct access denied");
}

function vpn_provisioning_MetaData(): array {
    return [
        'DisplayName' => 'VPN Service',
        'APIVersion' => '1.1',
        'RequiresServer' => true,
        'DefaultNonSSLPort' => 443,
        'Parameters' => ['server_username', 'server_password', 'server_access_hash'],
    ];
}

function vpn_provisioning_ConfigOptions(array $params): array {
    return [
        'vpn_protocol' => [
            'Type' => 'dropdown',
            'Options' => 'openvpn,wireguard,ikev2,wireguard-openvpn',
            'Default' => 'wireguard',
            'Description' => 'VPN protocol',
        ],
        'server_location' => [
            'Type' => 'dropdown',
            'Options' => 'us-east,us-west,eu-central,eu-west,uk,london,ap-singapore,ap-tokyo,au-sydney',
            'Default' => 'us-east',
            'Description' => 'Server location',
        ],
        'max_devices' => [
            'Type' => 'dropdown',
            'Options' => '1,3,5,10,unlimited',
            'Default' => '5',
            'Description' => 'Max concurrent devices',
        ],
        'enable_kill_switch' => [
            'Type' => 'yesno',
            'Description' => 'Enable kill switch',
        ],
        'enable_split_tunnel' => [
            'Type' => 'yesno',
            'Description' => 'Enable split tunneling',
        ],
        'bandwidth_limit' => [
            'Type' => 'dropdown',
            'Options' => '10gb,100gb,1tb,unlimited',
            'Default' => 'unlimited',
            'Description' => 'Monthly bandwidth',
        ],
    ];
}

function vpn_provisioning_CreateAccount(array $params): string {
    try {
        $api = new VpnProvisioning\ApiClient($params);
        
        $accountData = [
            'username' => $params['username'] ?: generateVpnUsername($params),
            'password' => $params['password'] ?? '',
            'protocol' => $params['configoption1'],
            'location' => $params['configoption2'],
            'max_devices' => $params['configoption3'] === 'unlimited' ? 999 : (int) $params['configoption3'],
            'kill_switch' => ($params['configoption4'] === 'on'),
            'split_tunnel' => ($params['configoption5'] === 'on'),
        ];

        $result = $api->createAccount($accountData);

        saveCustomFieldValue($params['serviceid'], 'VPN Account ID', $result['account_id']);
        saveCustomFieldValue($params['serviceid'], 'VPN Username', $result['username']);
        saveCustomFieldValue($params['serviceid'], 'Server Config', base64_encode($result['config']));
        saveCustomFieldValue($params['serviceid'], 'VPN IP', $result['server_ip']);
        saveCustomFieldValue($params['serviceid'], 'Protocol', $params['configoption1']);

        logActivity("VPN Provisioning: Created account for service {$params['serviceid']}");
        return 'success';
        
    } catch (\Exception $e) {
        logActivity("VPN Provisioning CreateAccount Error: " . $e->getMessage());
        return 'Error: ' . $e->getMessage();
    }
}

function vpn_provisioning_SuspendAccount(array $params): string {
    try {
        $accountId = getCustomFieldValue($params['serviceid'], 'VPN Account ID');
        $api = new VpnProvisioning\ApiClient($params);
        $api->suspendAccount($accountId);
        return 'success';
    } catch (\Exception $e) {
        return 'Error: ' . $e->getMessage();
    }
}

function vpn_provisioning_UnsuspendAccount(array $params): string {
    try {
        $accountId = getCustomFieldValue($params['serviceid'], 'VPN Account ID');
        $api = new VpnProvisioning\ApiClient($params);
        $api->unsuspendAccount($accountId);
        return 'success';
    } catch (\Exception $e) {
        return 'Error: ' . $e->getMessage();
    }
}

function vpn_provisioning_TerminateAccount(array $params): string {
    try {
        $accountId = getCustomFieldValue($params['serviceid'], 'VPN Account ID');
        $api = new VpnProvisioning\ApiClient($params);
        $api->terminateAccount($accountId);
        return 'success';
    } catch (\Exception $e) {
        return 'Error: ' . $e->getMessage();
    }
}

function vpn_provisioning_ChangePassword(array $params): string {
    try {
        $accountId = getCustomFieldValue($params['serviceid'], 'VPN Account ID');
        $api = new VpnProvisioning\ApiClient($params);
        $api->updatePassword($accountId, $params['password']);
        return 'success';
    } catch (\Exception $e) {
        return 'Error: ' . $e->getMessage();
    }
}

function vpn_provisioning_ChangePackage(array $params): string {
    try {
        $accountId = getCustomFieldValue($params['serviceid'], 'VPN Account ID');
        $api = new VpnProvisioning\ApiClient($params);
        $api->updateAccount($accountId, [
            'protocol' => $params['configoption1'],
            'location' => $params['configoption2'],
            'max_devices' => $params['configoption3'] === 'unlimited' ? 999 : (int) $params['configoption3'],
        ]);
        return 'success';
    } catch (\Exception $e) {
        return 'Error: ' . $e->getMessage();
    }
}

function vpn_provisioning_TestConnection(array $params): array {
    try {
        $api = new VpnProvisioning\ApiClient($params);
        $api->ping();
        return ['success' => true, 'error' => ''];
    } catch (\Exception $e) {
        return ['success' => false, 'error'': $e->getMessage()];
    }
}

function vpn_provisioning_AdminServices(array $params): array {
    return [
        'Account ID' => getCustomFieldValue($params['serviceid'], 'VPN Account ID') ?: 'N/A',
        'Protocol' => $params['configoption1'],
        'Location' => $params['configoption2'],
        'Max Devices' => $params['configoption3'],
        'Kill Switch' => ($params['configoption4'] === 'on') ? 'Yes' : 'No',
    ];
}

function vpn_provisioning_ClientArea(array $params): array {
    $vpnUsername = getCustomFieldValue($params['serviceid'], 'VPN Username');
    $serverConfig = getCustomFieldValue($params['serviceid'], 'Server Config');
    $protocol = getCustomFieldValue($params['serviceid'], 'Protocol');
    
    $usage = [];
    try {
        $api = new VpnProvisioning\ApiClient($params);
        $accountId = getCustomFieldValue($params['serviceid'], 'VPN Account ID');
        $usage = $api->getUsage($accountId);
    } catch (\Exception $e) {}

    return [
        'pagetitle' => 'VPN Service - ' . ($params['domain'] ?: 'VPN'),
        'templatefile' => 'templates/clientarea',
        'vars' => [
            'vpn_username' => $vpnUsername,
            'vpn_password' => decryptValue($params['password'] ?? ''),
            'config' => base64_decode($serverConfig),
            'protocol' => $protocol,
            'usage' => $usage,
            'status' => $params['status'],
            'config_options' => $params,
        ],
    ];
}

function vpn_provisioning_ClientAreaAllowedFunctions(): array {
    return [
        'DownloadConfig' => 'Download Config',
        'GenerateQrCode' => 'QR Code',
        'ViewUsageStats' => 'View Usage',
    ];
}

function generateVpnUsername(array $params): string {
    return 'vpn' . substr(md5($params['serviceid']), 0, 8);
}

function decryptValue(string $value): string {
    return $value;
}
```

## API Client: lib/ApiClient.php

```php
<?php
namespace VpnProvisioning;

class ApiClient {
    private string $baseUrl;
    private string $apiKey;
    private int $timeout = 30;

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

    public function createAccount(array $data): array {
        return $this->request('POST', '/api/v1/accounts', $data);
    }

    public function getUsage(string $accountId): array {
        return $this->request('GET', "/api/v1/accounts/{$accountId}/usage");
    }

    public function suspendAccount(string $accountId): array {
        return $this->request('POST', "/api/v1/accounts/{$accountId}/suspend");
    }

    public function unsuspendAccount(string $accountId): array {
        return $this->request('POST', "/api/v1/accounts/{$accountId}/unsuspend");
    }

    public function terminateAccount(string $accountId): array {
        return $this->request('DELETE', "/api/v1/accounts/{$accountId}");
    }

    public function updatePassword(string $accountId, string $password): array {
        return $this->request('PUT', "/api/v1/accounts/{$accountId}/password", ['password' => $password]);
    }

    public function updateAccount(string $accountId, array $updates): array {
        return $this->request('PUT', "/api/v1/accounts/{$accountId}", $updates);
    }

    public function getConfig(string $accountId): array {
        return $this->request('GET', "/api/v1/accounts/{$accountId}/config");
    }

    public function getQrCode(string $accountId): string {
        return $this->request('GET', "/api/v1/accounts/{$accountId}/qrcode")['qrcode'];
    }
}
```

## Client Area Template

```smarty
<div class="vpn-client">
    <div class="vpn-header">
        <h2><i class="fa fa-shield-alt"></i> VPN Service</h2>
        <span class="badge badge-{$status|lower}">{$status}</span>
    </div>

    <div class="vpn-credentials">
        <h4><i class="fa fa-key"></i> Connection Credentials</h4>
        <div class="credential-row">
            <label>Username:</label>
            <code>{$vpn_username}</code>
        </div>
        <div class="credential-row">
            <label>Protocol:</label>
            <span class="badge">{$protocol|upper}</span>
        </div>
        <div class="credential-row">
            <label>Server:</label>
            <code>{$config_options.configoption2}</code>
        </div>
    </div>

    <div class="vpn-config">
        <h4><i class="fa fa-file-code"></i> Configuration File</h4>
        <p>Download the configuration file for your VPN client:</p>
        <pre>{$config|escape:'html'}</pre>
        <div class="config-actions">
            <a href="?m=vpn_provisioning&action=download&id={$service.id}" class="btn btn-primary">
                <i class="fa fa-download"></i> Download Config
            </a>
            <a href="?m=vpn_provisioning&action=qrcode&id={$service.id}" class="btn">
                <i class="fa fa-qrcode"></i> QR Code
            </a>
        </div>
    </div>

    {if $usage}
    <div class="vpn-usage">
        <h4><i class="fa fa-chart-line"></i> Usage Statistics</h4>
        <div class="usage-grid">
            <div class="usage-stat">
                <div class="stat-value">{$usage.bandwidth_used|bytes}</div>
                <div class="stat-label">Bandwidth Used</div>
                <div class="usage-bar"><div class="usage-fill" style="width: {min(100, ($usage.bandwidth_used/$usage.bandwidth_limit)*100)}%"></div></div>
            </div>
            <div class="usage-stat">
                <div class="stat-value">{$usage.connected_devices}</div>
                <div class="stat-label">Connected Devices</div>
            </div>
        </div>
        <div class="usage-footer">
            <small>Cycle resets on: {$usage.reset_date|date_format:'%Y-%m-%d'}</small>
        </div>
    </div>
    {/if}

    <div class="vpn-setup">
        <h4><i class="fa fa-book"></i> Setup Guides</h4>
        <div class="guide-links">
            <a href="https://docs.example.com/vpn/wireguard" target="_blank" class="guide-link">WireGuard Setup</a>
            <a href="https://docs.example.com/vpn/openvpn" target="_blank" class="guide-link">OpenVPN Setup</a>
            <a href="https://docs.example.com/vpn/ikev2" target="_blank" class="guide-link">IKEv2 Setup</a>
        </div>
    </div>
</div>

<style>
.vpn-client { padding: 20px; }
.vpn-header { display: flex; justify-content: space-between; align-items: center; margin-bottom: 20px; }
.vpn-credentials { background: #f8f9fa; border-radius: 8px; padding: 15px; margin-bottom: 20px; }
.credential-row { display: flex; justify-content: space-between; padding: 8px 0; border-bottom: 1px solid #eee; }
.vpn-config { background: #f8f9fa; border-radius: 8px; padding: 15px; margin-bottom: 20px; }
.vpn-config pre { background: #2d2d2d; color: #f8f8f2; padding: 15px; border-radius: 4px; overflow-x: auto; font-size: 12px; }
.config-actions { display: flex; gap: 10px; margin-top: 15px; }
.vpn-usage { background: #f8f9fa; border-radius: 8px; padding: 15px; margin-bottom: 20px; }
.usage-grid { display: grid; grid-template-columns: 1fr 1fr; gap: 20px; }
.usage-stat { text-align: center; }
.stat-value { font-size: 24px; font-weight: bold; }
.usage-bar { height: 8px; background: #e0e0e0; border-radius: 4px; overflow: hidden; margin-top: 10px; }
.usage-fill { height: 100%; background: #28a745; }
.vpn-setup { background: #f8f9fa; border-radius: 8px; padding: 15px; }
.guide-links { display: flex; gap: 15px; margin-top: 10px; }
.guide-link { padding: 8px 16px; background: white; border-radius: 4px; text-decoration: none; }
</style>
```

## Required Custom Fields

| Field Name | Type | Description |
|------------|------|-------------|
| VPN Account ID | Text | Account identifier |
| VPN Username | Text | VPN username |
| Server Config | Password | Base64 encoded config |
| VPN IP | Text | Server IP |
| Protocol | Text | VPN protocol |
