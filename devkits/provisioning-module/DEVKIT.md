# WHMCS Provisioning Module DevKit
# Version: 1.0 | Updated: 2026-05-28

## DevKit Structure

```
devkits/provisioning-module/
├── template.php          # Complete module template
├── lib/
│   └── ApiClient.php     # API client skeleton
├── templates/
│   └── clientarea.tpl    # Client area template
└── hooks.php             # Hook examples
```

## Template

```php
<?php
/**
 * WHMCS Provisioning Module: {module}
 * DevKit Template
 */

if (!defined("WHMCS")) {
    die("Direct access denied");
}

function {module}_MetaData(): array {
    return [
        'DisplayName' => '{Provider Name}',
        'APIVersion' => '1.1',
        'RequiresServer' => true,
        'DefaultNonSSLPort' => 443,
        'Parameters' => ['server_username', 'server_password', 'server_access_hash'],
    ];
}

function {module}_ConfigOptions(array $params): array {
    return [
        'Plan' => [
            'Type' => 'dropdown',
            'Options' => 'plan1,plan2,plan3',
            'Default' => 'plan1',
            'Description' => 'Select the plan',
        ],
        'Image' => [
            'Type' => 'text',
            'Default' => 'ubuntu-22.04',
            'Description' => 'OS/Template name',
        ],
    ];
}

function {module}_CreateAccount(array $params): string {
    try {
        $api = new \{Module}\ApiClient($params);
        
        $result = $api->createInstance([
            'hostname' => $params['domain'],
            'plan' => $params['configoption1'],
            'image' => $params['configoption2'],
            'user_id' => $params['customfields']['user_id'] ?? $params['username'],
        ]);

        // Store credentials in custom fields
        saveCustomFieldValue($params['serviceid'], 'instance_id', $result['instance_id']);
        saveCustomFieldValue($params['serviceid'], 'api_key', $result['api_key']);

        return 'success';
    } catch (\Exception $e) {
        logActivity('{Module} CreateAccount Error: ' . $e->getMessage());
        return 'Error: ' . $e->getMessage();
    }
}

function {module}_SuspendAccount(array $params): string {
    try {
        $instanceId = getCustomFieldValue($params['serviceid'], 'instance_id');
        if (!$instanceId) {
            return 'Error: Instance ID not found';
        }

        $api = new \{Module}\ApiClient($params);
        $api->suspendInstance($instanceId);

        return 'success';
    } catch (\Exception $e) {
        return 'Error: ' . $e->getMessage();
    }
}

function {module}_UnsuspendAccount(array $params): string {
    try {
        $instanceId = getCustomFieldValue($params['serviceid'], 'instance_id');
        $api = new \{Module}\ApiClient($params);
        $api->unsuspendInstance($instanceId);

        return 'success';
    } catch (\Exception $e) {
        return 'Error: ' . $e->getMessage();
    }
}

function {module}_TerminateAccount(array $params): string {
    try {
        $instanceId = getCustomFieldValue($params['serviceid'], 'instance_id');
        $api = new \{Module}\ApiClient($params);
        $api->terminateInstance($instanceId);

        return 'success';
    } catch (\Exception $e) {
        return 'Error: ' . $e->getMessage();
    }
}

function {module}_ChangePassword(array $params): string {
    try {
        $instanceId = getCustomFieldValue($params['serviceid'], 'instance_id');
        $api = new \{Module}\ApiClient($params);
        $api->changePassword($instanceId, $params['password']);

        return 'success';
    } catch (\Exception $e) {
        return 'Error: ' . $e->getMessage();
    }
}

function {module}_ChangePackage(array $params): string {
    try {
        $instanceId = getCustomFieldValue($params['serviceid'], 'instance_id');
        $api = new \{Module}\ApiClient($params);
        $api->resizeInstance($instanceId, $params['configoption1']);

        return 'success';
    } catch (\Exception $e) {
        return 'Error: ' . $e->getMessage();
    }
}

function {module}_TestConnection(array $params): array {
    try {
        $api = new \{Module}\ApiClient($params);
        $result = $api->testConnection();

        return ['success' => true, 'error' => ''];
    } catch (\Exception $e) {
        return ['success' => false, 'error' => $e->getMessage()];
    }
}

function {module}_AdminServices(array $params): array {
    $instanceId = getCustomFieldValue($params['serviceid'], 'instance_id');

    return [
        'Instance ID' => $instanceId ?: 'N/A',
        'Plan' => $params['configoption1'],
        'Status' => $params['status'],
    ];
}

function {module}_ClientArea(array $params): array {
    return [
        'pagetitle' => 'Service Details',
        'templatefile' => 'templates/clientarea',
        'vars' => [
            'instance_id' => getCustomFieldValue($params['serviceid'], 'instance_id'),
            'status' => $params['status'],
        ],
    ];
}

function {module}_ClientAreaAllowedFunctions(): array {
    return [
        'RebootInstance' => 'Reboot',
        'ReinstallOS' => 'Reinstall OS',
    ];
}
```

## API Client Skeleton

```php
<?php
namespace {Module};

class ApiClient {
    private string $baseUrl;
    private string $apiKey;
    private int $timeout = 30;

    public function __construct(array $params) {
        $this->baseUrl = rtrim($params['serverhost'], '/');
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
        $error = curl_error($ch);
        curl_close($ch);

        if ($error) {
            throw new \Exception('cURL Error: ' . $error);
        }

        $result = json_decode($response, true);

        if ($httpCode >= 400 || ($result['status'] ?? '') === 'error') {
            throw new \Exception($result['message'] ?? 'API Error');
        }

        return $result;
    }

    public function testConnection(): array {
        return $this->request('GET', '/api/v1/ping');
    }

    public function createInstance(array $data): array {
        return $this->request('POST', '/api/v1/instances', $data);
    }

    public function suspendInstance(string $instanceId): array {
        return $this->request('POST', "/api/v1/instances/$instanceId/suspend");
    }

    public function unsuspendInstance(string $instanceId): array {
        return $this->request('POST', "/api/v1/instances/$instanceId/unsuspend");
    }

    public function terminateInstance(string $instanceId): array {
        return $this->request('DELETE', "/api/v1/instances/$instanceId");
    }

    public function changePassword(string $instanceId, string $password): array {
        return $this->request('POST', "/api/v1/instances/$instanceId/password", [
            'password' => $password,
        ]);
    }

    public function resizeInstance(string $instanceId, string $plan): array {
        return $this->request('POST', "/api/v1/instances/$instanceId/resize", [
            'plan' => $plan,
        ]);
    }
}
```

## Client Area Template

```smarty
<div class="client-area-module">
    <div class="module-header">
        <h2>{$service.product_name}</h2>
        <span class="badge badge-{$status|lower}">{$status}</span>
    </div>

    <div class="service-info">
        <div class="info-row">
            <span class="label">Instance ID:</span>
            <span class="value">{$instance_id}</span>
        </div>
    </div>

    <div class="action-buttons">
        <a href="?m={module}&action=console" class="btn btn-primary">
            Open Console
        </a>
        <a href="?m={module}&action=metrics" class="btn">
            View Metrics
        </a>
    </div>
</div>
```

## Checklist

```
Pre-Dev:
□ Get API documentation from provider
□ Understand authentication method
□ Identify all API endpoints
□ Create sandbox account for testing

Development:
□ Create ApiClient class
□ Implement CreateAccount → Return 'success'
□ Implement SuspendAccount → Return 'success'
□ Implement UnsuspendAccount → Return 'success'
□ Implement TerminateAccount → Return 'success'
□ Implement ChangePassword → Return 'success'
□ Implement ChangePackage → Return 'success'
□ Implement TestConnection → Return ['success' => bool]
□ Add MetaData function
□ Add ConfigOptions function
□ Add AdminServices if needed
□ Add ClientArea if needed

Testing:
□ Test with sandbox credentials
□ Test account creation flow
□ Test suspend/unsuspend
□ Test termination
□ Verify custom fields saved
□ Test error scenarios
```
