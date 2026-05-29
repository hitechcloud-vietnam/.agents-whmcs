# WHMCS Maintenance Automation Workflow

## Overview
This workflow implements comprehensive system maintenance for WHMCS.

## Prerequisites
- WHMCS with root/admin access
- Cron configured
- Database access

## Step-by-Step Process

### Step 1: Create Maintenance Manager
```php
<?php
// /includes/maintenance/MaintenanceManager.php

namespace WHMCS\Maintenance;

class MaintenanceManager
{
    /**
     * Run daily maintenance
     */
    public function dailyMaintenance(): array
    {
        $results = [];

        $results['cleanup'] = $this->runCleanup();
        $results['optimize'] = $this->optimizeDatabase();
        $results['health_check'] = $this->runHealthCheck();
        $results['alerts'] = $this->checkAlerts();

        return $results;
    }

    /**
     * Run weekly maintenance
     */
    public function weeklyMaintenance(): array
    {
        $results = [];

        $results['backup'] = $this->runBackup();
        $results['stats'] = $this->updateStatistics();
        $results['reports'] = $this->generateWeeklyReport();

        return $results;
    }

    /**
     * Optimize database tables
     */
    private function optimizeDatabase(): array
    {
        $tables = Capsule::select("SHOW TABLES");

        $optimized = 0;
        foreach ($tables as $table) {
            $tableName = array_values((array)$table)[0];
            Capsule::statement("OPTIMIZE TABLE {$tableName}");
            $optimized++;
        }

        return ['tables_optimized' => $optimized];
    }

    /**
     * System health check
     */
    private function runHealthCheck(): array
    {
        $checks = [
            'disk_space' => $this->checkDiskSpace(),
            'database_connection' => $this->checkDatabaseConnection(),
            'cron_health' => $this->checkCronHealth(),
            'email_queue' => $this->checkEmailQueue(),
            'pending_operations' => $this->checkPendingOperations()
        ];

        $healthStatus = 'healthy';
        foreach ($checks as $check) {
            if (isset($check['status']) && $check['status'] !== 'ok') {
                $healthStatus = 'warning';
            }
        }

        return [
            'status' => $healthStatus,
            'checks' => $checks
        ];
    }

    private function checkDiskSpace(): array
    {
        $freeSpace = disk_free_space('/');
        $totalSpace = disk_total_space('/');
        $percentFree = ($freeSpace / $totalSpace) * 100;

        return [
            'free_space' => round($freeSpace / 1024 / 1024 / 1024, 2) . ' GB',
            'percent_free' => round($percentFree, 1),
            'status' => $percentFree > 20 ? 'ok' : 'warning'
        ];
    }

    private function checkDatabaseConnection(): array
    {
        try {
            Capsule::select('SELECT 1');
            return ['status' => 'ok', 'message' => 'Connected'];
        } catch (Exception $e) {
            return ['status' => 'error', 'message' => $e->getMessage()];
        }
    }

    private function checkCronHealth(): array
    {
        $lastCronRun = Capsule::table('tblactivitylog')
            ->where('description', 'like', '%cron%')
            ->orderBy('date', 'DESC')
            ->first();

        if (!$lastCronRun) {
            return ['status' => 'warning', 'message' => 'No cron activity found'];
        }

        $hoursSince = (time() - strtotime($lastCronRun->date)) / 3600;

        return [
            'last_run' => $lastCronRun->date,
            'hours_ago' => round($hoursSince, 1),
            'status' => $hoursSince < 1 ? 'ok' : 'warning'
        ];
    }

    private function checkEmailQueue(): array
    {
        $pendingEmails = Capsule::table('mod_email_queue')
            ->where('sent', 0)
            ->where('cancelled', 0)
            ->count();

        return [
            'pending' => $pendingEmails,
            'status' => $pendingEmails < 100 ? 'ok' : 'warning'
        ];
    }

    private function checkPendingOperations(): array
    {
        $pendingServices = Capsule::table('tblhosting')
            ->where('domainstatus', 'Pending')
            ->count();

        return [
            'pending_services' => $pendingServices,
            'status' => 'ok'
        ];
    }
}
```

### Step 2: Add Maintenance Hooks
```php
<?php
// /includes/hooks/maintenance_hooks.php

use WHMCS\Maintenance\MaintenanceManager;

add_hook('DailyCronJob', 1, function($vars) {
    $maintenance = new MaintenanceManager();
    $results = $maintenance->dailyMaintenance();

    // Alert on health issues
    if ($results['health_check']['status'] !== 'healthy') {
        sendAdminEmail('WHMCS Health Alert', $results);
    }

    return $results;
});

add_hook('WeeklyCronJob', 1, function($vars) {
    $maintenance = new MaintenanceManager();
    return $maintenance->weeklyMaintenance();
});
```

## Related Workflows
- [WHMCS Cleanup Automation](./whmcs-cleanup-automation.md)
- [WHMCS Backup Automation](./whmcs-backup-automation.md)