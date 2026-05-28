# WHMCS DNS Management Skill
# Version: 1.0 | Updated: 2026-05-28

## Purpose

Build DNS management modules and integrate with DNS providers.

## DNS Module Structure

```php
<?php
/**
 * DNS Management Module
 * Location: modules/servers/{module}/
 */

function {module}_MetaData(): array {
    return [
        'DisplayName' => 'DNS Management',
        'APIVersion' => '1.0',
        'RequiresServer' => true,
        'Parameters' => ['api_key', 'api_secret'],
    ];
}

function {module}_ConfigOptions(array $params): array {
    return [
        'DefaultTTL' => [
            'Type' => 'dropdown',
            'Options' => '300,600,900,1800,3600,7200,14400,28800,86400',
            'Default' => '3600',
        ],
        'DynamicDNS' => [
            'Type' => 'yesno',
            'Description' => 'Enable dynamic DNS updates',
        ],
    ];
}
```

## DNS Record Management

```php
function {module}_CreateAccount(array $params): string {
    $domain = $params['domain'];

    Capsule::table('mod_dns_zones')->insert([
        'service_id' => $params['serviceid'],
        'domain' => $domain,
        'ns1' => $params['serverip'],
        'ns2' => $this->getSecondaryNS(),
        'ttl' => (int) $params['configoption1'],
        'created_at' => date('Y-m-d H:i:s'),
    ]);

    $this->api->createZone($domain, [
        'ttl' => (int) $params['configoption1'],
        'nameservers' => [$params['serverip'], $this->getSecondaryNS()],
    ]);

    return 'success';
}

function {module}_TerminateAccount(array $params): string {
    $zone = Capsule::table('mod_dns_zones')
        ->where('service_id', $params['serviceid'])
        ->first();

    if ($zone) {
        $this->api->deleteZone($zone->domain);
        Capsule::table('mod_dns_zones')
            ->where('id', $zone->id)
            ->delete();
    }

    return 'success';
}
```

## Record Operations

```php
public function addRecord(int $zoneId, array $record): array {
    $zone = Capsule::table('mod_dns_zones')->where('id', $zoneId)->first();

    $result = $this->api->addRecord($zone->domain, [
        'name' => $record['name'],
        'type' => $record['type'],
        'value' => $record['value'],
        'ttl' => $record['ttl'] ?? $zone->ttl,
        'priority' => $record['priority'] ?? 0,
    ]);

    Capsule::table('mod_dns_records')->insert([
        'zone_id' => $zoneId,
        'name' => $record['name'],
        'type' => $record['type'],
        'value' => $record['value'],
        'ttl' => $record['ttl'] ?? $zone->ttl,
        'priority' => $record['priority'] ?? 0,
        'remote_id' => $result['id'],
        'created_at' => date('Y-m-d H:i:s'),
    ]);

    return $result;
}

public function deleteRecord(int $recordId): bool {
    $record = Capsule::table('mod_dns_records')->where('id', $recordId)->first();

    if (!$record) {
        return false;
    }

    $this->api->deleteRecord($record->remote_id);

    Capsule::table('mod_dns_records')->where('id', $recordId)->delete();

    return true;
}

public function updateRecord(int $recordId, array $data): bool {
    $record = Capsule::table('mod_dns_records')->where('id', $recordId)->first();

    if (!$record) {
        return false;
    }

    $this->api->updateRecord($record->remote_id, $data);

    Capsule::table('mod_dns_records')
        ->where('id', $recordId)
        ->update($data);

    return true;
}
```

## Dynamic DNS

```php
public function dynamicUpdate(int $zoneId, string $ipAddress): bool {
    $zone = Capsule::table('mod_dns_zones')->where('id', $zoneId)->first();

    $record = Capsule::table('mod_dns_records')
        ->where('zone_id', $zoneId)
        ->where('type', 'A')
        ->where('name', '@')
        ->first();

    if ($record) {
        $this->api->updateRecord($record->remote_id, [
            'value' => $ipAddress,
        ]);
    } else {
        $this->api->addRecord($zone->domain, [
            'name' => '@',
            'type' => 'A',
            'value' => $ipAddress,
        ]);
    }

    return true;
}

add_hook('ServiceChangePackage', 1, function($vars) {
    if ($vars['params']['configoptions']['DynamicDNS'] ?? false) {
        $module = new DNSModule();
        $module->dynamicUpdate($vars['serviceid'], $_SERVER['REMOTE_ADDR']);
    }
});
```

## Record Types Support

```php
public function getSupportedRecordTypes(): array {
    return [
        'A', 'AAAA', 'CNAME', 'MX', 'TXT',
        'NS', 'SRV', 'CAA', 'DNSKEY',
        'DS', 'TLSA', 'URI',
    ];
}

public function validateRecord(string $type, string $value): bool {
    $validators = [
        'A' => fn($v) => filter_var($v, FILTER_VALIDATE_IP, FILTER_FLAG_IPV4),
        'AAAA' => fn($v) => filter_var($v, FILTER_VALIDATE_IP, FILTER_FLAG_IPV6),
        'CNAME' => fn($v) => preg_match('/^[a-z0-9.-]+\.[a-z]{2,}$/i', $v),
        'MX' => fn($v) => preg_match('/^\d+\s+[a-z0-9.-]+\.[a-z]{2,}$/i', $v),
        'TXT' => fn($v) => strlen($v) <= 255 && strlen($v) >= 1,
    ];

    return $validators[$type]($value) ?? true;
}
```

## Client Area

```php
function {module}_ClientArea(array $params): array {
    $zone = Capsule::table('mod_dns_zones')
        ->where('service_id', $params['serviceid'])
        ->first();

    $records = [];
    if ($zone) {
        $records = Capsule::table('mod_dns_records')
            ->where('zone_id', $zone->id)
            ->orderBy('type')
            ->get();
    }

    return [
        'pagetitle' => 'DNS Management',
        'templatefile' => 'templates/dns_clientarea',
        'vars' => [
            'zone' => $zone,
            'records' => $records,
            'record_types' => $this->getSupportedRecordTypes(),
        ],
    ];
}
```

---

**Related Skills:**
- whmcs-server-builder
- whmcs-api-integration
- whmcs-domain-sync