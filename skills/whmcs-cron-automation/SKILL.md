# WHMCS Cron Automation Skill
# Version: 1.0 | Updated: 2026-05-28

## Purpose

Guide for creating automated tasks and cron jobs for WHMCS modules.

## When to Use

- Setting up scheduled synchronization
- Creating automated reports
- Implementing cleanup tasks
- Running periodic health checks

## Cron Setup

### Addon Cron

```php
<?php
// modules/addons/{module}/cron.php
if (!defined("WHMCS")) { die("Direct access denied"); }

function {module}_cron() {
    // Get cron settings
    $settings = Capsule::table('mod_{module}_settings')
        ->pluck('setting_value', 'setting_key')
        ->toArray();

    // Run tasks based on frequency
    $lastRun = $settings['last_cron_run'] ?? null;
    $frequency = $settings['cron_frequency'] ?? 'daily';

    if ($frequency === 'hourly' || shouldRunNow($lastRun, $frequency)) {
        runHourlyTasks();
    }

    if ($frequency === 'daily' || shouldRunNow($lastRun, 'daily')) {
        runDailyTasks();
    }

    // Update last run time
    updateSetting('last_cron_run', date('Y-m-d H:i:s'));
}

function runHourlyTasks(): void {
    // Quick sync tasks
    // Queue processing
    // Cleanup temp files
}

function runDailyTasks(): void {
    // Generate reports
    // Sync data with external systems
    // Archive old records
    // Send notifications
}

function shouldRunNow(?string $lastRun, string $frequency): bool {
    if (!$lastRun) return true;

    $last = strtotime($lastRun);
    $now = time();

    return match ($frequency) {
        'hourly' => ($now - $last) >= 3600,
        'daily' => ($now - $last) >= 86400,
        'weekly' => ($now - $last) >= 604800,
        default => false,
    };
}

// Run if accessed directly
if (php_sapi_name() === 'cli' || isset($_GET['force'])) {
    {module}_cron();
}
```

### Hook-based Cron

```php
<?php
// hooks.php
add_hook('DailyCronJob', 1, function($vars) {
    runModuleDailyTasks();
});

add_hook('HourlyCronJob', 1, function($vars) {
    runModuleHourlyTasks();
});
```

### WHMCS Cron Registration

```php
// In config function
'fields' => [
    'cron_enabled' => [
        'FriendlyName' => 'Enable Cron',
        'Type' => 'yesno',
    ],
    'cron_frequency' => [
        'FriendlyName' => 'Cron Frequency',
        'Type' => 'dropdown',
        'Options' => 'hourly,daily,weekly',
        'Default' => 'daily',
    ],
]
```

## Common Cron Tasks

### Sync Task

```php
function syncExternalData(): void {
    $servers = Capsule::table('tblservers')
        ->where('type', 'modulename')
        ->where('disabled', 0)
        ->get();

    foreach ($servers as $server) {
        try {
            $api = new ApiClient($server);
            $data = $api->getServerList();

            foreach ($data as $item) {
                syncServerStatus($item);
            }

            logCronActivity('sync', 'Synced ' . count($data) . ' servers');
        } catch (\Exception $e) {
            logCronActivity('error', 'Sync failed: ' . $e->getMessage());
        }
    }
}
```

### Cleanup Task

```php
function cleanupOldData(): void {
    $retentionDays = 90;

    // Delete old logs
    Capsule::table('mod_{module}_logs')
        ->where('created_at', '<', date('Y-m-d H:i:s', strtotime("-{$retentionDays} days")))
        ->delete();

    // Archive old transactions
    Capsule::statement("
        INSERT INTO mod_{module}_archive
        SELECT * FROM mod_{module}_data
        WHERE created_at < DATE_SUB(NOW(), INTERVAL {$retentionDays} DAY)
        AND status = 'completed'
    ");

    Capsule::table('mod_{module}_data')
        ->where('created_at', '<', date('Y-m-d H:i:s', strtotime("-{$retentionDays} days")))
        ->where('status', 'completed')
        ->delete();
}
```

### Report Generation

```php
function generateDailyReport(): void {
    $yesterday = date('Y-m-d', strtotime('-1 day'));

    $stats = [
        'new_services' => Capsule::table('tblhosting')
            ->whereDate('regdate', $yesterday)
            ->count(),
        'suspended' => Capsule::table('tblhosting')
            ->where('domainstatus', 'Suspended')
            ->whereDate('updated_at', $yesterday)
            ->count(),
        'terminated' => Capsule::table('tblhosting')
            ->where('domainstatus', 'Terminated')
            ->whereDate('updated_at', $yesterday)
            ->count(),
    ];

    // Store report
    Capsule::table('mod_{module}_reports')->insert([
        'report_date' => $yesterday,
        'stats' => json_encode($stats),
        'created_at' => date('Y-m-d H:i:s'),
    ]);

    // Send email to admin
    sendReportEmail($stats);
}
```

## Cron Security

```php
function validateCronAuth(): bool {
    // Check if called from WHMCS cron
    $secret = $_GET['secret'] ?? '';

    if (empty($secret)) {
        return false;
    }

    $validSecret = Capsule::table('mod_{module}_settings')
        ->where('setting_key', 'cron_secret')
        ->value('setting_value');

    return hash_equals($validSecret, $secret);
}

// Protect cron endpoint
if (!validateCronAuth()) {
    die('Unauthorized');
}
```

## Checklist

- [ ] Cron file created
- [ ] Daily/Hourly hooks registered
- [ ] Error handling with logging
- [ ] Timeout protection
- [ ] Idempotent operations
- [ ] Email notifications

---

**Related Skills:**
- whmcs-hooks-development
- whmcs-automation
- whmcs-reporting