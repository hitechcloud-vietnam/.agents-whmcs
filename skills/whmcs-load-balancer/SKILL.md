# WHMCS Load Balancer Module Skill
# Version: 1.0 | Updated: 2026-05-28

## Purpose

Build load balancer provisioning modules with health checks and scaling.

## Load Balancer Module Structure

```php
<?php
/**
 * Load Balancer Module
 * Location: modules/servers/{module}/
 */

function {module}_MetaData(): array {
    return [
        'DisplayName' => 'Load Balancer',
        'APIVersion' => '1.0',
        'RequiresServer' => true,
        'Parameters' => ['api_key', 'region'],
    ];
}

function {module}_ConfigOptions(array $params): array {
    return [
        'Algorithm' => [
            'Type' => 'dropdown',
            'Options' => 'round_robin,least_conn,ip_hash,weighted',
            'Default' => 'round_robin',
        ],
        'HealthCheck' => [
            'Type' => 'dropdown',
            'Options' => 'http,https,tcp,ping',
            'Default' => 'http',
        ],
        'StickySessions' => [
            'Type' => 'yesno',
            'Description' => 'Enable session persistence',
        ],
        'SSLOffload' => [
            'Type' => 'yesno',
            'Description' => 'Enable SSL termination',
        ],
    ];
}
```

## Load Balancer Operations

```php
function {module}_CreateAccount(array $params): string {
    $lb = $this->api->createLoadBalancer([
        'algorithm' => $params['configoption1'],
        'health_check' => [
            'type' => $params['configoption2'],
            'path' => '/health',
            'interval' => 30,
            'timeout' => 5,
            'healthy_threshold' => 2,
            'unhealthy_threshold' => 3,
        ],
        'sticky_sessions' => $params['configoption3'] === 'on',
        'ssl_offload' => $params['configoption4'] === 'on',
    ]);

    Capsule::table('mod_loadbalancer')->insert([
        'service_id' => $params['serviceid'],
        'lb_id' => $lb['id'],
        'vip' => $lb['vip'],
        'algorithm' => $params['configoption1'],
        'health_check' => $params['configoption2'],
        'sticky_sessions' => $params['configoption3'] === 'on',
        'ssl_offload' => $params['configoption4'] === 'on',
        'status' => 'active',
        'created_at' => date('Y-m-d H:i:s'),
    ]);

    return 'success';
}

function {module}_SuspendAccount(array $params): string {
    $lb = $this->getLoadBalancer($params['serviceid']);

    $this->api->disableLoadBalancer($lb['lb_id']);

    Capsule::table('mod_loadbalancer')
        ->where('service_id', $params['serviceid'])
        ->update(['status' => 'suspended']);

    return 'success';
}

function {module}_UnsuspendAccount(array $params): string {
    $lb = $this->getLoadBalancer($params['serviceid']);

    $this->api->enableLoadBalancer($lb['lb_id']);

    Capsule::table('mod_loadbalancer')
        ->where('service_id', $params['serviceid'])
        ->update(['status' => 'active']);

    return 'success';
}

function {module}_TerminateAccount(array $params): string {
    $lb = $this->getLoadBalancer($params['serviceid']);

    if ($lb) {
        $this->api->deleteLoadBalancer($lb['lb_id']);
        Capsule::table('mod_loadbalancer')
            ->where('id', $lb['id'])
            ->delete();
    }

    return 'success';
}
```

## Backend Management

```php
public function addBackend(int $serviceId, array $backend): int {
    $lb = $this->getLoadBalancer($serviceId);

    $node = $this->api->addBackend($lb['lb_id'], [
        'ip' => $backend['ip'],
        'port' => $backend['port'],
        'weight' => $backend['weight'] ?? 1,
        'enabled' => $backend['enabled'] ?? true,
    ]);

    Capsule::table('mod_lb_backends')->insert([
        'lb_id' => $lb['id'],
        'remote_id' => $node['id'],
        'ip' => $backend['ip'],
        'port' => $backend['port'],
        'weight' => $backend['weight'] ?? 1,
        'enabled' => $backend['enabled'] ?? true,
        'created_at' => date('Y-m-d H:i:s'),
    ]);

    return $node['id'];
}

public function removeBackend(int $serviceId, int $backendId): bool {
    $backend = Capsule::table('mod_lb_backends')->where('id', $backendId)->first();

    $lb = $this->getLoadBalancer($serviceId);
    $this->api->removeBackend($lb['lb_id'], $backend->remote_id);

    Capsule::table('mod_lb_backends')->where('id', $backendId)->delete();

    return true;
}

public function toggleBackend(int $backendId, bool $enabled): bool {
    $backend = Capsule::table('mod_lb_backends')->where('id', $backendId)->first();

    $lb = $this->getLoadBalancerById($backend->lb_id);
    $this->api->updateBackend($lb['lb_id'], $backend->remote_id, [
        'enabled' => $enabled,
    ]);

    Capsule::table('mod_lb_backends')
        ->where('id', $backendId)
        ->update(['enabled' => $enabled]);

    return true;
}
```

## Statistics

```php
public function getStats(int $serviceId): array {
    $lb = $this->getLoadBalancer($serviceId);

    $stats = $this->api->getStats($lb['lb_id']);

    Capsule::table('mod_lb_stats')->insert([
        'lb_id' => $lb['id'],
        'requests_total' => $stats['requests'],
        'requests_healthy' => $stats['healthy'],
        'requests_unhealthy' => $stats['unhealthy'],
        'bytes_in' => $stats['bytes_in'],
        'bytes_out' => $stats['bytes_out'],
        'recorded_at' => date('Y-m-d H:i:s'),
    ]);

    return [
        'vip' => $lb['vip'],
        'requests' => $stats['requests'],
        'healthy_backends' => $stats['healthy'],
        'unhealthy_backends' => $stats['unhealthy'],
        'bytes_in' => $this->formatBytes($stats['bytes_in']),
        'bytes_out' => $this->formatBytes($stats['bytes_out']),
    ];
}
```

---

**Related Skills:**
- whmcs-server-builder
- whmcs-monitoring
- whmcs-ssl-offload