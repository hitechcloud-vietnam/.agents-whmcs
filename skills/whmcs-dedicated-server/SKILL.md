# WHMCS Dedicated Server Module Skill
# Version: 1.0 | Updated: 2026-05-28

## Purpose

Build dedicated server provisioning modules with advanced hardware control.

## Dedicated Server Module Structure

```php
<?php
/**
 * Dedicated Server Module
 * Location: modules/servers/{module}/
 */

function {module}_MetaData(): array {
    return [
        'DisplayName' => 'Dedicated Server',
        'APIVersion' => '1.1',
        'RequiresServer' => true,
        'DefaultNonSSLPort' => 443,
        'Parameters' => ['api_key', 'server_location'],
    ];
}

function {module}_ConfigOptions(array $params): array {
    return [
        'ServerSpec' => [
            'Type' => 'dropdown',
            'Options' => 'entry,pro,elite,enterprise',
        ],
        'ControlPanel' => [
            'Type' => 'dropdown',
            'Options' => 'none,cpanel,plesk,directadmin',
            'Default' => 'none',
        ],
        'BackupStorage' => [
            'Type' => 'dropdown',
            'Options' => '0GB,500GB,1TB,2TB',
            'Default' => '0GB',
        ],
        'IPAddresses' => [
            'Type' => 'dropdown',
            'Options' => '1,2,4,8',
            'Default' => '1',
        ],
    ];
}
```

## Server Lifecycle

```php
function {module}_CreateAccount(array $params): string {
    $spec = $params['configoption1'];

    $server = $this->api->provisionServer([
        'hostname' => $params['domain'],
        'spec' => $spec,
        'location' => $params['serveraccesshash'],
        'control_panel' => $params['configoption2'],
        'backup_storage' => $this->parseStorage($params['configoption3']),
        'ip_count' => (int) $params['configoption4'],
    ]);

    Capsule::table('mod_dedicated_servers')->insert([
        'service_id' => $params['serviceid'],
        'server_id' => $server['id'],
        'primary_ip' => $server['primary_ip'],
        'additional_ips' => json_encode($server['additional_ips'] ?? []),
        'spec' => $spec,
        'control_panel' => $params['configoption2'],
        'bandwidth_limit' => $server['bandwidth_limit'],
        'status' => 'provisioning',
        'created_at' => date('Y-m-d H:i:s'),
    ]);

    return 'success';
}

function {module}_SuspendAccount(array $params): string {
    $server = $this->getServer($params['serviceid']);

    $this->api->stopServer($server['server_id']);

    Capsule::table('mod_dedicated_servers')
        ->where('service_id', $params['serviceid'])
        ->update(['status' => 'suspended']);

    return 'success';
}

function {module}_UnsuspendAccount(array $params): string {
    $server = $this->getServer($params['serviceid']);

    $this->api->startServer($server['server_id']);

    Capsule::table('mod_dedicated_servers')
        ->where('service_id', $params['serviceid'])
        ->update(['status' => 'active']);

    return 'success';
}

function {module}_TerminateAccount(array $params): string {
    $server = $this->getServer($params['serviceid']);

    if ($server) {
        $this->api->destroyServer($server['server_id']);

        Capsule::table('mod_dedicated_servers')
            ->where('service_id', $params['serviceid'])
            ->update(['status' => 'terminated']);
    }

    return 'success';
}
```

## Hardware Control

```php
public function getServerPowerState(int $serviceId): string {
    $server = $this->getServer($serviceId);
    $state = $this->api->getPowerState($server['server_id']);

    return $state['state'];
}

public function powerOn(int $serviceId): string {
    $server = $this->getServer($serviceId);
    $this->api->setPowerState($server['server_id'], 'on');

    return 'success';
}

public function powerOff(int $serviceId): string {
    $server = $this->getServer($serviceId);
    $this->api->setPowerState($server['server_id'], 'off');

    return 'success';
}

public function powerReset(int $serviceId): string {
    $server = $this->getServer($serviceId);
    $this->api->setPowerState($server['server_id'], 'reset');

    return 'success';
}

public function bootToRescue(int $serviceId): string {
    $server = $this->getServer($serviceId);
    $this->api->setBootMode($server['server_id'], 'rescue');

    return 'success';
}
```

## Remote Management

```php
public function getIPMIInfo(int $serviceId): array {
    $server = $this->getServer($serviceId);

    $ipmi = $this->api->getIPMI($server['server_id']);

    return [
        'ip' => $ipmi['ip'],
        'port' => $ipmi['port'],
        'username' => $ipmi['username'],
        'password' => decrypt($ipmi['password']),
    ];
}

public function getSerialConsole(int $serviceId): array {
    $server = $this->getServer($serviceId);

    $console = $this->api->getSerialConsole($server['server_id']);

    return [
        'url' => $console['url'],
        'token' => $console['token'],
        'expires' => $console['expires'],
    ];
}
```

## Bandwidth Tracking

```php
public function getBandwidthUsage(int $serviceId): array {
    $server = $this->getServer($serviceId);

    $usage = $this->api->getBandwidthUsage($server['server_id']);

    Capsule::table('mod_dedicated_bw_history')->insert([
        'service_id' => $serviceId,
        'incoming' => $usage['incoming'],
        'outgoing' => $usage['outgoing'],
        'total' => $usage['total'],
        'recorded_at' => date('Y-m-d H:i:s'),
    ]);

    return [
        'incoming' => $this->formatBytes($usage['incoming']),
        'outgoing' => $this->formatBytes($usage['outgoing']),
        'total' => $this->formatBytes($usage['total']),
        'limit' => $this->formatBytes($server['bandwidth_limit']),
        'percent' => round(($usage['total'] / $server['bandwidth_limit']) * 100, 2),
    ];
}
```

## Client Area

```php
function {module}_ClientArea(array $params): array {
    $server = $this->getServer($params['serviceid']);
    $powerState = $this->getServerPowerState($params['serviceid']);
    $bandwidth = $this->getBandwidthUsage($params['serviceid']);

    try {
        $ipmi = $this->getIPMIInfo($params['serviceid']);
    } catch (\Exception $e) {
        $ipmi = null;
    }

    $recentTasks = Capsule::table('mod_dedicated_tasks')
        ->where('service_id', $params['serviceid'])
        ->orderBy('created_at', 'desc')
        ->limit(10)
        ->get();

    return [
        'pagetitle' => 'Dedicated Server',
        'templatefile' => 'templates/dedicated_clientarea',
        'vars' => [
            'server' => $server,
            'power_state' => $powerState,
            'bandwidth' => $bandwidth,
            'ipmi' => $ipmi,
            'recent_tasks' => $recentTasks,
            'can_power_on' => $powerState === 'off',
            'can_power_off' => $powerState === 'on',
        ],
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

private function parseStorage(string $value): int {
    if ($value === '0GB') return 0;
    return (int) str_replace(['GB', 'TB'], ['', ''], $value) *
        (strpos($value, 'TB') !== false ? 1024 : 1) * 1024 * 1024 * 1024;
}
```

---

**Related Skills:**
- whmcs-server-builder
- whmcs-monitoring
- whmcs-vps-module