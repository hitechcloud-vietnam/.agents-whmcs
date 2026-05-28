# WHMCS Server Provisioning Module Builder Skill
# Version: 1.0 | Updated: 2026-05-28

## Purpose

Systematic guide for building WHMCS provisioning/server modules from scratch.

## When to Use

- Creating new VPS, cloud, or dedicated server provisioning modules
- Converting existing API integrations into WHMCS modules
- Adding new provider support

## Prerequisites

1. Read `.agents-whmcs/CLAUDE.md`
2. Read `Core_exapm_whmcs/sample-provisioning-module/`
3. Have provider API documentation ready

## Module Structure

```
modules/servers/{module}/
├── {module}.php              ← Main module file
├── lib/
│   └── ApiClient.php        ← API client class
├── templates/
│   ├── overview.tpl         ← Service overview
│   ├── manage.tpl           ← Management page
│   └── error.tpl            ← Error template
├── hooks.php                ← Hooks (optional)
├── logo.png                 ← 80x80px logo
└── composer.json            ← (optional)
```

## Building Steps

### Step 1: Create API Client

```php
<?php
// lib/ApiClient.php
namespace Provider;

class ApiClient {
    private string $baseUrl;
    private string $apiKey;
    private string $apiSecret;

    public function __construct(array $params) {
        $this->baseUrl = $params['serverhttpprefix'] . '://' . $params['serverhostname'];
        $this->apiKey = $params['serverusername'];
        $this->apiSecret = $params['serverpassword'];
    }

    public function request(string $method, string $endpoint, array $data = []): array {
        $ch = curl_init();
        curl_setopt_array($ch, [
            CURLOPT_URL => $this->baseUrl . $endpoint,
            CURLOPT_RETURNTRANSFER => true,
            CURLOPT_TIMEOUT => 60,
            CURLOPT_SSL_VERIFYPEER => true,
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

        return json_decode($response, true) ?? [];
    }

    public function createServer(array $params): array {
        return $this->request('POST', '/api/servers', [
            'hostname' => $params['hostname'],
            'plan' => $params['plan'],
            'image' => $params['image'],
            'region' => $params['region'],
        ]);
    }

    public function getServer(string $id): array {
        return $this->request('GET', '/api/servers/' . $id);
    }

    public function deleteServer(string $id): array {
        return $this->request('DELETE', '/api/servers/' . $id);
    }

    public function startServer(string $id): array {
        return $this->request('POST', '/api/servers/' . $id . '/start');
    }

    public function stopServer(string $id): array {
        return $this->request('POST', '/api/servers/' . $id . '/stop');
    }

    public function rebootServer(string $id): array {
        return $this->request('POST', '/api/servers/' . $id . '/reboot');
    }

    public function rebuildServer(string $id, string $image): array {
        return $this->request('POST', '/api/servers/' . $id . '/rebuild', [
            'image' => $image,
        ]);
    }

    public function changePassword(string $id, string $password): array {
        return $this->request('POST', '/api/servers/' . $id . '/password', [
            'password' => $password,
        ]);
    }

    public function resizeServer(string $id, string $plan): array {
        return $this->request('POST', '/api/servers/' . $id . '/resize', [
            'plan' => $plan,
        ]);
    }

    public function getVNC(string $id): array {
        return $this->request('GET', '/api/servers/' . $id . '/vnc');
    }

    public function getPlans(): array {
        return $this->request('GET', '/api/plans');
    }

    public function getImages(): array {
        return $this->request('GET', '/api/images');
    }

    public function getRegions(): array {
        return $this->request('GET', '/api/regions');
    }
}
```

### Step 2: Create Main Module

```php
<?php
// modules/servers/{module}/{module}.php
if (!defined("WHMCS")) { die("Direct access denied"); }

function {module}_MetaData(): array {
    return [
        'DisplayName' => '{Provider Name}',
        'APIVersion' => '1.0',
        'RequiresServer' => true,
        'DefaultNonSSLPort' => 443,
    ];
}

function {module}_ConfigOptions(array $params): array {
    return [
        'Plan' => [
            'Type' => 'dropdown',
            'Options' => 'starter,basic,premium,enterprise',
            'Default' => 'starter',
        ],
        'Location' => [
            'Type' => 'dropdown',
            'Options' => 'us-east,eu-west,asia-pacific',
            'Default' => 'us-east',
        ],
        'Image' => [
            'Type' => 'dropdown',
            'Options' => 'ubuntu-22.04,ubuntu-24.04,debian-12,centos-9',
            'Default' => 'ubuntu-22.04',
        ],
    ];
}

function {module}_CreateAccount(array $params): string {
    try {
        $api = new \Provider\ApiClient($params);
        $hostname = $params['customfields']['hostname'] ?? $params['domain'];

        $result = $api->createServer([
            'hostname' => $hostname,
            'plan' => $params['configoption1'],
            'image' => $params['configoption3'],
            'region' => $params['configoption2'],
        ]);

        if (isset($result['error'])) {
            return 'Error: ' . $result['error'];
        }

        saveCustomFieldValue($params['serviceid'], 'server_id', $result['id']);
        saveCustomFieldValue($params['serviceid'], 'server_ip', $result['ip']);

        return 'success';
    } catch (\Exception $e) {
        logActivity('{Module} Error: ' . $e->getMessage());
        return 'Error: ' . $e->getMessage();
    }
}

function {module}_SuspendAccount(array $params): string {
    try {
        $serverId = getCustomFieldValue($params['serviceid'], 'server_id');
        if (!$serverId) return 'Error: Server ID not found';

        $api = new \Provider\ApiClient($params);
        $api->stopServer($serverId);

        return 'success';
    } catch (\Exception $e) {
        return 'Error: ' . $e->getMessage();
    }
}

function {module}_UnsuspendAccount(array $params): string {
    try {
        $serverId = getCustomFieldValue($params['serviceid'], 'server_id');
        if (!$serverId) return 'Error: Server ID not found';

        $api = new \Provider\ApiClient($params);
        $api->startServer($serverId);

        return 'success';
    } catch (\Exception $e) {
        return 'Error: ' . $e->getMessage();
    }
}

function {module}_TerminateAccount(array $params): string {
    try {
        $serverId = getCustomFieldValue($params['serviceid'], 'server_id');
        if (!$serverId) return 'Error: Server ID not found';

        $api = new \Provider\ApiClient($params);
        $api->deleteServer($serverId);

        return 'success';
    } catch (\Exception $e) {
        return 'Error: ' . $e->getMessage();
    }
}

function {module}_ChangePassword(array $params): string {
    try {
        $serverId = getCustomFieldValue($params['serviceid'], 'server_id');
        if (!$serverId) return 'Error: Server ID not found';

        $api = new \Provider\ApiClient($params);
        $api->changePassword($serverId, $params['password']);

        return 'success';
    } catch (\Exception $e) {
        return 'Error: ' . $e->getMessage();
    }
}

function {module}_ChangePackage(array $params): string {
    try {
        $serverId = getCustomFieldValue($params['serviceid'], 'server_id');
        if (!$serverId) return 'Error: Server ID not found';

        $api = new \Provider\ApiClient($params);
        $api->resizeServer($serverId, $params['configoption1']);

        return 'success';
    } catch (\Exception $e) {
        return 'Error: ' . $e->getMessage();
    }
}

function {module}_TestConnection(array $params): array {
    try {
        $api = new \Provider\ApiClient($params);
        $result = $api->request('GET', '/api/ping');

        return ['success' => true, 'error' => '', 'raw' => json_encode($result)];
    } catch (\Exception $e) {
        return ['success' => false, 'error' => $e->getMessage(), 'raw' => ''];
    }
}

function {module}_ClientArea(array $params): array {
    $serverId = getCustomFieldValue($params['serviceid'], 'server_id');
    $serverIp = getCustomFieldValue($params['serviceid'], 'server_ip');

    return [
        'pagetitle' => 'Server Details',
        'templatefile' => 'overview',
        'vars' => [
            'serverId' => $serverId,
            'serverIp' => $serverIp,
            'status' => $params['status'],
        ],
    ];
}

function {module}_AdminServices(array $params): array {
    return [
        'Server ID' => getCustomFieldValue($params['serviceid'], 'server_id') ?: 'N/A',
        'Server IP' => getCustomFieldValue($params['serviceid'], 'server_ip') ?: 'N/A',
    ];
}

// Helper functions
function saveCustomFieldValue(int $serviceId, string $fieldName, string $value): void {
    $field = Capsule::table('tblcustomfields')
        ->where('relid', $serviceId)
        ->where('fieldname', $fieldName)
        ->first();

    if ($field) {
        Capsule::table('tblcustomfieldvalues')
            ->updateOrInsert(
                ['relid' => $serviceId, 'fieldid' => $field->id],
                ['value' => $value]
            );
    }
}

function getCustomFieldValue(int $serviceId, string $fieldName): ?string {
    $field = Capsule::table('tblcustomfields')
        ->where('relid', $serviceId)
        ->where('fieldname', $fieldName)
        ->first();

    if (!$field) return null;

    $value = Capsule::table('tblcustomfieldvalues')
        ->where('relid', $serviceId)
        ->where('fieldid', $field->id)
        ->value('value');

    return $value ?: null;
}
```

### Step 3: Create Templates

```smarty
{* templates/overview.tpl *}
<div class="module-service">
    <h2>Server Information</h2>

    <div class="info-grid">
        <div class="info-item">
            <label>Status</label>
            <span class="badge badge-{$status|lower}">{$status}</span>
        </div>
        <div class="info-item">
            <label>Server ID</label>
            <span>{$serverId|default:'Pending'}</span>
        </div>
        <div class="info-item">
            <label>Server IP</label>
            <span>{$serverIp|default:'Pending'}</span>
        </div>
    </div>

    <div class="actions">
        <a href="clientarea.php?action=manage" class="btn btn-primary">
            Manage Server
        </a>
    </div>
</div>

<style>
.module-service { padding: 20px; }
.info-grid { display: grid; gap: 15px; margin: 20px 0; }
.info-item { display: flex; justify-content: space-between; padding: 10px; background: #f5f5f5; }
.badge { padding: 4px 12px; border-radius: 4px; }
.badge-active { background: #28a745; color: white; }
.badge-suspended { background: #ffc107; color: black; }
.badge-terminated { background: #dc3545; color: white; }
</style>
```

## Checklist

- [ ] ApiClient with all API methods
- [ ] MetaData with DisplayName
- [ ] ConfigOptions with dropdowns/options
- [ ] CreateAccount returns 'success' or error string
- [ ] SuspendAccount/UnsuspendAccount/TerminateAccount
- [ ] ChangePassword/ChangePackage
- [ ] TestConnection returns ['success' => bool]
- [ ] ClientArea returns template array
- [ ] AdminServices returns custom fields
- [ ] logo.png (80x80px)

---

**Related Skills:**
- whmcs-api-integration
- whmcs-testing-qa
- whmcs-clientarea-builder