# WHMCS Alert Management Workflow

## Overview
This workflow covers setting up and managing alerts for WHMCS.

## Step 1: Alert Service

```php
<?php
// src/Service/AlertService.php

namespace WHMCS\Module\Addon\YourModule\Service;

use WHMCS\Database\Capsule;

class AlertService
{
    private $alertChannels = [];

    public function registerChannel(string $name, callable $handler): void
    {
        $this->alertChannels[$name] = $handler;
    }

    public function triggerAlert(string $severity, string $title, string $message, array $context = []): void
    {
        $alertId = Capsule::table('mod_alerts')->insertGetId([
            'severity' => $severity,
            'title' => $title,
            'message' => $message,
            'context' => json_encode($context),
            'status' => 'active',
            'triggered_at' => date('Y-m-d H:i:s')
        ]);

        // Send to registered channels
        foreach ($this->alertChannels as $channel => $handler) {
            try {
                $handler($severity, $title, $message, $context);
                $this->logAlertSent($alertId, $channel, 'success');
            } catch (\Exception $e) {
                $this->logAlertSent($alertId, $channel, 'failed', $e->getMessage());
            }
        }
    }

    public function checkThresholds(): void
    {
        $this->checkDiskSpace();
        $this->checkDatabaseSize();
        $this->checkFailedLogins();
        $this->checkPendingOrders();
        $this->checkServiceHealth();
    }

    private function checkDiskSpace(): void
    {
        $freeSpace = disk_free_space('/');
        $totalSpace = disk_total_space('/');
        $percentFree = ($freeSpace / $totalSpace) * 100;

        if ($percentFree < 10) {
            $this->triggerAlert('critical', 'Low Disk Space', 'Disk space below 10%', [
                'percent_free' => round($percentFree, 2),
                'free_gb' => round($freeSpace / 1024 / 1024 / 1024, 2)
            ]);
        } elseif ($percentFree < 20) {
            $this->triggerAlert('warning', 'Disk Space Warning', 'Disk space below 20%', [
                'percent_free' => round($percentFree, 2)
            ]);
        }
    }

    private function checkDatabaseSize(): void
    {
        $size = Capsule::connection()->select(
            "SELECT ROUND(SUM(data_length + index_length) / 1024 / 1024, 2) AS size FROM information_schema.tables WHERE table_schema = ?",
            [Capsule::config('db_name')]
        );

        $dbSize = $size[0]->size ?? 0;

        if ($dbSize > 5000) { // 5GB
            $this->triggerAlert('warning', 'Large Database', 'Database size exceeds 5GB', [
                'size_mb' => $dbSize
            ]);
        }
    }

    private function checkFailedLogins(): void
    {
        $recentFailures = Capsule::table('tblactivitylog')
            ->where('description', 'like', '%Failed Login%')
            ->where('date', '>=', date('Y-m-d H:i:s', strtotime('-1 hour')))
            ->count();

        if ($recentFailures > 10) {
            $this->triggerAlert('warning', 'High Login Failure Rate', 'Multiple failed login attempts detected', [
                'failures_in_last_hour' => $recentFailures
            ]);
        }
    }

    private function checkPendingOrders(): void
    {
        $pendingOrders = Capsule::table('tblorders')
            ->where('status', 'Pending')
            ->where('date', '<', date('Y-m-d H:i:s', strtotime('-24 hours')))
            ->count();

        if ($pendingOrders > 20) {
            $this->triggerAlert('info', 'Pending Orders', 'Many orders pending review', [
                'pending_count' => $pendingOrders
            ]);
        }
    }

    private function checkServiceHealth(): void
    {
        $suspendedServices = Capsule::table('tblhosting')
            ->where('domainstatus', 'Suspended')
            ->where('suspendreason', 'like', '%Overdue%')
            ->count();

        if ($suspendedServices > 50) {
            $this->triggerAlert('info', 'Suspended Services', 'Many services suspended for non-payment', [
                'suspended_count' => $suspendedServices
            ]);
        }
    }

    private function logAlertSent(int $alertId, string $channel, string $status, string $error = null): void
    {
        Capsule::table('mod_alert_notifications')->insert([
            'alert_id' => $alertId,
            'channel' => $channel,
            'status' => $status,
            'error' => $error,
            'sent_at' => date('Y-m-d H:i:s')
        ]);
    }

    public function acknowledgeAlert(int $alertId, string $acknowledgedBy): void
    {
        Capsule::table('mod_alerts')
            ->where('id', $alertId)
            ->update([
                'status' => 'acknowledged',
                'acknowledged_by' => $acknowledgedBy,
                'acknowledged_at' => date('Y-m-d H:i:s')
            ]);
    }

    public function resolveAlert(int $alertId, string $resolvedBy, string $notes = ''): void
    {
        Capsule::table('mod_alerts')
            ->where('id', $alertId)
            ->update([
                'status' => 'resolved',
                'resolved_by' => $resolvedBy,
                'resolved_at' => date('Y-m-d H:i:s'),
                'resolution_notes' => $notes
            ]);
    }

    public function getActiveAlerts(): array
    {
        return Capsule::table('mod_alerts')
            ->whereIn('status', ['active', 'acknowledged'])
            ->orderByRaw("FIELD(severity, 'critical', 'high', 'medium', 'low')")
            ->get()
            ->toArray();
    }
}
```

## Step 2: Alert Channels

```php
<?php
// includes/hooks/alert_channels.php

use WHMCS\Module\Addon\YourModule\Service\AlertService;

$alertService = new AlertService();

// Email alerts
$alertService->registerChannel('email', function($severity, $title, $message, $context) {
    $admins = Capsule::table('tbladmins')->where('roleid', 1)->get();

    foreach ($admins as $admin) {
        send_email('SystemAlert', $admin->email, [
            'severity' => $severity,
            'title' => $title,
            'message' => $message,
            'context' => $context
        ]);
    }
});

// Slack alerts
$alertService->registerChannel('slack', function($severity, $title, $message, $context) {
    $webhookUrl = Capsule::config('slack_alert_webhook');

    $severityEmoji = match($severity) {
        'critical' => ':rotating_light:',
        'high' => ':warning:',
        'medium' => ':large_yellow_circle:',
        'low' => ':information_source:',
        default => ':bell:'
    };

    $ch = curl_init($webhookUrl);
    curl_setopt_array($ch, [
        CURLOPT_POST => true,
        CURLOPT_POSTFIELDS => json_encode([
            'text' => "{$severityEmoji} WHMCS Alert: {$title}",
            'blocks' => [
                ['type' => 'section', 'text' => ['type' => 'mrkdwn', 'text' => "*$title*\n$message"]],
                ['type' => 'context', 'elements' => [
                    ['type' => 'mrkdwn', 'text' => "Severity: $severity | Time: " . date('Y-m-d H:i:s')]
                ]]
            ]
        ]),
        CURLOPT_RETURNTRANSFER => true
    ]);
    curl_exec($ch);
    curl_close($ch);
});
```

## Verification Checklist

- [ ] Alert service implemented
- [ ] Alert channels registered
- [ ] Threshold checks working
- [ ] Alert acknowledgment working
- [ ] Email alerts sending
- [ ] Slack alerts sending
