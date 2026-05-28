# WHMCS Monitoring Skill
# Version: 1.0 | Updated: 2026-05-28

## Purpose

Guide for implementing monitoring and alerting in WHMCS modules.

## When to Use

- Monitoring service health
- Alerting on issues
- Tracking SLA metrics

## Monitoring Patterns

### Health Checks

```php
function performHealthCheck(): array {
    $results = [
        'api_connection' => checkApiConnection(),
        'database' => checkDatabase(),
        'queue_size' => checkQueueSize(),
        'error_rate' => checkErrorRate(),
    ];

    $overall = !in_array(false, array_column($results, 'healthy'));

    Capsule::table('mod_{module}_health')->insert([
        'healthy' => $overall,
        'results' => json_encode($results),
        'checked_at' => date('Y-m-d H:i:s'),
    ]);

    if (!$overall) {
        sendAlert('Health check failed', $results);
    }

    return $results;
}

function checkApiConnection(): array {
    try {
        $api = new ApiClient($this->params);
        $result = $api->ping();

        return ['healthy' => true, 'latency' => $result['latency'] ?? 0];
    } catch (\Exception $e) {
        return ['healthy' => false, 'error' => $e->getMessage()];
    }
}

function checkDatabase(): array {
    try {
        Capsule::select('SELECT 1');
        return ['healthy' => true];
    } catch (\Exception $e) {
        return ['healthy' => false, 'error' => 'Database connection failed'];
    }
}

function checkQueueSize(): array {
    $pending = Capsule::table('mod_{module}_queue')
        ->where('status', 'pending')
        ->count();

    $threshold = 100;

    return [
        'healthy' => $pending < $threshold,
        'pending' => $pending,
        'threshold' => $threshold,
    ];
}
```

### Alert System

```php
function sendAlert(string $subject, array $data): void {
    $alertConfig = getAlertConfig();

    if (!$alertConfig['enabled']) {
        return;
    }

    $message = buildAlertMessage($subject, $data);

    // Email alert
    if ($alertConfig['email_enabled']) {
        sendAdminEmail($alertConfig['email'], $subject, $message);
    }

    // SMS alert (if configured)
    if ($alertConfig['sms_enabled']) {
        sendAdminSms($alertConfig['phone'], $subject);
    }

    // Log alert
    logAlert($subject, $data);
}

function buildAlertMessage(string $subject, array $data): string {
    $message = "Alert: {$subject}\n\n";
    $message .= "Time: " . date('Y-m-d H:i:s') . "\n\n";
    $message .= "Details:\n";

    foreach ($data as $key => $value) {
        $message .= "- {$key}: " . (is_array($value) ? json_encode($value) : $value) . "\n";
    }

    return $message;
}
```

### Dashboard Widget

```php
function {module}_widget_dashboard(): array {
    $health = getLatestHealthCheck();
    $stats = getModuleStats();

    return [
        'variables' => [
            'health_status' => $health['healthy'] ? 'OK' : 'ISSUE',
            'api_status' => $health['api_connection']['healthy'] ?? false,
            'queue_pending' => $stats['queue_pending'] ?? 0,
            'error_count' => $stats['errors_today'] ?? 0,
        ],
        'template' => 'admin/dashboard-widget.tpl',
        'name' => '{Module} Status',
    ];
}
```

## Checklist

- [ ] Health check function
- [ ] Alert system
- [ ] Dashboard widget
- [ ] Scheduled monitoring
- [ ] Alert throttling

---

**Related Skills:**
- whmcs-cron-automation
- whmcs-notification-builder
- whmcs-reporting