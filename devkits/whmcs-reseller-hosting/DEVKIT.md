# WHMCS Reseller Hosting Provisioning Module DevKit
# Version: 1.0 | Updated: 2026-05-29

## DevKit Structure

```
devkits/whmcs-reseller-hosting/
├── reseller_hosting.php       # Main provisioning module
├── lib/
│   └── ApiClient.php         # Reseller panel API client
├── templates/
│   └── clientarea.tpl         # Client area template
└── DEVKIT.md                  # This file
```

## Module Code

```php
<?php
/**
 * WHMCS Reseller Hosting Provisioning Module
 */

if (!defined("WHMCS")) {
    die("Direct access denied");
}

function reseller_hosting_MetaData(): array {
    return [
        'DisplayName' => 'Reseller Hosting',
        'APIVersion' => '1.1',
        'RequiresServer' => true,
        'DefaultNonSSLPort' => 443,
        'Parameters' => ['server_username', 'server_password', 'server_access_hash'],
    ];
}

function reseller_hosting_ConfigOptions(array $params): array {
    return [
        'reseller_tier' => [
            'Type' => 'dropdown',
            'Options' => 'starter,bronze,silver,gold,platinum',
            'Default' => 'bronze',
            'Description' => 'Reseller account tier',
        ],
        'max_domains' => [
            'Type' => 'text',
            'Default' => '50',
            'Description' => 'Maximum domains',
        ],
        'max_disk_gb' => [
            'Type' => 'text',
            'Default' => '500',
            'Description' => 'Disk space (GB)',
        ],
        'max_bandwidth_gb' => [
            'Type' => 'text',
            'Default' => '1000',
            'Description' => 'Bandwidth (GB)',
        ],
        'max_mailboxes' => [
            'Type' => 'text',
            'Default' => '100',
            'Description' => 'Maximum mailboxes',
        ],
        'allow_ssh' => [
            'Type' => 'yesno',
            'Description' => 'Enable SSH access',
        ],
        'allow_whm' => [
            'Type' => 'yesno',
            'Description' => 'Provide WHM access',
        ],
        'include_license' => [
            'Type' => 'yesno',
            'Description' => 'Include cPanel license',
        ],
    ];
}

function reseller_hosting_CreateAccount(array $params): string {
    try {
        $api = new ResellerHosting\ApiClient($params);
        
        $result = $api->createReseller([
            'username' => generateUsername($params),
            'password' => $params['password'] ?? '',
            'domain' => $params['domain'] ?: 'reseller-' . $params['serviceid'],
            'tier' => $params['configoption1'],
            'max_domains' => (int) $params['configoption2'],
            'max_disk' => (int) $params['configoption3'] * 1024,
            'max_bandwidth' => (int) $params['configoption4'] * 1024,
            'max_mailboxes' => (int) $params['configoption5'],
            'ssh_access' => ($params['configoption6'] === 'on'),
            'whm_access' => ($params['configoption7'] === 'on'),
            'cpanel_license' => ($params['configoption8'] === 'on'),
        ]);

        saveCustomFieldValue($params['serviceid'], 'Reseller Username', $result['username']);
        saveCustomFieldValue($params['serviceid'], 'cPanel URL', $result['cpanel_url']);
        saveCustomFieldValue($params['serviceid'], 'WHM URL', $result['whm_url']);
        saveCustomFieldValue($params['serviceid'], 'Nameservers', implode(',', $result['nameservers']));

        logActivity("ResellerHosting: Created reseller account for service {$params['serviceid']}");
        return 'success';
        
    } catch (\Exception $e) {
        logActivity("ResellerHosting CreateAccount Error: " . $e->getMessage());
        return 'Error Connecting to Reseller API: ' . $e->getMessage();
    }
}

function reseller_hosting_SuspendAccount(array $params): string {
    try {
        $username = getCustomFieldValue($params['serviceid'], 'Reseller Username');
        $api = new ResellerHosting\ApiClient($params);
        $api->suspendReseller($username);
        return 'success';
    } catch (\Exception $e) {
        return 'Error: ' . $e->getMessage();
    }
}

function reseller_hosting_UnsuspendAccount(array $params): string {
    try {
        $username = getCustomFieldValue($params['serviceid'], 'Reseller Username');
        $api = new ResellerHosting\ApiClient($params);
        $api->unsuspendReseller($username);
        return 'success';
    } catch (\Exception $e) {
        return 'Error: ' . $e->getMessage();
    }
}

function reseller_hosting_TerminateAccount(array $params): string {
    try {
        $username = getCustomFieldValue($params['serviceid'], 'Reseller Username');
        $api = new ResellerHosting\ApiClient($params);
        $api->terminateReseller($username);
        return 'success';
    } catch (\Exception $e) {
        return 'Error: ' . $e->getMessage();
    }
}

function reseller_hosting_ChangePassword(array $params): string {
    try {
        $username = getCustomFieldValue($params['serviceid'], 'Reseller Username');
        $api = new ResellerHosting\ApiClient($params);
        $api->updatePassword($username, $params['password']);
        return 'success';
    } catch (\Exception $e) {
        return 'Error: ' . $e->getMessage();
    }
}

function reseller_hosting_ChangePackage(array $params): string {
    try {
        $username = getCustomFieldValue($params['serviceid'], 'Reseller Username');
        $api = new ResellerHosting\ApiClient($params);
        $api->updateReseller($username, [
            'tier' => $params['configoption1'],
            'max_domains' => (int) $params['configoption2'],
            'max_disk' => (int) $params['configoption3'] * 1024,
            'max_bandwidth' => (int) $params['configoption4'] * 1024,
            'max_mailboxes' => (int) $params['configoption5'],
        ]);
        return 'success';
    } catch (\Exception $e) {
        return 'Error: ' . $e->getMessage();
    }
}

function reseller_hosting_TestConnection(array $params): array {
    try {
        $api = new ResellerHosting\ApiClient($params);
        $api->ping();
        return ['success' => true, 'error' => ''];
    } catch (\Exception $e) {
        return ['success' => false, 'error' => $e->getMessage()];
    }
}

function reseller_hosting_AdminServices(array $params): array {
    return [
        'Reseller Username' => getCustomFieldValue($params['serviceid'], 'Reseller Username') ?: 'N/A',
        'cPanel URL' => getCustomFieldValue($params['serviceid'], 'cPanel URL') ?: 'N/A',
        'Tier' => $params['configoption1'],
        'Max Domains' => $params['configoption2'],
        'Disk Limit' => $params['configoption3'] . ' GB',
        'Bandwidth Limit' => $params['configoption4'] . ' GB',
    ];
}

function reseller_hosting_ClientArea(array $params): array {
    $cpanelUrl = getCustomFieldValue($params['serviceid'], 'cPanel URL');
    $whmUrl = getCustomFieldValue($params['serviceid'], 'WHM URL');
    $nameservers = getCustomFieldValue($params['serviceid'], 'Nameservers');
    
    $usage = [];
    try {
        $api = new ResellerHosting\ApiClient($params);
        $username = getCustomFieldValue($params['serviceid'], 'Reseller Username');
        $usage = $api->getResellerUsage($username);
    } catch (\Exception $e) {}

    return [
        'pagetitle' => 'Reseller Hosting - ' . ($params['domain'] ?: 'Reseller Account'),
        'templatefile' => 'templates/clientarea',
        'vars' => [
            'cpanel_url' => $cpanelUrl,
            'whm_url' => $whmUrl,
            'nameservers' => $nameservers ? explode(',', $nameservers) : [],
            'usage' => $usage,
            'status' => $params['status'],
            'config' => $params,
        ],
    ];
}

function reseller_hosting_ClientAreaAllowedFunctions(): array {
    return [
        'CreateAccount' => 'Create Hosting Account',
        'ListAccounts' => 'View All Accounts',
    ];
}

function generateUsername(array $params): string {
    $username = preg_replace('/[^a-zA-Z0-9]/', '', strtolower($params['domain'] ?? 'reseller' . $params['serviceid']));
    return substr($username, 0, 8) . substr($params['serviceid'], -3);
}
```

## API Client: lib/ApiClient.php

```php
<?php
namespace ResellerHosting;

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

    public function createReseller(array $data): array {
        return $this->request('POST', '/api/v1/resellers', $data);
    }

    public function getResellerUsage(string $username): array {
        return $this->request('GET', "/api/v1/resellers/{$username}/usage");
    }

    public function suspendReseller(string $username): array {
        return $this->request('POST', "/api/v1/resellers/{$username}/suspend");
    }

    public function unsuspendReseller(string $username): array {
        return $this->request('POST', "/api/v1/resellers/{$username}/unsuspend");
    }

    public function terminateReseller(string $username): array {
        return $this->request('DELETE', "/api/v1/resellers/{$username}");
    }

    public function updatePassword(string $username, string $password): array {
        return $this->request('PUT', "/api/v1/resellers/{$username}/password", ['password' => $password]);
    }

    public function updateReseller(string $username, array $updates): array {
        return $this->request('PUT', "/api/v1/resellers/{$username}", $updates);
    }

    public function createAccount(string $resellerUsername, array $accountData): array {
        return $this->request('POST', "/api/v1/resellers/{$resellerUsername}/accounts", $accountData);
    }

    public function listAccounts(string $resellerUsername): array {
        return $this->request('GET', "/api/v1/resellers/{$resellerUsername}/accounts");
    }
}
```

## Client Area Template

```smarty
<div class="reseller-client">
    <div class="reseller-header">
        <h2><i class="fa fa-users"></i> Reseller Hosting</h2>
        <span class="badge badge-{$status|lower}">{$status}</span>
    </div>

    <div class="access-panel">
        <h4><i class="fa fa-external-link-alt"></i> Control Panels</h4>
        <div class="access-buttons">
            <a href="https://{$cpanel_url}" target="_blank" class="btn btn-primary">
                <i class="fa fa-cog"></i> cPanel
            </a>
            {if $whm_url}
            <a href="https://{$whm_url}" target="_blank" class="btn">
                <i class="fa fa-tachometer-alt"></i> WHM
            </a>
            {/if}
        </div>
    </div>

    <div class="nameservers-section">
        <h4><i class="fa fa-server"></i> Nameservers</h4>
        <p>Configure your domain to use these nameservers:</p>
        <div class="ns-list">
            {foreach $nameservers as $ns}
            <code>{$ns}</code>
            {/foreach}
        </div>
    </div>

    {if $usage}
    <div class="reseller-usage">
        <h4><i class="fa fa-chart-pie"></i> Resource Usage</h4>
        <div class="usage-stats">
            <div class="stat-row">
                <span>Domains</span>
                <span>{$usage.domains_used} / {$usage.domains_limit}</span>
            </div>
            <div class="usage-bar"><div class="usage-fill" style="width: {($usage.domains_used/$usage.domains_limit)*100}%"></div></div>
            
            <div class="stat-row">
                <span>Disk Space</span>
                <span>{$usage.disk_used} GB / {$usage.disk_limit} GB</span>
            </div>
            <div class="usage-bar"><div class="usage-fill" style="width: {($usage.disk_used/$usage.disk_limit)*100}%"></div></div>
            
            <div class="stat-row">
                <span>Bandwidth</span>
                <span>{$usage.bandwidth_used} GB / {$usage.bandwidth_limit} GB</span>
            </div>
            <div class="usage-bar"><div class="usage-fill" style="width: {($usage.bandwidth_used/$usage.bandwidth_limit)*100}%"></div></div>
        </div>
    </div>
    {/if}

    <div class="tier-info">
        <h4><i class="fa fa-star"></i> Your Reseller Tier</h4>
        <p>Current Tier: <strong>{$config.configoption1|upper}</strong></p>
        <ul>
            <li>Max Domains: {$config.configoption2}</li>
            <li>Disk Resolution: {$config.configoption3} GB</li>
            <li>Bandwidth: {$config.configoption4} GB</li>
            <li>Max Mailboxes: {$config.configoption5}</li>
        </ul>
    </div>
</div>

<style>
.reseller-client { padding: 20px; }
.reseller-header { display: flex; justify-content: space-between; align-items: center; margin-bottom: 20px; }
.access-panel { background: #f8f9fa; border-radius: 8px; padding: 15px; margin-bottom: 20px; }
.access-buttons { display: flex; gap: 10px; }
.nameservers-section { background: #fff3cd; border-radius: 8px; padding: 15px; margin-bottom: 20px; }
.ns-list code { display: block; background: white; padding: 8px; margin: 5px 0; border-radius: 4px; }
.reseller-usage { background: #f8f9fa; border-radius: 8px; padding: 15px; margin-bottom: 20px; }
.usage-stats .stat-row { display: flex; justify-content: space-between; margin-bottom: 5px; }
.usage-bar { height: 6px; background: #e0e0e0; border-radius: 3px; overflow: hidden; margin-bottom: 15px; }
.usage-fill { height: 100%; background: #28a745; }
.tier-info { background: #f8f9fa; border-radius: 8px; padding: 15px; }
.tier-info ul { list-style: none; padding: 0; margin: 10px 0 0 0; }
.tier-info li { padding: 5px 0; }
</style>
```

## Required Custom Fields

| Field Name | Type | Description |
|------------|------|-------------|
| Reseller Username | Text | System username |
| cPanel URL | Text | cPanel login URL |
| WHM URL | Text | WHM login URL |
| Nameservers | Text | Comma-separated NS records |
