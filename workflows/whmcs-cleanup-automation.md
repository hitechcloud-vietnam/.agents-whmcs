# WHMCS Scheduled Cleanup Workflow

## Overview
This workflow implements automated cleanup tasks to maintain WHMCS performance and storage.

## Prerequisites
- WHMCS with cron access
- Admin access for cleanup operations

## Step-by-Step Process

### Step 1: Create Cleanup Manager
```php
<?php
// /includes/cleanup/CleanupManager.php

namespace WHMCS\Cleanup;

class CleanupManager
{
    private $results = [];

    /**
     * Run all cleanup tasks
     */
    public function runCleanup(): array
    {
        $this->cleanupActivityLogs();
        $this->cleanupTempFiles();
        $this->cleanupCache();
        $this->cleanupOldBackups();
        $this->cleanupFailedOrders();
        $this->cleanupStaleSessions();
        $this->cleanupOrphanedRecords();

        return $this->results;
    }

    /**
     * Cleanup activity logs
     */
    private function cleanupActivityLogs(): void
    {
        $retentionDays = getConfig('activity_log_retention_days') ?? 90;

        $deleted = Capsule::table('tblactivitylog')
            ->where('date', '<', date('Y-m-d', strtotime("-{$retentionDays} days")))
            ->delete();

        $this->results['activity_logs'] = $deleted;
    }

    /**
     * Cleanup temporary files
     */
    private function cleanupTempFiles(): void
    {
        $tempDirs = [
            __DIR__ . '/../../temp',
            __DIR__ . '/../../cache/templates_c'
        ];

        $totalDeleted = 0;

        foreach ($tempDirs as $dir) {
            if (is_dir($dir)) {
                $files = glob($dir . '/*');
                foreach ($files as $file) {
                    if (is_file($file) && filemtime($file) < strtotime('-7 days')) {
                        unlink($file);
                        $totalDeleted++;
                    }
                }
            }
        }

        $this->results['temp_files'] = $totalDeleted;
    }

    /**
     * Cleanup cache
     */
    private function cleanupCache(): void
    {
        $cacheDir = __DIR__ . '/../../cache';

        // Clear opcache if available
        if (function_exists('opcache_get_status')) {
            opcache_reset();
        }

        // Clear file cache
        $deleted = 0;
        foreach (glob($cacheDir . '/*') as $file) {
            if (is_file($file)) {
                unlink($file);
                $deleted++;
            }
        }

        $this->results['cache_files'] = $deleted;
    }

    /**
     * Cleanup failed/abandoned orders
     */
    private function cleanupFailedOrders(): void
    {
        $retentionDays = getConfig('failed_order_retention_days') ?? 30;

        $deleted = Capsule::table('tblorders')
            ->where('status', 'Cancelled')
            ->where('date', '<', date('Y-m-d', strtotime("-{$retentionDays} days")))
            ->delete();

        $this->results['failed_orders'] = $deleted;
    }

    /**
     * Cleanup stale sessions
     */
    private function cleanupStaleSessions(): void
    {
        $sessionTimeout = getConfig('session_timeout') ?? 3600;

        $deleted = Capsule::table('tblsessions')
            ->where('last_updated', '<', date('Y-m-d H:i:s', time() - $sessionTimeout))
            ->delete();

        $this->results['stale_sessions'] = $deleted;
    }

    /**
     * Cleanup orphaned records
     */
    private function cleanupOrphanedRecords(): void
    {
        // Orphaned invoice items
        $orphanedItems = Capsule::table('tblinvoiceitems')
            ->whereNotExists(function($q) {
                $q->select(Capsule::raw(1))
                    ->from('tblinvoices')
                    ->whereColumn('tblinvoices.id', 'tblinvoiceitems.invoiceid');
            })
            ->delete();

        // Orphaned service addons
        $orphanedAddons = Capsule::table('tblhostingaddons')
            ->whereNotExists(function($q) {
                $q->select(Capsule::raw(1))
                    ->from('tblhosting')
                    ->whereColumn('tblhosting.id', 'tblhostingaddons.hostingid');
            })
            ->delete();

        $this->results['orphaned_records'] = $orphanedItems + $orphanedAddons;
    }
}
```

### Step 2: Add Cleanup Hooks
```php
<?php
// /includes/hooks/cleanup_hooks.php

use WHMCS\Cleanup\CleanupManager;

add_hook('DailyCronJob', 1, function($vars) {
    $cleanupManager = new CleanupManager();
    $results = $cleanupManager->runCleanup();

    logActivity("Cleanup completed: " . json_encode($results));

    return $results;
});
```

## Related Workflows
- [WHMCS Maintenance Automation](./whmcs-maintenance-automation.md)
- [WHMCS Backup Automation](./whmcs-backup-automation.md)