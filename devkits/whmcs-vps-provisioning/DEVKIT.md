# WHMCS VPS Provisioning Module DevKit
# Version: 1.0 | Updated: 2026-05-29

## DevKit Structure

```
devkits/whmcs-vps-provisioning/
├── vps_provisioning.php    # Main provisioning module
├── lib/
│   └── ApiClient.php       # VPS Provider API Client
├── templates/
│   └── clientarea.tpl       # Client area template
├── hooks.php               # Hooks for automation
└── README.md               # This file
```

## Module Code

### Main Module: vps_provisioning.php

```php
<?php
/**
 * WHMCS VPS Provisioning Module
 * Provisions Virtual Private Servers with full lifecycle management
 */

if (!defined("WHMCS")) {
    die("Direct access denied");
}

/**
 * Module metadata
 */
function vps_provisioning_MetaData(): array {
    return [
        'DisplayName' => 'VPS Provisioning',
        'APIVersion' => '1.2',
        'RequiresServer' => true,
        'DefaultNonSSLPort' => 443,
        'Parameters' => [
            'server_username',
            'server_password',
            'server_access_hash',
        ],
    ];
}

/**
 * Configuration options for product setup
 */
function vps_provisioning_ConfigOptions(array $params): array {
    return [
        'vm_plan' => [
            'Type' => 'dropdown',
            'Options' => 'vps-s,vps-m,vps-l,vps-xl,vps-2xl',
            'Default' => 'vps-m',
            'Description' => 'VPS plan size',
        ],
        'operating_system' => [
            'Type' => 'dropdown',
            'Options' => 'ubuntu-22.04,ubuntu-24.04,debian-11,debian-12,centos-7,centos-stream-8,almalinux-8,windows-2019,windows-2022',
            'Default' => 'ubuntu-22.04',
            'Description' => 'Operating system image',
        ],
        'datacenter' => [
            'Type' => 'dropdown',
            'Options' => 'us-east,us-west,eu-central,eu-west,ap-south,ap-northeast',
            'Default' => 'us-east',
            'Description' => 'Datacenter region',
        ],
        'enable_backups' => [
            'Type' => 'yesno',
            'Description' => 'Enable automated backups',
        ],
        'enable_monitoring' => [
            'Type' => 'yesno',
            'Description' => 'Enable server monitoring',
        ],
        'firewall_policy' => [
            'Type' => 'dropdown',
            'Options' => 'allow-all,strict,custom',
            'Default' => 'allow-all',
            'Description' => 'Firewall policy',
        ],
    ];
}

/**
 * Create new VPS instance
 */
function vps_provisioning_CreateAccount(array $params): string {
    try {
        $api = new VpsProvisioning\ApiClient($params);
        
        // Prepare instance creation data
        $instanceData = [
            'hostname' => $params['domain'] ?: generateHostname($params['serviceid']),
            'plan' => $params['configoption1'],
            'image' => $params['configoption2'],
            'region' => $params['configoption3'],
            'user_data' => generateCloudInit($params),
            'CreateBackupPolicy' => ($params['configoption4'] === 'on'),
            'EnableMonitoring' => ($params['configoption5'] === 'on'),
        ];

        // Create the VPS instance
        $result = $api->createInstance($instanceData);

        if (empty($result['instance_id'])) {
            throw new \Exception('Instance ID not returned from API');
        }

        // Store credentials in custom fields
        saveCustomFieldValue($params['serviceid'], 'Instance ID', $result['instance_id']);
        saveCustomFieldValue($params['serviceid'], 'VPS IP Address', $result['ip_address']);
        saveCustomFieldValue($params['serviceid'], 'VPS Root Password', $result['root_password']);
        saveCustomFieldValue($params['serviceid'], 'VPS Username', 'root');
        saveCustomFieldValue($params['serviceid'], 'VPS Port', '22');

        // Log successful creation
        logActivity("VPS Module: Created instance {$result['instance_id']} for service {$params['serviceid']}");

        return 'success';
    } catch (\Exception $e) {
        logActivity("VPS Module CreateAccount Error: " . $e->getMessage());
        return 'Error: ' . $e->getMessage();
    }
}

/**
 * Suspend VPS instance
 */
function vps_provisioning_SuspendAccount(array $params): string {
    try {
        $instanceId = getCustomFieldValue($params['serviceid'], 'Instance ID');
        
        if (empty($instanceId)) {
            return 'Error: Instance ID not found. Service may not be provisioned.';
        }

        $api = new VpsProvisioning\ApiClient($params);
        $api->suspendInstance($instanceId);

        logActivity("VPS Module: Suspended instance {$instanceId}");
        return 'success';
    } catch (\Exception $e) {
        return 'Error: ' . $e->getMessage();
    }
}

/**
 * Unsuspend VPS instance
 */
function vps_provisioning_UnsuspendAccount(array $params): string {
    try {
        $instanceId = getCustomFieldValue($params['serviceid'], 'Instance ID');
        
        if (empty($instanceId)) {
            return 'Error: Instance ID not found.';
        }

        $api = new VpsProvisioning\ApiClient($params);
        $api->unsuspendInstance($instanceId);

        logActivity("VPS Module: Unsuspended instance {$instanceId}");
        return 'success';
    } catch (\Exception $e) {
        return 'Error: ' . $e->getMessage();
    }
}

/**
 * Terminate VPS instance
 */
function vps_provisioning_TerminateAccount(array $params): string {
    try {
        $instanceId = getCustomFieldValue($params['serviceid'], 'Instance ID');
        
        if (empty($instanceId)) {
            return 'Error: Instance ID not found.';
        }

        $api = new VpsProvisioning\ApiClient($params);
        $api->terminateInstance($instanceId);

        logActivity("VPS Module: Terminated instance {$instanceId}");
        return 'success';
    } catch (\Exception $e) {
        return 'Error: ' . $e->getMessage();
    }
}

/**
 * Change VPS root password
 */
function vps_provisioning_ChangePassword(array $params): string {
    try {
        $instanceId = getCustomFieldValue($params['serviceid'], 'Instance ID');
        
        if (empty($instanceId)) {
            return 'Error: Instance ID not found.';
        }

        $api = new VpsProvisioning\ApiClient($params);
        $result = $api->changePassword($instanceId, $params['password']);

        saveCustomFieldValue($params['serviceid'], 'VPS Root Password', $result['new_password']);

        logActivity("VPS Module: Password changed for instance {$instanceId}");
        return 'success';
    } catch (\Exception $e) {
        return 'Error: ' . $e->getMessage();
    }
}

/**
 * Upgrade/downgrade VPS plan
 */
function vps_provisioning_ChangePackage(array $params): string {
    try {
        $instanceId = getCustomFieldValue($params['serviceid'], 'Instance ID');
        
        if (empty($instanceId)) {
            return 'Error: Instance ID not found.';
        }

        $api = new VpsProvisioning\ApiClient($params);
        $result = $api->resizeInstance($instanceId, $params['configoption1']);

        logActivity("VPS Module: Resized instance {$instanceId} to plan {$params['configoption1']}");
        return 'success';
    } catch (\Exception $e) {
        return 'Error: ' . $e->getMessage();
    }
}

/**
 * Test connection to VPS provider API
 */
function vps_provisioning_TestConnection(array $params): array {
    try {
        $api = new VpsProvisioning\ApiClient($params);
        $result = $api->ping();

        if (($result['status'] ?? '') === 'ok') {
            return ['success' => true, 'error' => ''];
        }

        throw new \Exception('Unexpected API response');
    } catch (\Exception $e) {
        return ['success' => false, 'error' => $e->getMessage()];
    }
}

/**
 * Admin services tab information
 */
function vps_provisioning_AdminServices(array $params): array {
    $instanceId = getCustomFieldValue($params['serviceid'], 'Instance ID');
    $ipAddress = getCustomFieldValue($params['serviceid'], 'VPS IP Address');

    return [
        'Instance ID' => $instanceId ?: 'Not provisioned',
        'IP Address' => $ipAddress ?: 'N/A',
        'Plan' => $params['configoption1'],
        'Operating System' => $params['configoption2'],
        'Datacenter' => $params['configoption3'],
        'Monitoring' => ($params['configoption5'] === 'on') ? 'Enabled' : 'Disabled',
        'Status' => ucfirst($params['status']),
    ];
}

/**
 * Client area output
 */
function vps_provisioning_ClientArea(array $params): array {
    $instanceId = getCustomFieldValue($params['serviceid'], 'Instance ID');
    $ipAddress = getCustomFieldValue($params['serviceid'], 'VPS IP Address');
    $rootPassword = getCustomFieldValue($params['serviceid'], 'VPS Root Password');

    $api = null;
    $metrics = [];
    
    if ($instanceId) {
        try {
            $api = new VpsProvisioning\ApiClient($params);
            $metrics = $api->getMetrics($instanceId);
        } catch (\Exception $e) {
            // Silently fail for metrics
        }
    }

    return [
        'pagetitle' => 'VPS Service - ' . ($params['domain'] ?: 'Service #' . $params['serviceid']),
        'templatefile' => 'templates/clientarea',
        'vars' => [
            'instance_id' => $instanceId,
            'ip_address' => $ipAddress,
            'root_password' => decryptPassword($rootPassword),
            'status' => $params['status'],
            'plan' => $params['configoption1'],
            'os' => $params['configoption2'],
            'region' => $params['configoption3'],
            'metrics' => $metrics,
            'module_active' => ($params['status'] === 'Active'),
        ],
    ];
}

/**
 * Client area allowed functions (Smarty)
 */
function vps_provisioning_ClientAreaAllowedFunctions(): array {
    return [
        'RebootServer' => 'Reboot Server',
        'ReinstallOS' => 'Reinstall Operating System',
        'GetVNC' => 'Open VNC Console',
        'ToggleMonitoring' => 'Toggle Monitoring',
    ];
}

// ============================================
// Helper Functions
// ============================================

function generateHostname(int $serviceId): string {
    return 'vps-' . $serviceId . '-' . substr(md5($serviceId . time()), 0, 8);
}

function generateCloudInit(array $params): string {
    $userData = "#cloud-config\n";
    $userData .= "hostname: " . ($params['domain'] ?: 'vps-' . $params['serviceid']) . "\n";
    $userData .= "fqdn: " . ($params['domain'] ?: 'vps-' . $params['serviceid'] . '.local') . "\n";
    $userData .= "manage_etc_hosts: true\n";
    
    return base64_encode($userData);
}

function decryptPassword(string $encrypted): string {
    if (empty($encrypted)) {
        return '';
    }
    
    $decrypted = WHMCS\Application\Support\Str::decrypt($encrypted);
    return $decrypted;
}
```

### API Client: lib/ApiClient.php

```php
<?php
namespace VpsProvisioning;

use WHMCS\Database\Capsule;

class ApiClient {
    private string $baseUrl;
    private string $apiKey;
    private string $apiSecret;
    private int $timeout = 30;

    public function __construct(array $params) {
        $this->baseUrl = rtrim($params['serverhost'] ?? '', '/');
        $this->apiKey = $params['serveraccesshash'] ?? '';
        $this->apiSecret = $params['serverpassword'] ?? '';
    }

    /**
     * Make API request
     */
    public function request(string $method, string $endpoint, array $data = [], bool $auth = true): array {
        $ch = curl_init();
        $headers = [
            'Content-Type: application/json',
            'Accept: application/json',
        ];

        if ($auth) {
            $headers[] = 'Authorization: Bearer ' . $this->generateAuthToken();
        }

        curl_setopt_array($ch, [
            CURLOPT_URL => $this->baseUrl . $endpoint,
            CURLOPT_RETURNTRANSFER => true,
            CURLOPT_TIMEOUT => $this->timeout,
            CURLOPT_HTTPHEADER => $headers,
            CURLOPT_SSL_VERIFYPEER =>false,
            CURLOPT_SSL_VERIFYHOST => 0,
        ]);

        if ($method === 'POST') {
            curl_setopt($ch, CURLOPT_POST, true);
            curl_setopt($ch, CURLOPT_POSTFIELDS, json_encode($data));
        } elseif ($method !== 'GET') {
            curl_setopt($ch, CURLOPT_CUSTOMREQUEST, $method);
        }

        $response = curl_exec($ch);
        $httpCode = curl_getinfo($ch, CURLINFO_HTTP_CODE);
        $error = curl_error($ch);
        curl_close($ch);

        if ($error) {
            throw new \Exception('cURL Error: ' . $error);
        }

        $result = json_decode($response, true) ?? [];

        if ($httpCode >= 400) {
            throw new \Exception($result['message'] ?? "HTTP Error: {$httpCode}");
        }

        if (($result['success'] ?? true) === false) {
            throw new \Exception($result['error'] ?? 'API Error');
        }

        return $result['data'] ?? $result;
    }

    /**
     * Generate authentication token
     */
    private function generateAuthToken(): string {
        $payload = [
            'api_key' => $this->apiKey,
            'timestamp' => time(),
            'nonce' => bin2hex(random_bytes(16)),
        ];
        
        $signature = hash_hmac('sha256', json_encode($payload), $this->apiSecret);
        return base64_encode(json_encode(array_merge($payload, ['signature' => $signature])));
    }

    /**
     * Ping API to test connection
     */
    public function ping(): array {
        return $this->request('GET', '/api/v1/ping');
    }

    /**
     * Create new VPS instance
     */
    public function createInstance(array $data): array {
        return $this->request('POST', '/api/v1/instances', $data);
    }

    /**
     * Get instance details
     */
    public function getInstance(string $instanceId): array {
        return $this->request('GET', "/api/v1/instances/{$instanceId}");
    }

    /**
     * Suspend instance
     */
    public function suspendInstance(string $instanceId): array {
        return $this->request('POST', "/api/v1/instances/{$instanceId}/suspend");
    }

    /**
     * Unsuspend instance
     */
    public function unsuspendInstance(string $instanceId): array {
        return $this->request('POST', "/api/v1/instances/{$instanceId}/unsuspend");
    }

    /**
     * Terminate instance
     */
    public function terminateInstance(string $instanceId): array {
        return $this->request('DELETE', "/api/v1/instances/{$instanceId}");
    }

    /**
     * Change instance password
     */
    public function changePassword(string $instanceId, string $password): array {
        $hash = password_hash($password, PASSWORD_DEFAULT);
        return $this->request('POST', "/api/v1/instances/{$instanceId}/password", [
            'password' => $hash,
        ]);
    }

    /**
     * Resize/reconfigure instance
     */
    public function resizeInstance(string $instanceId, string $plan): array {
        return $this->request('POST', "/api/v1/instances/{$instanceId}/resize", [
            'plan' => $plan,
        ]);
    }

    /**
     * Reboot instance
     */
    public function rebootInstance(string $instanceId): array {
        return $this->request('POST', "/api/v1/instances/{$instanceId}/reboot");
    }

    /**
     * Reinstall operating system
     */
    public function reinstallOS(string $instanceId, string $image): array {
        return $this->request('POST', "/api/v1/instances/{$instanceId}/reinstall", [
            'image' => $image,
        ]);
    }

    /**
     * Get instance VNC console URL
     */
    public function getVNC(string $instanceId): array {
        return $this->request('GET', "/api/v1/instances/{$instanceId}/vnc");
    }

    /**
     * Get instance metrics
     */
    public function getMetrics(string $instanceId): array {
        return $this->request('GET', "/api/v1/instances/{$instanceId}/metrics");
    }

    /**
     * Take snapshot of instance
     */
    public function createSnapshot(string $instanceId, string $name = ''): array {
        return $this->request('POST', "/api/v1/instances/{$instanceId}/snapshots", [
            'name' => $name ?: "snapshot-{$instanceId}-" . date('Ymd-His'),
        ]);
    }

    /**
     * Restore from snapshot
     */
    public function restoreSnapshot(string $instanceId, string $snapshotId): array {
        return $this->request('POST', "/api/v1/instances/{$instanceId}/snapshots/{$snapshotId}/restore");
    }
}
```

### Client Area Template: templates/clientarea.tpl

```smarty
<div class="vps-client-area">
    <div class="vps-header">
        <h2><i class="fa fa-server"></i> VPS Service</h2>
        <span class="badge badge-{if $status eq 'Active'}success{elseif $status eq 'Suspended'}warning{else}danger{/if}">
            {$status}
        </span>
    </div>

    {if $module_active}
    
    <div class="vps-info-grid">
        <div class="info-card">
            <h4><i class="fa fa-tachometer"></i> Instance</h4>
            <div class="info-row">
                <label>Instance ID:</label>
                <span class="value">{$instance_id}</span>
            </div>
            <div class="info-row">
                <label>IP Address:</label>
                <span class="value"><code>{$ip_address}</code></span>
            </div>
            <div class="info-row">
                <label>Port:</label>
                <span class="value">22</span>
            </div>
        </div>

        <div class="info-card">
            <h4><i class="fa fa-cogs"></i> Configuration</h4>
            <div class="info-row">
                <label>Plan:</label>
                <span class="value">{$plan|upper}</span>
            </div>
            <div class="info-row">
                <label>OS:</label>
                <span class="value">{$os}</span>
            </div>
            <div class="info-row">
                <label>Region:</label>
                <span class="value">{$region}</span>
            </div>
        </div>
    </div>

    {if $metrics}
    <div class="vps-metrics">
        <h4><i class="fa fa-chart-line"></i> Resource Usage</h4>
        <div class="metrics-grid">
            <div class="metric-box">
                <div class="metric-value">{$metrics.cpu_usage}%</div>
                <div class="metric-label">CPU</div>
                <div class="metric-bar">
                    <div class="metric-fill" style="width: {$metrics.cpu_usage}%"></div>
                </div>
            </div>
            <div class="metric-box">
                <div class="metric-value">{$metrics.memory_usage}%</div>
                <div class="metric-label">Memory</div>
                <div class="metric-bar">
                    <div class="metric-fill" style="width: {$metrics.memory_usage}%"></div>
                </div>
            </div>
            <div class="metric-box">
                <div class="metric-value">{$metrics.disk_usage}%</div>
                <div class="metric-label">Disk</div>
                <div class="metric-bar">
                    <div class="metric-fill" style="width: {$metrics.disk_usage}%"></div>
                </div>
            </div>
            <div class="metric-box">
                <div class="metric-value">{$metrics.bandwidth_used}</div>
                <div class="metric-label">Bandwidth</div>
            </div>
        </div>
    </div>
    {/if}

    <div class="vps-actions">
        <h4><i class="fa fa-bolt"></i> Quick Actions</h4>
        <div class="action-buttons">
            <button type="button" class="btn btn-primary" onclick="vpsAction('reboot')">
                <i class="fa fa-sync"></i> Reboot
            </button>
            <button type="button" class="btn" onclick="vpsAction('console')">
                <i class="fa fa-desktop"></i> Console
            </button>
            <button type="button" class="btn" onclick="vpsAction('metrics')">
                <i class="fa fa-chart-bar"></i> Metrics
            </button>
            <button type="button" class="btn btn-danger" onclick="vpsAction('reinstall')">
                <i class="fa fa-redo"></i> Reinstall OS
            </button>
        </div>
    </div>

    {else}
    <div class="vps-pending">
        <i class="fa fa-clock"></i>
        <p>Your VPS is being provisioned. This page will update automatically.</p>
    </div>
    {/if}
</div>

<script>
function vpsAction(action) {
    window.location.href = 'clientarea.php?action=' + action + '&service={$service.id}';
}
</script>

<style>
.vps-client-area { padding: 20px; }
.vps-header { display: flex; justify-content: space-between; align-items: center; margin-bottom: 20px; }
.vps-info-grid { display: grid; grid-template-columns: repeat(auto-fit, minmax(300px, 1fr)); gap: 20px; margin-bottom: 20px; }
.info-card { background: #f8f9fa; border-radius: 8px; padding: 15px; }
.info-card h4 { margin: 0 0 15px 0; color: #333; }
.info-row { display: flex; justify-content: space-between; padding: 8px 0; border-bottom: 1px solid #eee; }
.info-row:last-child { border-bottom: none; }
.info-row label { color: #666; font-weight: 500; }
.info-row .value { font-weight: 600; }
.vps-metrics { background: #f8f9fa; border-radius: 8px; padding: 15px; margin-bottom: 20px; }
.vps-metrics h4 { margin: 0 0 15px 0; }
.metrics-grid { display: grid; grid-template-columns: repeat(auto-fit, minmax(150px, 1fr)); gap: 15px; }
.metric-box { text-align: center; }
.metric-value { font-size: 24px; font-weight: bold; color: #333; }
.metric-label { color: #666; margin-bottom: 10px; }
.metric-bar { height: 8px; background: #e0e0e0; border-radius: 4px; overflow: hidden; }
.metric-fill { height: 100%; background: linear-gradient(90deg, #28a745, #ffc107, #dc3545); transition: width 0.3s; }
.vps-actions { background: #f8f9fa; border-radius: 8px; padding: 15px; }
.vps-actions h4 { margin: 0 0 15px 0; }
.action-buttons { display: flex; gap: 10px; flex-wrap: wrap; }
.vps-pending { text-align: center; padding: 40px; color: #666; }
</style>
```

### Hooks: hooks.php

```php
<?php
/**
 * VPS Provisioning Module Hooks
 */

use WHMCS\Database\Capsule;

/**
 * Hook: After VPS creation - Send welcome email
 */
add_hook('AfterModuleCreate', 1, function($vars) {
    if ($vars['module'] !== 'vps_provisioning') {
        return;
    }

    $serviceId = $vars['serviceid'];
    $instanceId = getCustomFieldValue($serviceId, 'Instance ID');
    $ipAddress = getCustomFieldValue($serviceId, 'VPS IP Address');
    $rootPassword = decryptCustomFieldValue(getCustomFieldValue($serviceId, 'VPS Root Password'));

    $emailData = [
        'instance_id' => $instanceId,
        'ip_address' => $ipAddress,
        'root_password' => $rootPassword,
        'port' => '22',
    ];

    sendEmail('vps-welcome', $serviceId, $emailData);
});

/**
 * Hook: Before VPS termination - Create backup snapshot
 */
add_hook('PreModuleTerminate', 1, function($vars) {
    if ($vars['module'] !== 'vps_provisioning') {
        return;
    }

    $serviceId = $vars['serviceid'];
    $instanceId = getCustomFieldValue($serviceId, 'Instance ID');

    if (empty($instanceId)) {
        return;
    }

    // Create final backup before termination
    $api = new VpsProvisioning\ApiClient(loadServerParams($serviceId));
    $api->createSnapshot($instanceId, "pre-termination-backup");

    logActivity("VPS Module: Created pre-termination backup for instance {$instanceId}");
});

/**
 * Hook: DailyCronJob - Sync VPS status
 */
add_hook('DailyCronJob', 1, function($vars) {
    $modules = Capsule::table('tblhosting')
        ->where('domain', 'like', 'vps-%')
        ->where('module', 'vps_provisioning')
        ->whereIn('status', ['Active', 'Suspended'])
        ->get();

    foreach ($modules as $service) {
        $instanceId = getCustomFieldValue($service->id, 'Instance ID');
        
        if (empty($instanceId)) {
            continue;
        }

        $api = new VpsProvisioning\ApiClient(loadServerParams($service->server));
        $status = $api->getInstanceStatus($instanceId);

        updateServiceStatus($service->id, $status);
    }
});
```

## Database Schema SQL

```sql
-- VPS Instances Table
CREATE TABLE `mod_vps_provisioning_instances` (
    `id` INT UNSIGNED NOT NULL AUTO_INCREMENT,
    `service_id` INT UNSIGNED NOT NULL,
    `instance_id` VARCHAR(100) NOT NULL,
    `ip_address` VARCHAR(45) NULL,
    `plan` VARCHAR(50) NULL,
    `os_image` VARCHAR(100) NULL,
    `region` VARCHAR(50) NULL,
    `created_at` TIMESTAMP NULL,
    `updated_at` TIMESTAMP NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    PRIMARY KEY (`id`),
    UNIQUE KEY `idx_instance_id` (`instance_id`),
    INDEX `idx_service_id` (`service_id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;

-- VPS Snapshots Table
CREATE TABLE `mod_vps_provisioning_snapshots` (
    `id` INT UNSIGNED NOT NULL AUTO_INCREMENT,
    `instance_id` VARCHAR(100) NOT NULL,
    `snapshot_id` VARCHAR(100) NOT NULL,
    `name` VARCHAR(255) NULL,
    `size_gb` DECIMAL(10,2) NULL,
    `status` VARCHAR(20) DEFAULT 'pending',
    `created_at` TIMESTAMP NULL,
    PRIMARY KEY (`id`),
    UNIQUE KEY `idx_snapshot_id` (`snapshot_id`),
    INDEX `idx_instance_id` (`instance_id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;

-- VPS Logs Table
CREATE TABLE `mod_vps_provisioning_logs` (
    `id` INT UNSIGNED NOT NULL AUTO_INCREMENT,
    `service_id` INT UNSIGNED NULL,
    `level` VARCHAR(20) DEFAULT 'info',
    `message` TEXT NOT NULL,
    `context` JSON NULL,
    `created_at` TIMESTAMP NULL DEFAULT CURRENT_TIMESTAMP,
    PRIMARY KEY (`id`),
    INDEX `idx_service_id` (`service_id`),
    INDEX `idx_created_at` (`created_at`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
```

## Checklist

```
Pre-Dev:
[ ] Obtain VPS Provider API documentation
[ ] Get sandbox/test API credentials
[ ] Identify all available OS images and plans
[ ] Map API endpoints to WHMCS functions
[ ] Document custom fields to create

Development:
[ ] Create ApiClient class with OAuth/API key auth
[ ] Implement CreateAccount - return 'success' or error
[ ] Implement SuspendAccount
[ ] Implement UnsuspendAccount
[ ] Implement TerminateAccount
[ ] Implement ChangePassword
[ ] Implement ChangePackage for upgrades
[ ] Implement TestConnection
[ ] Add AdminServices for admin area
[ ] Add ClientArea for customer portal
[ ] Add ClientAreaAllowedFunctions for smarty
[ ] Add MetaData function
[ ] Add ConfigOptions function

Testing:
[ ] Create test instance in sandbox
[ ] Verify credentials are saved to custom fields
[ ] Test suspend/unsuspend cycle
[ ] Test termination and cleanup
[ ] Test password change
[ ] Test plan upgrade/downgrade
[ ] Test error handling and logging

Production:
[ ] Add production API credentials
[ ] Configure product with correct options
[ ] Test full order and provisioning flow
[ ] Verify emails are sent
[ ] Monitor logs for any issues
```

## Required Custom Fields

| Field Name | Type | Description |
|------------|------|-------------|
| Instance ID | Text | VPS instance identifier |
| VPS IP Address | Text | Server IPv4 address |
| VPS Root Password | Password | Encrypted root password |
| VPS Username | Text | Admin username (root) |
| VPS Port | Text | SSH port (default: 22) |
