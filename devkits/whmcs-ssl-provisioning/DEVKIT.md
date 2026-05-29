# WHMCS SSL Certificate Provisioning Module DevKit
# Version: 1.0 | Updated: 2026-05-29

## DevKit Structure

```
devkits/whmcs-ssl-provisioning/
├── ssl_provisioning.php        # Main provisioning module
├── lib/
│   └── ApiClient.php          # SSL provider API client
├── templates/
│   ├── admin.tpl               # Admin area template
│   └── clientarea.tpl          # Client area template
└── DEVKIT.md                   # This file
```

## Module Code

```php
<?php
/**
 * WHMCS SSL Certificate Provisioning Module
 */

if (!defined("WHMCS")) {
    die("Direct access denied");
}

function ssl_provisioning_MetaData(): array {
    return [
        'DisplayName' => 'SSL Certificates',
        'APIVersion' => '1.1',
        'RequiresServer' => true,
        'DefaultNonSSLPort' => 443,
        'Parameters' => ['server_username', 'server_password', 'server_access_hash'],
    ];
}

function ssl_provisioning_ConfigOptions(array $params): array {
    return [
        'certificate_type' => [
            'Type' => 'dropdown',
            'Options' => 'dv,ov,ev,multi-domain,wildcard',
            'Default' => 'dv',
            'Description' => 'Certificate type',
        ],
        'validation_method' => [
            'Type' => 'dropdown',
            'Options' => 'dns,email,file',
            'Default' => 'dns',
            'Description' => 'Domain validation method',
        ],
        'term' => [
            'Type' => 'dropdown',
            'Options' => '1,2,3',
            'Default' => '1',
            'Description' => 'Years',
        ],
        'wildcard_included' => [
            'Type' => 'yesno',
            'Description' => 'Include www wildcard',
        ],
    ];
}

function ssl_provisioning_CreateAccount(array $params): string {
    try {
        $api = new SslProvisioning\ApiClient($params);
        
        $domains = $params['customfields']['Domains'] ?? $params['domain'];
        
        $orderData = [
            'common_name' => $domains,
            'type' => $params['configoption1'],
            'validation' => $params['configoption2'],
            'term' => (int) $params['configoption3'],
            'csr' => $params['customfields']['CSR'] ?? '',
            'contact_email' => $params['clientdetails']['email'],
        ];

        $result = $api->createCertificate($orderData);

        saveCustomFieldValue($params['serviceid'], 'Order ID', $result['order_id']);
        saveCustomFieldValue($params['serviceid'], 'Certificate ID', $result['cert_id']);
        saveCustomFieldValue($params['serviceid'], 'Validation Method', $params['configoption2']);
        saveCustomFieldValue($params['serviceid'], 'Validation Token', $result['validation_token'] ?? '');

        logActivity("SSL Provisioning: Created certificate order {$result['order_id']}");
        return 'success';
        
    } catch (\Exception $e) {
        logActivity("SSL Provisioning CreateAccount Error: " . $e->getMessage());
        return 'Error: ' . $e->getMessage();
    }
}

function ssl_provisioning_SuspendAccount(array $params): string {
    return 'success';
}

function ssl_provisioning_UnsuspendAccount(array $params): string {
    return 'success';
}

function ssl_provisioning_TerminateAccount(array $params): string {
    try {
        $orderId = getCustomFieldValue($params['serviceid'], 'Order ID');
        $api = new SslProvisioning\ApiClient($params);
        $api->cancelCertificate($orderId);
        return 'success';
    } catch (\Exception $e) {
        return 'Error: ' . $e->getMessage();
    }
}

function ssl_provisioning_ChangePackage(array $params): string {
    return 'success';
}

function ssl_provisioning_TestConnection(array $params): array {
    try {
        $api = new SslProvisioning\ApiClient($params);
        $api->listProducts();
        return ['success' => true, 'error' => ''];
    } catch (\Exception $e) {
        return ['success' => false, 'error' => $e->getMessage()];
    }
}

function ssl_provisioning_AdminServices(array $params): array {
    return [
        'Order ID' => getCustomFieldValue($params['serviceid'], 'Order ID') ?: 'N/A',
        'Certificate ID' => getCustomFieldValue($params['serviceid'], 'Certificate ID') ?: 'N/A',
        'Type' => $params['configoption1'],
        'Validation' => $params['configoption2'],
        'Term' => $params['configoption3'] . ' year(s)',
    ];
}

function ssl_provisioning_ClientArea(array $params): array {
    $orderId = getCustomFieldValue($params['serviceid'], 'Order ID');
    $validationMethod = getCustomFieldValue($params['serviceid'], 'Validation Method');
    $validationToken = getCustomFieldValue($params['serviceid'], 'Validation Token');
    
    $status = [];
    try {
        $api = new SslProvisioning\ApiClient($params);
        $status = $api->getOrderStatus($orderId);
    } catch (\Exception $e) {}

    return [
        'pagetitle' => 'SSL Certificate - ' . ($params['domain'] ?: 'Certificate'),
        'templatefile' => 'templates/clientarea',
        'vars' => [
            'order_id' => $orderId,
            'validation_method' => $validationMethod,
            'validation_token' => $validationToken,
            'status' => $status,
            'service_status' => $params['status'],
        ],
    ];
}

function ssl_provisioning_ClientAreaAllowedFunctions(): array {
    return [
        'ReIssueCertificate' => 'Re-issue Certificate',
        'DownloadCertificate' => 'Download Cert',
        'ValidateDomain' => 'Validate Domain',
    ];
}
```

## API Client: lib/ApiClient.php

```php
<?php
namespace SslProvisioning;

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

    public function listProducts(): array {
        return $this->request('GET', '/api/v1/products');
    }

    public function createCertificate(array $data): array {
        return $this->request('POST', '/api/v1/certificates', $data);
    }

    public function getOrderStatus(string $orderId): array {
        return $this->request('GET', "/api/v1/orders/{$orderId}");
    }

    public function reIssueCertificate(string $orderId, array $data): array {
        return $this->request('POST', "/api/v1/orders/{$orderId}/reissue", $data);
    }

    public function verifyDomain(string $orderId, string $domain, string $method): array {
        return $this->request('POST', "/api/v1/orders/{$orderId}/verify", [
            'domain' => $domain,
            'method' => $method,
        ]);
    }

    public function downloadCertificate(string $certId): array {
        return $this->request('GET', "/api/v1/certificates/{$certId}/download");
    }

    public function cancelCertificate(string $orderId): array {
        return $this->request('DELETE', "/api/v1/orders/{$orderId}");
    }
}
```

## Client Area Template

```smarty
<div class="ssl-cert-client">
    <div class="ssl-header">
        <h2><i class="fa fa-lock"></i> SSL Certificate</h2>
        <span class="badge">{$service_status}</span>
    </div>

    {if $status}
    <div class="ssl-status-card">
        <h4><i class="fa fa-info-circle"></i> Certificate Status</h4>
        <div class="status-info">
            <div class="status-row">
                <span class="label">Status:</span>
                <span class="value badge badge-{if $status.status eq 'issued'}success{elseif $status.status eq 'pending'}warning{else}danger{/if}">
                    {$status.status|upper}
                </span>
            </div>
            <div class="status-row">
                <span class="label">Issued:</span>
                <span class="value">{$status.issued_date|date_format:'%Y-%m-%d'}</span>
            </div>
            <div class="status-row">
                <span class="label">Expires:</span>
                <span class="value">{$status.expires_date|date_format:'%Y-%m-%d'}</span>
            </div>
        </div>
    </div>
    {/if}

    <div class="validation-info">
        <h4><i class="fa fa-check-circle"></i> Domain Validation</h4>
        <p>Method: <strong>{$validation_method|upper}</strong></p>
        
        {if $validation_method eq 'dns'}
        <div class="validation-steps">
            <p>Add the following CNAME record to your DNS:</p>
            <code> _acme-challenge.yourdomain.com IN CNAME {$validation_token}</code>
        </div>
        {elseif $validation_method eq 'email'}
        <p>A validation email has been sent to your domain's admin contacts.</p>
        {elseif $validation_method eq 'file'}
        <p>Create a file at: <code>/.well-known/pki-validation/{$validation_token}.txt</code></p>
        {/if}
    </div>

    <div class="ssl-actions">
        <a href="?m=ssl_provisioning&action=download&id={$order_id}" class="btn btn-primary">
            <i class="fa fa-download"></i> Download Certificate
        </a>
        <a href="?m=ssl_provisioning&action=reissue&id={$order_id}" class="btn">
            <i class="fa fa-redo"></i> Re-issue
        </a>
    </div>
</div>

<style>
.ssl-cert-client { padding: 20px; }
.ssl-header { display: flex; justify-content: space-between; align-items: center; margin-bottom: 20px; }
.ssl-status-card { background: #f8f9fa; border-radius: 8px; padding: 15px; margin-bottom: 20px; }
.status-info .status-row { display: flex; justify-content: space-between; padding: 8px 0; }
.validation-info { background: #fff3cd; border-radius: 8px; padding: 15px; margin-bottom: 20px; }
.validation-info code { display: block; background: #fff; padding: 10px; margin-top: 10px; border-radius: 4px; }
.ssl-actions { display: flex; gap: 10px; }
</style>
```

## Required Custom Fields

| Field Name | Type | Description |
|------------|------|-------------|
| Domains | Textarea | Comma-separated domains |
| CSR | Textarea | Certificate Signing Request |
| Order ID | Text | Order identifier |
| Certificate ID | Text | Certificate identifier |
| Validation Method | Text | DNS, email, or file |
| Validation Token | Text | Validation token from CA |
