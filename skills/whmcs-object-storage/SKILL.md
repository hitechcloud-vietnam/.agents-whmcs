# WHMCS Object Storage Module Skill
# Version: 1.0 | Updated: 2026-05-28

## Purpose

Build object storage (S3-compatible) provisioning modules.

## Object Storage Module Structure

```php
<?php
/**
 * Object Storage Module
 * Location: modules/servers/{module}/
 */

function {module}_MetaData(): array {
    return [
        'DisplayName' => 'Object Storage',
        'APIVersion' => '1.0',
        'RequiresServer' => true,
        'Parameters' => ['api_key', 'api_secret', 'region'],
    ];
}

function {module}_ConfigOptions(array $params): array {
    return [
        'StorageQuota' => [
            'Type' => 'dropdown',
            'Options' => '100GB,500GB,1TB,5TB,10TB',
            'Default' => '1TB',
        ],
        'MaxBuckets' => [
            'Type' => 'dropdown',
            'Options' => '10,50,100,unlimited',
            'Default' => '100',
        ],
        'CDNEnabled' => [
            'Type' => 'yesno',
            'Description' => 'Enable CDN integration',
        ],
    ];
}
```

## Storage Operations

```php
function {module}_CreateAccount(array $params): string {
    $user = $this->api->createUser([
        'email' => $params['clientsdetails']['email'],
        'quota' => $this->parseQuota($params['configoption1']),
        'max_buckets' => $params['configoption2'] === 'unlimited' ? -1 : (int) $params['configoption2'],
    ]);

    $accessKey = $this->api->createAccessKey($user['id']);

    Capsule::table('mod_object_storage')->insert([
        'service_id' => $params['serviceid'],
        'user_id' => $user['id'],
        'access_key' => encrypt($accessKey['access_key']),
        'secret_key' => encrypt($accessKey['secret_key']),
        'endpoint' => $this->getEndpoint($params),
        'quota' => $this->parseQuota($params['configoption1']),
        'max_buckets' => $params['configoption2'],
        'cdn_enabled' => $params['configoption3'] === 'on',
        'created_at' => date('Y-m-d H:i:s'),
    ]);

    return 'success';
}

function {module}_SuspendAccount(array $params): string {
    $storage = $this->getStorage($params['serviceid']);

    $this->api->suspendUser($storage['user_id']);

    Capsule::table('mod_object_storage')
        ->where('service_id', $params['serviceid'])
        ->update(['status' => 'suspended']);

    return 'success';
}

function {module}_UnsuspendAccount(array $params): string {
    $storage = $this->getStorage($params['serviceid']);

    $this->api->unsuspendUser($storage['user_id']);

    Capsule::table('mod_object_storage')
        ->where('service_id', $params['serviceid'])
        ->update(['status' => 'active']);

    return 'success';
}

function {module}_TerminateAccount(array $params): string {
    $storage = $this->getStorage($params['serviceid']);

    if ($storage) {
        $this->api->deleteUser($storage['user_id']);
        Capsule::table('mod_object_storage')
            ->where('id', $storage['id'])
            ->delete();
    }

    return 'success';
}
```

## Usage Statistics

```php
public function getUsageStats(int $serviceId): array {
    $storage = $this->getStorage($serviceId);

    $stats = $this->api->getUserStats($storage['user_id']);

    Capsule::table('mod_object_storage_stats')->insert([
        'storage_id' => $storage['id'],
        'bytes_used' => $stats['bytes_used'],
        'objects_count' => $stats['objects_count'],
        'buckets_count' => $stats['buckets_count'],
        'requests_count' => $stats['requests_count'],
        'recorded_at' => date('Y-m-d H:i:s'),
    ]);

    return [
        'bytes_used' => $this->formatBytes($stats['bytes_used']),
        'bytes_limit' => $this->formatBytes($storage['quota']),
        'percent_used' => round(($stats['bytes_used'] / $storage['quota']) * 100, 2),
        'objects' => $stats['objects_count'],
        'buckets' => $stats['buckets_count'],
    ];
}

private function formatBytes(int $bytes): string {
    $units = ['B', 'KB', 'MB', 'GB', 'TB'];
    $i = 0;
    while ($bytes >= 1024 && $i < 4) {
        $bytes /= 1024;
        $i++;
    }
    return round($bytes, 2) . ' ' . $units[$i];
}

private function parseQuota(string $value): int {
    $value = str_replace(['GB', 'TB'], ['', ''], $value);
    $unit = str_contains($value, 'GB') ? 'GB' : 'TB';
    $num = (int) str_replace($unit, '', $value);
    return $num * ($unit === 'TB' ? 1024 : 1) * 1024 * 1024 * 1024;
}
```

## Client Area

```php
function {module}_ClientArea(array $params): array {
    $storage = $this->getStorage($params['serviceid']);
    $stats = $this->getUsageStats($params['serviceid']);

    $credentials = [
        'endpoint' => $storage['endpoint'],
        'access_key' => decrypt($storage['access_key']),
        'secret_key' => decrypt($storage['secret_key']),
    ];

    return [
        'pagetitle' => 'Object Storage',
        'templatefile' => 'templates/storage_clientarea',
        'vars' => [
            'stats' => $stats,
            'credentials' => $credentials,
            'max_buckets' => $storage['max_buckets'],
            'cdn_enabled' => $storage['cdn_enabled'],
        ],
    ];
}
```

---

**Related Skills:**
- whmcs-server-builder
- whmcs-reporting
- whmcs-cdn-integration