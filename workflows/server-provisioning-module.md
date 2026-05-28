# WHMCS Server/Provisioning Module Workflow
# Version: 1.0 | Created: 2026-05-28

---

## Overview

This workflow guides the creation of WHMCS provisioning/server modules for hosting services like VPS, cloud, dedicated servers.

## Prerequisites

1. Read `.agents-whmcs/CLAUDE.md` (Technical Reference)
2. Read `Core_exapm_whmcs/sample-provisioning-module/` (Sample module)
3. Identify API documentation for the target provider

---

## Module Structure

```
modules/servers/{module}/
├── {module}.php              ← Main module file
├── lib/
│   └── ApiClient.php        ← API client class
├── templates/
│   ├── overview.tpl         ← Service overview page
│   ├── error.tpl            ← Error template
│   └── manage.tpl            ← Management page
├── hooks.php                ← Hooks (optional)
├── logo.png                 ← 80x80px logo
└── composer.json            ← (optional)
```

---

## Step-by-Step Development

### Step 1: Create Module Directory

```bash
mkdir -p "module_dev_whmcs/modules/servers/{module}/lib"
mkdir -p "module_dev_whmcs/modules/servers/{module}/templates"
```

### Step 2: Create Main Module File

Create `{module}.php` with these required functions:

```php
<?php
/**
 * {Module} WHMCS Provisioning Module
 * Version: 1.0.0
 * Author: HiTechCloud
 */

if (!defined("WHMCS")) {
    die("Direct access denied");
}

/**
 * Module Metadata
 */
function {module}_MetaData(): array {
    return [
        'DisplayName' => '{Provider Name}',
        'APIVersion'  => '1.0',
        'RequiresServer' => true,
        'DefaultNonSSLPort' => 443,
    ];
}

/**
 * Configuration Options
 */
function {module}_ConfigOptions(array $params): array {
    return [
        'Plan' => [
            'Type' => 'dropdown',
            'Options' => 'starter,basic,premium',
            'Default' => 'starter',
        ],
        'Location' => [
            'Type' => 'dropdown',
            'Options' => 'us-east,eu-west,asia-pacific',
            'Default' => 'us-east',
        ],
    ];
}

/**
 * Create Account
 */
function {module}_CreateAccount(array $params): string {
    try {
        $api = new \Module\ApiClient($params);
        $result = $api->createServer([
            'hostname' => $params['customfields']['hostname'] ?? $params['domain'],
            'plan' => $params['configoption1'],
            'location' => $params['configoption2'],
        ]);

        if (isset($result['error'])) {
            return 'Error: ' . $result['error'];
        }

        // Save server details
        saveCustomFieldValue($params['serviceid'], 'Server IP', $result['ip']);
        saveCustomFieldValue($params['serviceid'], 'Server ID', $result['server_id']);

        return 'success';
    } catch (\Exception $e) {
        logActivity('{Module} Create Error: ' . $e->getMessage());
        return 'Error: ' . $e->getMessage();
    }
}

/**
 * Suspend Account
 */
function {module}_SuspendAccount(array $params): string {
    try {
        $api = new \Module\ApiClient($params);
        $result = $api->suspendServer($params['customfields']['server_id']);

        return isset($result['error']) ? 'Error: ' . $result['error'] : 'success';
    } catch (\Exception $e) {
        return 'Error: ' . $e->getMessage();
    }
}

/**
 * Unsuspend Account
 */
function {module}_UnsuspendAccount(array $params): string {
    try {
        $api = new \Module\ApiClient($params);
        $result = $api->unsuspendServer($params['customfields']['server_id']);

        return isset($result['error']) ? 'Error: ' . $result['error'] : 'success';
    } catch (\Exception $e) {
        return 'Error: ' . $e->getMessage();
    }
}

/**
 * Terminate Account
 */
function {module}_TerminateAccount(array $params): string {
    try {
        $api = new \Module\ApiClient($params);
        $result = $api->deleteServer($params['customfields']['server_id']);

        return isset($result['error']) ? 'Error: ' . $result['error'] : 'success';
    } catch (\Exception $e) {
        return 'Error: ' . $e->getMessage();
    }
}

/**
 * Change Password
 */
function {module}_ChangePassword(array $params): string {
    try {
        $api = new \Module\ApiClient($params);
        $result = $api->changeRootPassword(
            $params['customfields']['server_id'],
            $params['password']
        );

        return isset($result['error']) ? 'Error: ' . $result['error'] : 'success';
    } catch (\Exception $e) {
        return 'Error: ' . $e->getMessage();
    }
}

/**
 * Change Package
 */
function {module}_ChangePackage(array $params): string {
    try {
        $api = new \Module\ApiClient($params);
        $result = $api->resizeServer(
            $params['customfields']['server_id'],
            $params['configoption1']
        );

        return isset($result['error']) ? 'Error: ' . $result['error'] : 'success';
    } catch (\Exception $e) {
        return 'Error: ' . $e->getMessage();
    }
}

/**
 * Test Connection
 */
function {module}_TestConnection(array $params): array {
    try {
        $api = new \Module\ApiClient($params);
        $result = $api->ping();

        return [
            'success' => true,
            'error' => '',
            'raw' => json_encode($result),
        ];
    } catch (\Exception $e) {
        return [
            'success' => false,
            'error' => $e->getMessage(),
            'raw' => '',
        ];
    }
}

/**
 * Client Area
 */
function {module}_ClientArea(array $params): array {
    return [
        'pagetitle' => 'Service Details',
        'templatefile' => 'overview',
        'vars' => [
            'serverIp' => $params['customfields']['server_ip'] ?? '',
            'serverId' => $params['customfields']['server_id'] ?? '',
            'status' => $params['status'],
        ],
    ];
}

/**
 * Admin Service Details
 */
function {module}_AdminServices(array $params): array {
    return [
        'Server ID' => $params['customfields']['server_id'] ?? 'N/A',
        'Server IP' => $params['customfields']['server_ip'] ?? 'N/A',
    ];
}

// Helper function to save custom field values
function saveCustomFieldValue($serviceId, $fieldName, $value) {
    $fieldId = \WHMCS\Database\Capsule::table('tblcustomfields')
        ->where('relid', $serviceId)
        ->where('fieldname', $fieldName)
        ->first();

    if ($fieldId) {
        \WHMCS\Database\Capsule::table('tblcustomfieldvalues')
            ->updateOrInsert(
                ['relid' => $serviceId, 'fieldid' => $fieldId->id],
                ['value' => $value]
            );
    }
}
```

### Step 3: Create API Client

Create `lib/ApiClient.php`:

```php
<?php
namespace Module;

class ApiClient {
    private $apiUrl;
    private $apiKey;
    private $apiSecret;

    public function __construct(array $params) {
        $this->apiUrl = rtrim($params['serverhttpprefix'] . '://' . $params['serverhostname'], '/');
        $this->apiKey = $params['serverusername'];
        $this->apiSecret = $params['serverpassword'];
    }

    public function request(string $method, string $endpoint, array $data = []): array {
        $ch = curl_init();
        $url = $this->apiUrl . $endpoint;

        curl_setopt_array($ch, [
            CURLOPT_URL => $url,
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
        $error = curl_error($ch);
        curl_close($ch);

        if ($error) {
            throw new \Exception('cURL Error: ' . $error);
        }

        $decoded = json_decode($response, true);

        if ($httpCode >= 400) {
            throw new \Exception($decoded['message'] ?? 'API Error');
        }

        return $decoded;
    }

    public function createServer(array $data): array {
        return $this->request('POST', '/api/servers', $data);
    }

    public function deleteServer(string $serverId): array {
        return $this->request('DELETE', '/api/servers/' . $serverId);
    }

    public function suspendServer(string $serverId): array {
        return $this->request('POST', '/api/servers/' . $serverId . '/suspend');
    }

    public function unsuspendServer(string $serverId): array {
        return $this->request('POST', '/api/servers/' . $serverId . '/unsuspend');
    }

    public function changeRootPassword(string $serverId, string $password): array {
        return $this->request('POST', '/api/servers/' . $serverId . '/password', [
            'password' => $password,
        ]);
    }

    public function resizeServer(string $serverId, string $plan): array {
        return $this->request('PUT', '/api/servers/' . $serverId, [
            'plan' => $plan,
        ]);
    }

    public function ping(): array {
        return $this->request('GET', '/api/ping');
    }
}
```

### Step 4: Create Templates

Create `templates/overview.tpl`:

```smarty
<div class="module-detail">
    <h2>Server Information</h2>

    <div class="info-grid">
        <div class="info-row">
            <span class="label">Status:</span>
            <span class="value status-{$status|lower}">{$status}</span>
        </div>
        <div class="info-row">
            <span class="label">Server IP:</span>
            <span class="value">{$serverIp|default:'Pending'}</span>
        </div>
    </div>

    <div class="actions">
        <a href="clientarea.php?action=custom&module={$MODULE}" class="btn btn-primary">
            Manage Server
        </a>
    </div>
</div>

<style>
.module-detail { padding: 20px; }
.info-grid { margin: 20px 0; }
.info-row { display: flex; padding: 10px 0; border-bottom: 1px solid #eee; }
.info-row .label { font-weight: bold; width: 120px; }
.status-active { color: green; }
.status-suspended { color: orange; }
.status-terminated { color: red; }
</style>
```

### Step 5: Create Logo

Create `logo.png` (80x80px PNG)

### Step 6: Create whmcs.json (optional)

```json
{
    "schema": "1.0",
    "type": "provisioning-module",
    "name": {
        "en": "Provider Name"
    },
    "version": "1.0.0",
    "authors": [
        {
            "name": "HiTechCloud",
            "homepage": "https://hitechcloud.vn"
        }
    ]
}
```

---

## Checklist

- [ ] Module file with all required functions
- [ ] API Client class
- [ ] Templates
- [ ] Logo (80x80px)
- [ ] Test connection works
- [ ] Create/Suspend/Unsuspend/Terminate tested
- [ ] Password change works
- [ ] Client area displays correctly

---

## Common Issues

| Issue | Solution |
|-------|----------|
| 405 Error | Check URL format and HTTP method |
| 401 Error | Verify API credentials |
| "success" returned but no server | Check API response handling |
| Custom fields not saving | Verify field names match exactly |

---

Last updated: 2026-05-28