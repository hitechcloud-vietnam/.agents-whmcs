# WHMCS Cloudflare Module Skill
# Version: 1.0 | Updated: 2026-05-28

## Purpose

Guide for building Cloudflare/DNS management modules.

## When to Use

- Creating DNS provisioning modules
- Building CDN integration modules
- Implementing zone management

## Cloudflare Module Pattern

```php
<?php
// modules/servers/{cloudflaremoudle}/{cloudflaremoudle}.php
if (!defined("WHMCS")) { die("Direct access denied"); }

function {cloudflaremoudle}_MetaData(): array {
    return [
        'DisplayName' => 'Cloudflare DNS',
        'APIVersion' => '1.1',
        'RequiresServer' => false,
    ];
}

function {cloudflaremoudle}_ConfigOptions(array $params): array {
    return [
        'ZonePlan' => [
            'Type' => 'dropdown',
            'Options' => 'free,pro,business,enterprise',
            'Default' => 'free',
        ],
        'AutoSSL' => ['Type' => 'yesno', 'Default' => 'on'],
        'SecurityLevel' => [
            'Type' => 'dropdown',
            'Options' => 'medium,high,essentially_off',
            'Default' => 'medium',
        ],
    ];
}

function {cloudflaremoudle}_CreateAccount(array $params): string {
    $api = new \Cloudflare\ApiClient($params);

    try {
        // Create Cloudflare zone
        $zone = $api->createZone($params['domain']);

        if (!$zone['success']) {
            return 'Error: ' . ($zone['errors'][0]['message'] ?? 'Failed to create zone');
        }

        // Get zone ID
        $zoneId = $zone['result']['id'];

        // Configure zone settings based on config options
        $api->updateZoneSettings($zoneId, [
            'ssl' => 'full',
            'security_level' => $params['configoption3'],
            'always_use_https' => 'on',
            'automatic_https_rewrites' => 'on',
        ]);

        // Enable Automatic SSL if configured
        if ($params['configoption2'] == 'on') {
            $api->enableAutomaticSSL($zoneId);
        }

        // Create DNS records
        $api->createDnsRecord($zoneId, [
            'type' => 'A',
            'name' => '@',
            'content' => $params['serverip'] ?? '',
            'proxied' => true,
        ]);

        // Store zone ID
        saveCustomFieldValue($params['serviceid'], 'cf_zone_id', $zoneId);
        saveCustomFieldValue($params['serviceid'], 'cf_zone_name', $params['domain']);

        return 'success';
    } catch (\Exception $e) {
        return 'Error: ' . $e->getMessage();
    }
}

function {cloudflaremoudle}_SuspendAccount(array $params): string {
    $zoneId = getCustomFieldValue($params['serviceid'], 'cf_zone_id');

    try {
        $api = new \Cloudflare\ApiClient($params);
        $api->pauseZone($zoneId);
        return 'success';
    } catch (\Exception $e) {
        return 'Error: ' . $e->getMessage();
    }
}

function {cloudflaremoudle}_UnsuspendAccount(array $params): string {
    $zoneId = getCustomFieldValue($params['serviceid'], 'cf_zone_id');

    try {
        $api = new \Cloudflare\ApiClient($params);
        $api->unpauseZone($zoneId);
        return 'success';
    } catch (\Exception $e) {
        return 'Error: ' . $e->getMessage();
    }
}

function {cloudflaremoudle}_TerminateAccount(array $params): string {
    $zoneId = getCustomFieldValue($params['serviceid'], 'cf_zone_id');

    try {
        $api = new \Cloudflare\ApiClient($params);
        $api->deleteZone($zoneId);
        return 'success';
    } catch (\Exception $e) {
        return 'Error: ' . $e->getMessage();
    }
}

function {cloudflaremoudle}_ChangePackage(array $params): string {
    $zoneId = getCustomFieldValue($params['serviceid'], 'cf_zone_id');

    try {
        $api = new \Cloudflare\ApiClient($params);
        $api->updateZonePlan($zoneId, $params['configoption1']);
        return 'success';
    } catch (\Exception $e) {
        return 'Error: ' . $e->getMessage();
    }
}

function {cloudflaremoudle}_TestConnection(array $params): array {
    try {
        $api = new \Cloudflare\ApiClient($params);
        $zones = $api->listZones();

        return ['success' => true, 'error' => ''];
    } catch (\Exception $e) {
        return ['success' => false, 'error' => $e->getMessage()];
    }
}

function {cloudflaremoudle}_ClientArea(array $params): array {
    $zoneId = getCustomFieldValue($params['serviceid'], 'cf_zone_id');

    return [
        'pagetitle' => 'Cloudflare Dashboard',
        'templatefile' => 'dashboard',
        'vars' => [
            'zone_id' => $zoneId,
            'domain' => $params['domain'],
        ],
    ];
}
```

### API Client Class

```php
<?php
// modules/servers/{cloudflaremoudle}/lib/ApiClient.php

namespace Cloudflare;

class ApiClient {
    private string $token;
    private string $email;
    private string $baseUrl = 'https://api.cloudflare.com/client/v4';

    public function __construct(array $params) {
        $this->token = $params['serverusername'] ?? '';
        $this->email = $params['serveraccesshash'] ?? '';
    }

    private function request(string $method, string $endpoint, array $data = []): array {
        $ch = curl_init();

        curl_setopt_array($ch, [
            CURLOPT_URL => $this->baseUrl . $endpoint,
            CURLOPT_RETURNTRANSFER => true,
            CURLOPT_CUSTOMREQUEST => strtoupper($method),
            CURLOPT_HTTPHEADER => [
                'Authorization: Bearer ' . $this->token,
                'Content-Type: application/json',
            ],
        ]);

        if (!empty($data)) {
            curl_setopt($ch, CURLOPT_POSTFIELDS, json_encode($data));
        }

        $response = curl_exec($ch);
        $httpCode = curl_getinfo($ch, CURLINFO_HTTP_CODE);
        curl_close($ch);

        $result = json_decode($response, true);

        if ($httpCode >= 400) {
            throw new \Exception($result['errors'][0]['message'] ?? 'API Error');
        }

        return $result;
    }

    public function createZone(string $domain): array {
        return $this->request('POST', '/zones', [
            'name' => $domain,
            'type' => 'full',
        ]);
    }

    public function listZones(): array {
        return $this->request('GET', '/zones');
    }

    public function deleteZone(string $zoneId): array {
        return $this->request('DELETE', "/zones/$zoneId");
    }

    public function pauseZone(string $zoneId): array {
        return $this->request('PATCH', "/zones/$zoneId", [
            'paused' => true,
        ]);
    }

    public function unpauseZone(string $zoneId): array {
        return $this->request('PATCH', "/zones/$zoneId", [
            'paused' => false,
        ]);
    }

    public function updateZoneSettings(string $zoneId, array $settings): array {
        return $this->request('PATCH', "/zones/$zoneId/settings", $settings);
    }

    public function createDnsRecord(string $zoneId, array $record): array {
        return $this->request('POST', "/zones/$zoneId/dns_records", $record);
    }

    public function listDnsRecords(string $zoneId): array {
        return $this->request('GET', "/zones/$zoneId/dns_records");
    }

    public function enableAutomaticSSL(string $zoneId): array {
        return $this->request('PATCH', "/zones/$zoneId/ssl/ssl_settings", [
            'value' => 'automatic',
        ]);
    }
}
```

---

**Related Skills:**
- whmcs-server-builder
- whmcs-domain-sync
- whmcs-webhook-handler
