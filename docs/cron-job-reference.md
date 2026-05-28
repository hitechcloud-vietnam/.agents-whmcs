# WHMCS Cron Job Reference
# Version: 1.0 | Updated: 2026-05-28

## Purpose

Comprehensive reference for WHMCS cron jobs and automation.

## WHMCS Cron System

```
# WHMCS cron is typically at:
/usr/local/bin/php /path/to/whmcs/crons/cron.php

# Common cron schedules:
# Every minute: * * * * *
# Every 5 minutes: */5 * * * *
# Hourly: 0 * * * *
# Daily at midnight: 0 0 * * *
```

## Cron Hooks

```php
// Daily Cron Hook
add_hook('DailyCronJob', 1, function() {
    // Runs once per day
    runDailyMaintenance();
});

// Hourly Cron Hook
add_hook('HourlyCronJob', 1, function() {
    // Runs every hour
    runHourlyTasks();
});
```

## Common Cron Tasks

### Service Expiration Check
```php
add_hook('DailyCronJob', 1, function() {
    $expiringServices = Capsule::table('tblhosting')
        ->where('nextduedate', '<=', date('Y-m-d'))
        ->where('domainstatus', 'Active')
        ->get();

    foreach ($expiringServices as $service) {
        sendExpirationNotice($service);
    }
});
```

### Invoice Generation
```php
add_hook('DailyCronJob', 1, function() {
    $params = [
        'command' => 'createinvoices',
        'options' => ['days' => 7],
    ];

    localAPI($params['command'], $params['options']);
});
```

### Auto Suspension
```php
add_hook('DailyCronJob', 1, function() {
    $overdueServices = Capsule::table('tblhosting')
        ->join('tblinvoices', 'tblhosting.userid', '=', 'tblinvoices.userid')
        ->where('tblinvoices.status', 'Unpaid')
        ->where('tblinvoices.duedate', '<', date('Y-m-d', strtotime('-7 days')))
        ->where('tblhosting.domainstatus', 'Active')
        ->get();

    foreach ($overdueServices as $service) {
        localAPI('ModuleSuspend', ['serviceid' => $service->id]);
    }
});
```

## Custom Module Cron

```php
function {module}_cron(): void {
    $lastRun = Capsule::table('mod_{module}_meta')
        ->where('key', 'last_cron_run')
        ->value('value');

    // Process pending items
    $pending = Capsule::table('mod_{module}_queue')
        ->where('created_at', '>', $lastRun ?? '1970-01-01')
        ->where('status', 'pending')
        ->get();

    foreach ($pending as $item) {
        processQueueItem($item);
    }

    // Update last run time
    Capsule::table('mod_{module}_meta')->updateOrInsert(
        ['key' => 'last_cron_run'],
        ['value' => date('Y-m-d H:i:s')]
    );
}
```

## Monitoring Cron

```php
function checkCronHealth(): array {
    $lastDailyCron = Capsule::table('tblactivitylog')
        ->where('description', 'like', '%Cron Job%')
        ->orderBy('id', 'desc')
        ->first();

    return [
        'last_cron' => $lastDailyCron->created_at ?? null,
        'is_stale' => strtotime($lastDailyCron->created_at ?? '') < strtotime('-25 hours'),
    ];
}
```

---

**Related Skills:**
- whmcs-cron-automation
- whmcs-hooks-development
