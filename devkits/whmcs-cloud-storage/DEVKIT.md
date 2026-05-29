# WHMCS Cloud Storage Provisioning Module DevKit
# Version: 1.0 | Updated: 2026-05-29

## Module Overview

This provisioning module integrates with S3-compatible cloud storage providers to automatically provision storage buckets and access credentials when clients order the service.

## DevKit Structure

```
devkits/whmcs-cloud-storage/
├── cloud_storage.php         # Main provisioning module
├── lib/
│   └── ApiClient.php        # S3-compatible API client
├── templates/
│   └── clientarea.tpl        # Client area template
└── DEVKIT.md                 # This file
```

## Module Code

```php
<?php
/**
 * WHMCS Cloud Storage Provisioning Module
 * Provisions S3-compatible cloud storage buckets and credentials
 */

if (!defined("WHMCS")) {
    die("Direct access denied");
}

function cloud_storage_MetaData(): array {
    return [
        'DisplayName' => 'Cloud Storage',
        'APIVersion' => '1.1',
        'RequiresServer' => true,
        'DefaultNonSSLPort' => 443,
        'Parameters' => ['server_username', 'server_password', 'server_access_hash'],
    ];
}

function cloud_storage_ConfigOptions(array $params): array {
    return [
        'storage_tier' => [
            'Type' => 'dropdown',
            'Options' => 'standard,standard-ia,onezone-ia,glacier,glacier-deep-archive',
            'Default' => 'standard',
            'Description' => 'Storage class tier',
        ],
        'bucket_prefix' => [
            'Type' => 'text',
            'Default' => 'client',
            'Description' => 'Bucket name prefix',
        ],
        'max_capacity_gb' => [
            'Type' => 'text',
            'Default' => '100',
            'Description' => 'Maximum storage in GB',
        ],
        'enable_encryption' => [
            'Type' => 'yesno',
            'Description' => 'Enable server-side encryption',
        ],
    ];
}

function cloud_storage_CreateAccount(array $params): string {
    try {
        $api = new CloudStorage\ApiClient($params);
        
        $bucketName = generateBucketName($params);
        
        $result = $api->createBucket([
            'name' => $bucketName,
            'storage_class' => $params['configoption1'],
            'acl' => 'private',
            'enable_encryption' => ($params['configoption4'] === 'on'),
        ]);

        if (empty($result['bucket_name'])) {
            throw new \Exception('Bucket not created');
        }

        // Generate access credentials
        $credentials = $api->createAccessKey($result['bucket_name']);
        
        saveCustomFieldValue($params['serviceid'], 'Bucket Name', $result['bucket_name']);
        saveCustomFieldValue($params['serviceid'], 'Access Key ID', $credentials['access_key']);
        saveCustomFieldValue($params['serviceid'], 'Secret Key', encryptValue($credentials['secret_key']));
        saveCustomFieldValue($params['serviceid'], 'Endpoint', $api->getEndpoint());
        
        logActivity("CloudStorage: Created bucket {$bucketName} for service {$params['serviceid']}");
        return 'success';
        
    } catch (\Exception $e) {
        logActivity("CloudStorage CreateAccount Error: " . $e->getMessage());
        return 'Error: ' . $e->getMessage();
    }
}

function cloud_storage_SuspendAccount(array $params): string {
    try {
        $bucketName = getCustomFieldValue($params['serviceid'], 'Bucket Name');
        $api = new CloudStorage\ApiClient($params);
        $api->suspendBucket($bucketName);
        return 'success';
    } catch (\Exception $e) {
        return 'Error: ' . $e->getMessage();
    }
}

function cloud_storage_UnsuspendAccount(array $params): string {
    try {
        $bucketName = getCustomFieldValue($params['serviceid'], 'Bucket Name');
        $api = new CloudStorage\ApiClient($params);
        $api->activateBucket($bucketName);
        return 'success';
    } catch (\Exception $e) {
        return 'Error: ' . $e->getMessage();
    }
}

function cloud_storage_TerminateAccount(array $params): string {
    try {
        $bucketName = getCustomFieldValue($params['serviceid'], 'Bucket Name');
        $api = new CloudStorage\ApiClient($params);
        $api->deleteBucket($bucketName);
        return 'success';
    } catch (\Exception $e) {
        return 'Error: ' . $e->getMessage();
    }
}

function cloud_storage_ChangePassword(array $params): string {
    try {
        $bucketName = getCustomFieldValue($params['serviceid'], 'Bucket Name');
        $api = new CloudStorage\ApiClient($params);
        $newCredentials = $api->rotateAccessKey($bucketName);
        saveCustomFieldValue($params['serviceid'], 'Secret Key', encryptValue($newCredentials['secret_key']));
        return 'success';
    } catch (\Exception $e) {
        return 'Error: ' . $e->getMessage();
    }
}

function cloud_storage_ChangePackage(array $params): string {
    try {
        $bucketName = getCustomFieldValue($params['serviceid'], 'Bucket Name');
        $api = new CloudStorage\ApiClient($params);
        $api->updateBucketTier($bucketName, $params['configoption1']);
        return 'success';
    } catch (\Exception $e) {
        return 'Error: ' . $e->getMessage();
    }
}

function cloud_storage_TestConnection(array $params): array {
    try {
        $api = new CloudStorage\ApiClient($params);
        $api->listBuckets();
        return ['success' => true, 'error' => ''];
    } catch (\Exception $e) {
        return ['success' => false, 'error' => $e->getMessage()];
    }
}

function cloud_storage_AdminServices(array $params): array {
    return [
        'Bucket' => getCustomFieldValue($params['serviceid'], 'Bucket Name') ?: 'N/A',
        'Tier' => $params['configoption1'],
        'Capacity' => $params['configoption3'] . ' GB',
        'Encryption' => ($params['configoption4'] === 'on') ? 'Enabled' : 'Disabled',
    ];
}

function cloud_storage_ClientArea(array $params): array {
    $bucketName = getCustomFieldValue($params['serviceid'], 'Bucket Name');
    $accessKey = getCustomFieldValue($params['serviceid'], 'Access Key ID');
    $endpoint = getCustomFieldValue($params['serviceid'], 'Endpoint');
    
    $usage = [];
    try {
        $api = new CloudStorage\ApiClient($params);
        $usage = $api->getBucketUsage($bucketName);
    } catch (\Exception $e) {}

    return [
        'pagetitle' => 'Cloud Storage - ' . ($params['domain'] ?: 'Storage Service'),
        'templatefile' => 'templates/clientarea',
        'vars' => [
            'bucket_name' => $bucketName,
            'access_key' => $accessKey,
            'endpoint' => $endpoint,
            'usage' => $usage,
            'status' => $params['status'],
        ],
    ];
}

function cloud_storage_ClientAreaAllowedFunctions(): array {
    return [
        'GetS3Cmd' => 'S3 Command',
        'GeneratePresignedUrl' => 'Generate Download Link',
    ];
}

// Helper Functions
function generateBucketName(array $params): string {
    $prefix = $params['configoption2'] ?: 'client';
    $uniqueId = substr(md5($params['serviceid'] . time()), 0, 8);
    $cleanDomain = preg_replace('/[^a-zA-Z0-9]/', '', $params['domain'] ?? '');
    return strtolower($prefix . '-' . $cleanDomain . '-' . $uniqueId);
}

function encryptValue(string $value): string {
    if (function_exists('openssl_encrypt')) {
        return base64_encode(openssl_encrypt($value, 'AES-256-CBC', get_encryption_key(), 0, substr(md5(get_encryption_key()), 0, 16)));
    }
    return base64_encode($value);
}

function get_encryption_key(): string {
    return defined('CRYPT_KEY') ? CRYPT_KEY : 'default-fallback-key';
}
```

## API Client: lib/ApiClient.php

```php
<?php
namespace CloudStorage;

class ApiClient {
    private string $baseUrl;
    private string $accessKey;
    private string $secretKey;
    private int $timeout = 30;

    public function __construct(array $params) {
        $this->baseUrl = rtrim($params['serverhost'] ?? '', '/');
        $this->accessKey = $params['serverusername'] ?? '';
        $this->secretKey = $params['serveraccesshash'] ?? '';
    }

    public function request(string $method, string $endpoint, array $data = []): array {
        $ch = curl_init();
        $expires = time() + 3600;
        $authSignature = $this->generateSignature($method, $endpoint, $expires);

        curl_setopt_array($ch, [
            CURLOPT_URL => $this->baseUrl . $endpoint,
            CURLOPT_RETURNTRANSFER => true,
            CURLOPT_TIMEOUT => $this->timeout,
            CURLOPT_HTTPHEADER => [
                'Authorization: S3 ' . $this->accessKey . ':' . $authSignature,
                'x-amz-date: ' . gmdate('Ymd\THis\Z'),
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

    private function generateSignature(string $method, string $path, int $expires): string {
        $stringToSign = "{$method}\n{$path}\n{$expires}";
        return base64_encode(hash_hmac('sha256', $stringToSign, $this->secretKey, true));
    }

    public function getEndpoint(): string {
        return $this->baseUrl;
    }

    public function createBucket(array $data): array {
        return $this->request('POST', '/api/v1/buckets', $data);
    }

    public function listBuckets(): array {
        return $this->request('GET', '/api/v1/buckets');
    }

    public function suspendBucket(string $bucketName): array {
        return $this->request('POST', "/api/v1/buckets/{$bucketName}/suspend");
    }

    public function activateBucket(string $bucketName): array {
        return $this->request('POST', "/api/v1/buckets/{$bucketName}/activate");
    }

    public function deleteBucket(string $bucketName): array {
        return $this->request('DELETE', "/api/v1/buckets/{$bucketName}");
    }

    public function createAccessKey(string $bucketName): array {
        return $this->request('POST', "/api/v1/buckets/{$bucketName}/credentials");
    }

    public function rotateAccessKey(string $bucketName): array {
        return $this->request('POST', "/api/v1/buckets/{$bucketName}/credentials/rotate");
    }

    public function getBucketUsage(string $bucketName): array {
        return $this->request('GET', "/api/v1/buckets/{$bucketName}/usage");
    }

    public function updateBucketTier(string $bucketName, string $tier): array {
        return $this->request('PUT', "/api/v1/buckets/{$bucketName}/tier", ['tier' => $tier]);
    }
}
```

## Client Area Template

```smarty
<div class="cloud-storage-client">
    <div class="storage-header">
        <h2><i class="fa fa-cloud"></i> Cloud Storage</h2>
        <span class="badge badge-{$status|lower}">{$status}</span>
    </div>

    <div class="storage-credentials">
        <h4><i class="fa fa-key"></i> Access Credentials</h4>
        <div class="credential-grid">
            <div class="credential-item">
                <label>Endpoint:</label>
                <code>{$endpoint}</code>
            </div>
            <div class="credential-item">
                <label>Bucket:</label>
                <code>{$bucket_name}</code>
            </div>
            <div class="credential-item">
                <label>Access Key:</label>
                <code>{$access_key}</code>
            </div>
        </div>
    </div>

    {if $usage}
    <div class="storage-usage">
        <h4><i class="fa fa-chart-pie"></i> Usage Statistics</h4>
        <div class="usage-stats">
            <div class="stat-box">
                <div class="stat-value">{$usage.storage_used|bytes}</div>
                <div class="stat-label">Used</div>
            </div>
            <div class="stat-box">
                <div class="stat-value">{$usage.max_storage|bytes}</div>
                <div class="stat-label">Total</div>
            </div>
            <div class="stat-box">
                <div class="stat-value">{$usage.objects_count}</div>
                <div class="stat-label">Objects</div>
            </div>
        </div>
        <div class="usage-bar">
            <div class="usage-fill" style="width: {($usage.storage_used / $usage.max_storage) * 100}%"></div>
        </div>
    </div>
    {/if}
</div>

<style>
.cloud-storage-client { padding: 20px; }
.storage-header { display: flex; justify-content: space-between; margin-bottom: 20px; }
.storage-credentials { background: #f8f9fa; border-radius: 8px; padding: 15px; margin-bottom: 20px; }
.credential-grid { display: grid; gap: 10px; }
.credential-item { display: flex; justify-content: space-between; padding: 8px; background: white; border-radius: 4px; }
.storage-usage { background: #f8f9fa; border-radius: 8px; padding: 15px; }
.usage-stats { display: flex; gap: 20px; margin-bottom: 15px; }
.stat-box { text-align: center; flex: 1; }
.stat-value { font-size: 20px; font-weight: bold; }
.usage-bar { height: 10px; background: #e0e0e0; border-radius: 5px; overflow: hidden; }
.usage-fill { height: 100%; background: #28a745; transition: width 0.3s; }
</style>
```

## Required Custom Fields

| Field Name | Type | Description |
|------------|------|-------------|
| Bucket Name | Text | S3 bucket identifier |
| Access Key ID | Password | Access key for API |
| Secret Key | Password | Encrypted secret key |
| Endpoint | Text | S3 endpoint URL |
