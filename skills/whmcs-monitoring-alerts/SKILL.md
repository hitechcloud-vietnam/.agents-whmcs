# WHMCS Monitoring & Alerts Module Skill
# Version: 1.0 | Updated: 2026-05-28

## Purpose

Build server monitoring and alerting modules.

## Monitoring Module Structure

```php
<?php
/**
 * Monitoring Module (Addon)
 * Location: modules/addons/{module}/
 */

function {module}_config(): array {
    return [
        'name' => 'Server Monitoring',
        'description' => 'Real-time server monitoring and alerts',
        'version' => '1.0',
    ];
}

function {module}_activate(): array {
    Capsule::schema()->create('mod_monitoring_servers', function($t) {
        $t->increments('id');
        $t->integer('service_id')->unsigned();
        $t->string('monitor_type', 20);
        $t->string('check_interval', 10)->default('60s');
        $t->boolean('email_alerts')->default(true);
        $t->boolean('webhook_alerts')->default(false);
        $t->text('webhook_url');
        $t->timestamps();
    });

    Capsule::schema()->create('mod_monitoring_metrics', function($t) {
        $t->increments('id');
        $t->integer('server_id')->unsigned();
        $t->string('metric_name', 50);
        $t->decimal('value', 12, 4);
        $t->string('unit', 20);
        $t->timestamp('recorded_at');
        $t->index(['server_id', 'recorded_at']);
    });

    Capsule::schema()->create('mod_monitoring_alerts', function($t) {
        $t->increments('id');
        $t->integer('server_id')->unsigned();
        $t->string('alert_type', 50);
        $t->string('severity', 20);
        $t->text('message');
        $t->string('status', 20)->default('open');
        $t->timestamp('triggered_at');
        $t->timestamp('resolved_at')->nullable();
    });

    return ['status' => 'success', 'description' => 'Monitoring module activated'];
}

function {module}_deactivate(): array {
    Capsule::schema()->dropIfExists('mod_monitoring_servers');
    Capsule::schema()->dropIfExists('mod_monitoring_metrics');
    Capsule::schema()->dropIfExists('mod_monitoring_alerts');
    return ['status' => 'success'];
}
```

## Server Monitoring Setup

```php
public function addServer(int $serviceId, array $config): int {
    return Capsule::table('mod_monitoring_servers')->insertGetId([
        'service_id' => $serviceId,
        'monitor_type' => $config['type'] ?? 'ping',
        'check_interval' => $config['interval'] ?? '60s',
        'email_alerts' => $config['email_alerts'] ?? true,
        'webhook_alerts' => $config['webhook_alerts'] ?? false,
        'webhook_url' => $config['webhook_url'] ?? null,
        'created_at' => date('Y-m-d H:i:s'),
    ]);
}

public function removeServer(int $serviceId): bool {
    $server = Capsule::table('mod_monitoring_servers')
        ->where('service_id', $serviceId)
        ->first();

    if ($server) {
        Capsule::table('mod_monitoring_metrics')
            ->where('server_id', $server->id)
            ->delete();

        Capsule::table('mod_monitoring_alerts')
            ->where('server_id', $server->id)
            ->delete();

        Capsule::table('mod_monitoring_servers')
            ->where('id', $server->id)
            ->delete();
    }

    return true;
}
```

## Metric Collection

```php
public function recordMetrics(int $serviceId, array $metrics): void {
    $server = Capsule::table('mod_monitoring_servers')
        ->where('service_id', $serviceId)
        ->first();

    foreach ($metrics as $metric) {
        Capsule::table('mod_monitoring_metrics')->insert([
            'server_id' => $server->id,
            'metric_name' => $metric['name'],
            'value' => $metric['value'],
            'unit' => $metric['unit'] ?? '',
            'recorded_at' => date('Y-m-d H:i:s'),
        ]);
    }
}

public function getMetricHistory(int $serviceId, string $metric, int $hours = 24): array {
    $server = Capsule::table('mod_monitoring_servers')
        ->where('service_id', $serviceId)
        ->first();

    return Capsule::table('mod_monitoring_metrics')
        ->where('server_id', $server->id)
        ->where('metric_name', $metric)
        ->where('recorded_at', '>=', date('Y-m-d H:i:s', strtotime("-{$hours} hours")))
        ->orderBy('recorded_at')
        ->get();
}
```

## Alert Management

```php
public function createAlert(int $serviceId, string $type, string $severity, string $message): int {
    $alertId = Capsule::table('mod_monitoring_alerts')->insertGetId([
        'server_id' => $serviceId,
        'alert_type' => $type,
        'severity' => $severity,
        'message' => $message,
        'status' => 'open',
        'triggered_at' => date('Y-m-d H:i:s'),
    ]);

    $server = Capsule::table('mod_monitoring_servers')->where('id', $serviceId)->first();

    if ($server->email_alerts) {
        $this->sendEmailAlert($serviceId, $severity, $message);
    }

    if ($server->webhook_alerts && $server->webhook_url) {
        $this->sendWebhookAlert($server->webhook_url, $type, $severity, $message);
    }

    return $alertId;
}

public function resolveAlert(int $alertId, string $resolution = ''): bool {
    Capsule::table('mod_monitoring_alerts')
        ->where('id', $alertId)
        ->update([
            'status' => 'resolved',
            'resolution' => $resolution,
            'resolved_at' => date('Y-m-d H:i:s'),
        ]);

    return true;
}
```

## Cron Job for Monitoring

```php
add_hook('FiveMinuteCronJob', 1, function() {
    $servers = Capsule::table('mod_monitoring_servers')->get();

    foreach ($servers as $server) {
        $service = Capsule::table('tblhosting')
            ->where('id', $server->service_id)
            ->first();

        if ($service->domainstatus !== 'Active') {
            continue;
        }

        $monitor = new MonitoringModule();

        switch ($server->monitor_type) {
            case 'ping':
                $result = $monitor->checkPing($service->server->ip);
                break;
            case 'http':
                $result = $monitor->checkHttp('https://' . $service->domain);
                break;
            case 'port':
                $result = $monitor->checkPort($service->server->ip, 443);
                break;
        }

        if (!$result['up']) {
            $monitor->createAlert(
                $server->id,
                'downtime',
                'critical',
                "Server {$service->domain} is not responding"
            );
        }
    }
});
```

---

**Related Skills:**
- whmcs-addon-builder
- whmcs-webhook-integration
- whmcs-alerting