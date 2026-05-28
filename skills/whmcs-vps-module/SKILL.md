# WHMCS VPS Module Skill
# Version: 1.0 | Updated: 2026-05-28

## Purpose

Build VPS/provisioning modules with advanced features like console access, snapshots, and resource management.

## VPS Module Structure

```php
<?php
/**
 * VPS Module
 * Location: modules/servers/{module}/
 */

function {module}_MetaData(): array {
    return [
        'DisplayName' => 'VPS Provider',
        'APIVersion' => '1.1',
        'RequiresServer' => true,
        'DefaultNonSSLPort' => 443,
        'Parameters' => ['api_key', 'api_secret', 'location_id'],
    ];
}

function {module}_ConfigOptions(array $params): array {
    return [
        'Plan' => [
            'Type' => 'dropdown',
            'Options' => 'vps-s,vps-m,vps-l,vps-xl',
        ],
        'OS' => [
            'Type' => 'dropdown',
            'Options' => 'ubuntu-22.04,ubuntu-24.04,debian-12,centos-stream9',
        ],
        'EnableBackups' => [
            'Type' => 'yesno',
            'Description' => 'Enable automatic backups',
        ],
        'EnableMonitoring' => [
            'Type' => 'yesno',
            'Description' => 'Enable server monitoring',
        ],
    ];
}
```

## VPS Lifecycle

```php
function {module}_CreateAccount(array $params): string {
    $vps = $this->api->createInstance([
        'hostname' => $params['domain'],
        'plan' => $params['configoption1'],
        'os' => $params['configoption2'],
        'location' => $params['serveraccesshash'],
        'ssh_key' => $params['configoptions']['SSHKey'] ?? null,
    ]);

    Capsule::table('mod_vps_instances')->insert([
        'service_id' => $params['serviceid'],
        'instance_id' => $vps['id'],
        'vmid' => $vps['vmid'],
        'ip_address' => $vps['ip'],
        'ipv6_address' => $vps['ipv6'] ?? null,
        'plan' => $params['configoption1'],
        'os' => $params['configoption2'],
        'status' => 'provisioning',
        'created_at' => date('Y-m-d H:i:s'),
    ]);

    return 'success';
}

function {module}_SuspendAccount(array $params): string {
    $instance = $this->getInstance($params['serviceid']);
    $this->api->stopInstance($instance['instance_id']);

    Capsule::table('mod_vps_instances')
        ->where('service_id', $params['serviceid'])
        ->update(['status' => 'suspended']);

    return 'success';
}

function {module}_UnsuspendAccount(array $params): string {
    $instance = $this->getInstance($params['serviceid']);
    $this->api->startInstance($instance['instance_id']);

    Capsule::table('mod_vps_instances')
        ->where('service_id', $params['serviceid'])
        ->update(['status' => 'active']);

    return 'success';
}

function {module}_TerminateAccount(array $params): string {
    $instance = $this->getInstance($params['serviceid']);

    if ($instance) {
        $this->api->deleteInstance($instance['instance_id']);

        Capsule::table('mod_vps_instances')
            ->where('service_id', $params['serviceid'])
            ->update(['status' => 'terminated']);
    }

    return 'success';
}

function {module}_ChangePassword(array $params): string {
    $instance = $this->getInstance($params['serviceid']);

    $this->api->changeRootPassword($instance['instance_id'], decrypt($params['password']));

    logActivity("VPS password changed for service {$params['serviceid']}");

    return 'success';
}
```

## Advanced Operations

### Console Access
```php
public function getConsoleUrl(int $serviceId): array {
    $instance = $this->getInstance($serviceId);

    $console = $this->api->getVNCConsole($instance['instance_id']);

    return [
        'url' => $console['url'],
        'expires' => $console['expires'],
        'token' => $console['token'],
    ];
}
```

### Snapshots
```php
public function createSnapshot(int $serviceId, string $description): array {
    $instance = $this->getInstance($serviceId);

    $snapshot = $this->api->createSnapshot($instance['instance_id'], [
        'description' => $description,
    ]);

    Capsule::table('mod_vps_snapshots')->insert([
        'service_id' => $serviceId,
        'snapshot_id' => $snapshot['id'],
        'description' => $description,
        'size' => $snapshot['size'],
        'status' => 'creating',
        'created_at' => date('Y-m-d H:i:s'),
    ]);

    return $snapshot;
}

public function restoreSnapshot(int $serviceId, string $snapshotId): string {
    $instance = $this->getInstance($serviceId);

    $this->api->restoreSnapshot($instance['instance_id'], $snapshotId);

    return 'success';
}
```

### Resource Management
```php
public function resizeInstance(int $serviceId, string $newPlan): string {
    $instance = $this->getInstance($serviceId);

    $currentPlan = Capsule::table('mod_vps_instances')
        ->where('service_id', $serviceId)
        ->first();

    Capsule::table('mod_vps_instances')
        ->where('service_id', $serviceId)
        ->update(['plan' => $newPlan, 'pending_resize' => true]);

    $this->api->resizeInstance($instance['instance_id'], $newPlan);

    logActivity("VPS {$serviceId} resize initiated from {$currentPlan->plan} to {$newPlan}");

    return 'success';
}
```

## Statistics & Monitoring

```php
public function getStats(int $serviceId): array {
    $instance = $this->getInstance($serviceId);

    $stats = $this->api->getInstanceStats($instance['instance_id']);

    Capsule::table('mod_vps_stats')->insert([
        'service_id' => $serviceId,
        'cpu_percent' => $stats['cpu'],
        'ram_used' => $stats['ram_used'],
        'ram_total' => $stats['ram_total'],
        'disk_used' => $stats['disk_used'],
        'disk_total' => $stats['disk_total'],
        'bandwidth_used' => $stats['bw_used'],
        'bandwidth_total' => $stats['bw_total'],
        'recorded_at' => date('Y-m-d H:i:s'),
    ]);

    return $stats;
}
```

## Client Area

```php
function {module}_ClientArea(array $params): array {
    $instance = $this->getInstance($params['serviceid']);
    $stats = $this->getStats($params['serviceid']);
    $console = $this->getConsoleUrl($params['serviceid']);

    $snapshots = Capsule::table('mod_vps_snapshots')
        ->where('service_id', $params['serviceid'])
        ->where('status', 'completed')
        ->orderBy('created_at', 'desc')
        ->limit(5)
        ->get();

    return [
        'pagetitle' => 'VPS Control Panel',
        'templatefile' => 'templates/vps_clientarea',
        'vars' => [
            'instance' => $instance,
            'stats' => $stats,
            'console' => $console,
            'snapshots' => $snapshots,
            'actions' => [
                'start' => $instance['status'] !== 'running',
                'stop' => $instance['status'] === 'running',
                'restart' => $instance['status'] === 'running',
            ],
        ],
    ];
}
```

## AJAX Actions

```php
add_hook('VPSAjaxAction', 1, function($vars) {
    $action = $vars['action'];
    $serviceId = $vars['serviceid'];

    switch ($action) {
        case 'start':
            return $this->startVps($serviceId);
        case 'stop':
            return $this->stopVps($serviceId);
        case 'restart':
            return $this->restartVps($serviceId);
        case 'console':
            return $this->getConsoleUrl($serviceId);
        case 'snapshot':
            return $this->createSnapshot($serviceId, $vars['description'] ?? 'Manual snapshot');
        default:
            return ['error' => 'Unknown action'];
    }
});
```

---

**Related Skills:**
- whmcs-server-builder
- whmcs-monitoring
- whmcs-clientarea-builder