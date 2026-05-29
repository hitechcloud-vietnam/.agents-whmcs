# WHMCS Bacula Backup Provisioning Module DevKit
# Version: 1.0 | Updated: 2026-05-29

## DevKit Structure

```
devkits/whmcs-bacula-backup-provisioning/
├── bacula_backup.php          # Main provisioning module
├── lib/
│   └── ApiClient.php         # Bacula API client
├── templates/
│   └── clientarea.tpl         # Client area template
└── DEVKIT.md                  # This file
```

## Module Code

```php
<?php
/**
 * WHMCS Bacula Backup Provisioning Module
 */

if (!defined("WHMCS")) {
    die("Direct access denied");
}

function bacula_backup_MetaData(): array {
    return [
        'DisplayName' => 'Backup Service',
        'APIVersion' => '1.1',
        'RequiresServer' => true,
        'DefaultNonSSLPort' => 443,
        'Parameters' => ['server_username', 'server_password', 'server_access_hash'],
    ];
}

function bacula_backup_ConfigOptions(array $params): array {
    return [
        'backup_tier' => [
            'Type' => 'dropdown',
            'Options' => 'basic,standard,premium,enterprise',
            'Default' => 'standard',
            'Description' => 'Backup tier',
        ],
        'storage_size_gb' => [
            'Type' => 'dropdown',
            'Options' => '100gb,250gb,500gb,1tb,2tb,5tb,10tb',
            'Default' => '500gb',
            'Description' => 'Storage allocation',
        ],
        'retention_days' => [
            'Type' => 'dropdown',
            'Options' => '7,14,30,60,90,180,365',
            'Default' => '30',
            'Description' => 'Retention period (days)',
        ],
        'backup_schedule' => [
            'Type' => 'dropdown',
            'Options' => 'daily,twice-daily,hourly,weekly',
            'Default' => 'daily',
            'Description' => 'Backup frequency',
        ],
        'compression' => [
            'Type' => 'dropdown',
            'Options' => 'none,gzip,lzo,xz',
            'Default' => 'gzip',
            'Description' => 'Compression algorithm',
        ],
        'encryption' => [
            'Type' => 'yesno',
            'Description' => 'Enable encryption at rest',
        ],
    ];
}

function bacula_backup_CreateAccount(array $params): string {
    try {
        $api = new BaculaBackup\ApiClient($params);
        
        $result = $api->createClient([
            'client_name' => generateBackupClientName($params),
            'storage_quota' => convertToBytes($params['configoption2']),
            'retention_days' => (int) $params['configoption3'],
            'schedule' => $params['configoption4'],
            'compression' => $params['configoption5'],
            'encryption' => ($params['configoption6'] === 'on'),
        ]);

        saveCustomFieldValue($params['serviceid'], 'Backup Client ID', $result['client_id']);
        saveCustomFieldValue($params['serviceid'], 'Storage Pool', $result['pool_name']);
        saveCustomFieldValue($params['serviceid'], 'Director Address', $result['director_address']);
        saveCustomFieldValue($params['serviceid'], 'Director Port', $result['director_port']);
        saveCustomFieldValue($params['serviceid'], 'Client Secret', encryptValue($result['client_secret']));

        logActivity("BaculaBackup: Created backup client for service {$params['serviceid']}");
        return 'success';
        
    } catch (\Exception $e) {
        logActivity("BaculaBackup CreateAccount Error: " . $e->getMessage());
        return 'Error: ' . $e->getMessage();
    }
}

function bacula_backup_SuspendAccount(array $params): string {
    try {
        $clientId = getCustomFieldValue($params['serviceid'], 'Backup Client ID');
        $api = new BaculaBackup\ApiClient($params);
        $api->disableClient($clientId);
        return 'success';
    } catch (\Exception $e) {
        return 'Error: ' . $e->getMessage();
    }
}

function bacula_backup_UnsuspendAccount(array $params): string {
    try {
        $clientId = getCustomFieldValue($params['serviceid'], 'Backup Client ID');
        $api = new BaculaBackup\ApiClient($params);
        $api->enableClient($clientId);
        return 'success';
    } catch (\Exception $e) {
        return 'Error: ' . $e->getMessage();
    }
}

function bacula_backup_TerminateAccount(array $params): string {
    try {
        $clientId = getCustomFieldValue($params['serviceid'], 'Backup Client ID');
        $api = new BaculaBackup\ApiClient($params);
        $api->deleteClient($clientId);
        return 'success';
    } catch (\Exception $e) {
        return 'Error: ' . $e->getMessage();
    }
}

function bacula_backup_ChangePassword(array $params): string {
    try {
        $clientId = getCustomFieldValue($params['serviceid'], 'Backup Client ID');
        $api = new BaculaBackup\ApiClient($params);
        $newSecret = $api->rotateSecret($clientId);
        saveCustomFieldValue($params['serviceid'], 'Client Secret', encryptValue($newSecret));
        return 'success';
    } catch (\Exception $e) {
        return 'Error: ' . $e->getMessage();
    }
}

function bacula_backup_ChangePackage(array $params): string {
    try {
        $clientId = getCustomFieldValue($params['serviceid'], 'Backup Client ID');
        $api = new BaculaBackup\ApiClient($params);
        $api->updateClient($clientId, [
            'storage_quota' => convertToBytes($params['configoption2']),
            'retention_days' => (int) $params['configoption3'],
            'schedule' => $params['configoption4'],
        ]);
        return 'success';
    } catch (\Exception $e) {
        return 'Error: ' . $e->getMessage();
    }
}

function bacula_backup_TestConnection(array $params): array {
    try {
        $api = new BaculaBackup\ApiClient($params);
        $api->ping();
        return ['success' => true, 'error' => ''];
    } catch (\Exception $e) {
        return ['success' => false, 'error' => $e->getMessage()];
    }
}

function bacula_backup_AdminVariables(array $params): array {
    return [
        'Backup Client ID' => getCustomFieldValue($params['serviceid'], 'Backup Client ID') ?: 'N/A',
        'Director Address' => getCustomFieldValue($params['serviceid'], 'Director Address') ?: 'N/A',
        'Director Port' => getCustomFieldValue($params['serviceid'], 'Director Port') ?: 'N/A',
        'Storage Pool' => getCustomFieldValue($params['serviceid'], 'Storage Pool') ?: 'N/A',
        'Storage Quota' => $params['configoption2'],
        'Retention' => $params['configoption3'] . ' days',
        'Schedule' => $params['configoption4'],
    ];
}

function bacula_backup_ClientArea(array $params): array {
    $clientId = getCustomFieldValue($params['serviceid'], 'Backup Client ID');
    $directorAddress = getCustomFieldValue($params['serviceid'], 'Director Address');
    $directorPort = getCustomFieldValue($params['serviceid'], 'Director Port');
    
    $status = [];
    try {
        $api = new BaculaBackup\ApiClient($params);
        $status = $api->getClientStatus($clientId);
    } catch (\Exception $e) {}

    return [
        'pagetitle' => 'Backup Service - ' . ($params['domain'] ?: 'Backup'),
        'templatefile' => 'templates/clientarea',
        'vars' => [
            'client_id' => $clientId,
            'director_address' => $directorAddress,
            'director_port' => $directorPort,
            'storage_quota' => $params['configoption2'],
            'retention_days' => $params['configoption3'],
            'status' => $status,
            'service_status' => $params['status'],
        ],
    ];
}

function generateBackupClientName(array $params): string {
    return 'client-' . $params['serviceid'] . '-' . substr(md5($params['serviceid']), 0, 6);
}

function convertToBytes(string $size): int {
    $unit = strtolower(substr($size, -2));
    $value = (int) $size;
    return match($unit) {
        'tb' => $value * 1024 * 1024 * 1024 * 1024,
        'gb' => $value * 1024 * 1024 * 1024,
        'mb' => $value * 1024 * 1024,
        'kb' => $value * 1024,
        default => $value,
    };
}

function encryptValue(string $value): string {
    return base64_encode($value);
}
```

## API Client: lib/ApiClient.php

```php
<?php
namespace BaculaBackup;

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

    public function createClient(array $data): array {
        return $this->request('POST', '/api/v1/clients', $data);
    }

    public function getClientStatus(string $clientId): array {
        return $this->request('GET', "/api/v1/clients/{$clientId}/status");
    }

    public function enableClient(string $clientId): array {
        return $this->request('POST', "/api/v1/clients/{$clientId}/enable");
    }

    public function disableClient(string $clientId): array {
        return $this->request('POST', "/api/v1/clients/{$clientId}/disable");
    }

    public function deleteClient(string $clientId): array {
        return $this->request('DELETE', "/api/v1/clients/{$clientId}");
    }

    public function rotateSecret(string $clientId): string {
        $result = $this->request('POST', "/api/v1/clients/{$clientId}/rotate-secret");
        return $result['secret'];
    }

    public function updateClient(string $clientId, array $updates): array {
        return $this->request('PUT', "/api/v1/clients/{$clientId}", $updates);
    }

    public function listBackups(string $clientId): array {
        return $this->request('GET', "/api/v1/clients/{$clientId}/backups");
    }

    public function runBackup(string $clientId): array {
        return $this->request('POST', "/api/v1/clients/{$clientId}/backup");
    }

    public function restoreBackup(string $clientId, string $backupId, array $target): array {
        return $this->request('POST', "/api/v1/backups/{$backupId}/restore", [
            'target' => $target,
        ]);
    }
}
```

## Client Area Template

```smarty
<div class="backup-client">
    <div class="backup-header">
        <h2><i class="fa fa-database"></i> Backup Service</h2>
        <span class="badge badge-{$service_status|lower}">{$service_status}</span>
    </div>

    <div class="backup-info">
        <h4><i class="fa fa-info-circle"></i> Connection Details</h4>
        <div class="info-row">
            <label>Client ID:</label>
            <code>{$client_id}</code>
        </div>
        <div class="info-row">
            <label>Director:</label>
            <code>{$director_address}:{$director_port}</code>
        </div>
    </div>

    <div class="backup-config">
        <h4><i class="fa fa-cog"></i> Configuration</h4>
        <div class="config-grid">
            <div class="config-item">
                <span class="label">Storage Quota:</span>
                <span class="value">{$storage_quota}</span>
            </div>
            <div class="config-item">
                <span class="label">Retention:</span>
                <span class="value">{$retention_days} days</span>
            </div>
        </div>
    </div>

    {if $status}
    <div class="backup-status">
        <h4><i class="fa fa-chart-pie"></i> Backup Status</h4>
        <div class="status-stats">
            <div class="stat-box">
                <div class="stat-value">{$status.storage_used}</div>
                <div class="stat-label">Used ({$storage_quota})</div>
                <div class="usage-bar"><div class="usage-fill" style="width: {($status.storage_used_bytes/$status.storage_quota_bytes)*100}%"></div></div>
            </div>
            <div class="stat-box">
                <div class="stat-value">{$status.last_backup|date_format:'%Y-%m-d'}</div>
                <div class="stat-label">Last Backup</div>
            </div>
            <div class="stat-box">
                <div class="stat-value">{$status.total_backups}</div>
                <div class="stat-label">Total Backups</div>
            </div>
        </div>
    </div>
    {/if}

    <div class="backup-actions">
        <button class="btn btn-primary" onclick="runBackup()">
            <i class="fa fa-play"></i> Run Backup Now
        </button>
        <a href="?m=bacula_backup&action=list" class="btn">
            <i class="fa fa-list"></i> View Backups
        </a>
        <a href="?m=bacula_backup&action=restore" class="btn">
            <i class="fa fa-undo"></i> Restore
        </a>
    </div>
</div>

<style>
.backup-client { padding: 20px; }
.backup-header { display: flex; justify-content: space-between; align-items: center; margin-bottom: 20px; }
.backup-info, .backup-config { background: #f8f9fa; border-radius: 8px; padding: 15px; margin-bottom: 20px; }
.backup-status { background: #f8f9fa; border-radius: 8px; padding: 15px; margin-bottom: 20px; }
.info-row, .config-item { display: flex; justify-content: space-between; padding: 8px 0; }
.status-stats { display: grid; grid-template-columns: repeat(auto-fit, minmax(150px, 1fr)); gap: 20px; }
.stat-box { text-align: center; }
.stat-value { font-size: 20px; font-weight: bold; }
.usage-bar { height: 6px; background: #e0e0e0; border-radius: 3px; overflow: hidden; margin-top: 5px; }
.usage-fill { height: 100%; background: #28a745; }
.backup-actions { display: flex; gap: 10px; }
</style>
```

## Required Custom Fields

| Field Name | Type | Description |
|------------|------|-------------|
| Backup Client ID | Text | Bacula client identifier |
| Storage Pool | Text | Storage pool name |
| Director Address | Text | Director hostname |
| Director Port | Text | Director port (default 9101) ||
| Client Secret | Password | Encrypted client secret |
