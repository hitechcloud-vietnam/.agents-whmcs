# WHMCS Module Uninstallation

Complete guide for safe module uninstallation.

## Overview

Proper uninstallation removes module data while preserving critical information.

## Uninstall Process

### Basic Uninstallation

```php
<?php
/**
 * Module deactivation/uninstallation
 */
function yourmodule_deactivate(): array
{
    try {
        // Check for active services
        $activeCount = Capsule::table('mod_yourmodule_data')
            ->where('status', 'active')
            ->count();
        
        if ($activeCount > 0) {
            return [
                'status' => 'error',
                'description' => "Cannot deactivate: {$activeCount} active services exist",
            ];
        }
        
        // Remove hooks
        remove_hook('ClientAreaPrimaryNavbar', 'yourmodule_clientNavbarHook');
        remove_hook('ServiceProvision', 'yourmodule_serviceProvisionHook');
        
        // Clear cache
        $cache = \WHMCS\File\Cache::factory('YourModule');
        $cache->deleteAll();
        
        return [
            'status' => 'success',
            'description' => 'Module deactivated. Data preserved for potential reactivation.',
        ];
        
    } catch (Exception $e) {
        return [
            'status' => 'error',
            'description' => 'Deactivation failed: ' . $e->getMessage(),
        ];
    }
}

/**
 * Full uninstallation (called separately)
 */
function yourmodule_uninstall(bool $force = false): array
{
    try {
        // Final check for active services
        if (!$force) {
            $activeCount = Capsule::table('mod_yourmodule_data')
                ->whereIn('status', ['active', 'suspended'])
                ->count();
            
            if ($activeCount > 0) {
                return [
                    'status' => 'error',
                    'description' => "Cannot uninstall: {$activeCount} services still active",
                ];
            }
        }
        
        // Export data before deletion
        exportModuleData();
        
        // Remove database tables
        Capsule::schema()->dropIfExists('mod_yourmodule_data');
        Capsule::schema()->dropIfExists('mod_yourmodule_logs');
        Capsule::schema()->dropIfExists('mod_yourmodule_settings');
        
        // Remove configuration
        Capsule::table('tblconfiguration')
            ->where('setting', 'LIKE', 'YourModule%')
            ->delete();
        
        // Remove from modules table
        Capsule::table('tblmodules')
            ->where('type', 'servers')
            ->where('name', 'yourmodule')
            ->delete();
        
        logActivity('YourModule: Module uninstalled successfully');
        
        return [
            'status' => 'success',
            'description' => 'Module uninstalled. Data exported and removed.',
        ];
        
    } catch (Exception $e) {
        return [
            'status' => 'error',
            'description' => 'Uninstallation failed: ' . $e->getMessage(),
        ];
    }
}
```

## Data Export Before Deletion

### Export Service Data

```php
<?php
/**
 * Export all module data before deletion
 */
function exportModuleData(): bool
{
    try {
        $exportDir = __DIR__ . '/../backups';
        
        if (!is_dir($exportDir)) {
            mkdir($exportDir, 0755, true);
        }
        
        $timestamp = date('Y-m-d_H-i-s');
        $filename = "yourmodule_export_{$timestamp}.json";
        $filepath = "{$exportDir}/{$filename}";
        
        $data = [
            'export_date' => date('Y-m-d H:i:s'),
            'module_version' => yourmodule_getVersion(),
            'services' => [],
            'settings' => [],
        ];
        
        // Export service data
        $services = Capsule::table('mod_yourmodule_data')->get();
        foreach ($services as $service) {
            $data['services'][] = (array) $service;
        }
        
        // Export settings
        $settings = Capsule::table('tblconfiguration')
            ->where('setting', 'LIKE', 'YourModule%')
            ->get();
        foreach ($settings as $setting) {
            $data['settings'][] = (array) $setting;
        }
        
        // Write export file
        file_put_contents(
            $filepath,
            json_encode($data, JSON_PRETTY_PRINT)
        );
        
        logActivity("YourModule: Data exported to {$filepath}");
        
        return true;
        
    } catch (Exception $e) {
        logActivity("YourModule: Export failed - " . $e->getMessage());
        return false;
    }
}
```

### Export Logs

```php
<?php
/**
 * Export operation logs
 */
function exportModuleLogs(string $filepath): bool
{
    try {
        $logs = Capsule::table('mod_yourmodule_logs')
            ->orderBy('created_at', 'desc')
            ->limit(10000)
            ->get();
        
        $exportData = [
            'export_date' => date('Y-m-d H:i:s'),
            'total_records' => count($logs),
            'logs' => [],
        ];
        
        foreach ($logs as $log) {
            $exportData['logs'][] = (array) $log;
        }
        
        file_put_contents(
            $filepath,
            json_encode($exportData, JSON_PRETTY_PRINT)
        );
        
        return true;
        
    } catch (Exception $e) {
        return false;
    }
}
```

## Service Cleanup

### Mark Services for Cleanup

```php
<?php
/**
 * Mark services for cleanup without immediate termination
 */
function yourmodule_markForCleanup(): array
{
    try {
        $affectedRows = Capsule::table('mod_yourmodule_data')
            ->whereIn('status', ['active', 'suspended'])
            ->update([
                'status' => 'pending_cleanup',
                'cleanup_scheduled_at' => date('Y-m-d H:i:s'),
            ]);
        
        return [
            'success' => true,
            'affected_services' => $affectedRows,
        ];
        
    } catch (Exception $e) {
        return [
            'success' => false,
            'error' => $e->getMessage(),
        ];
    }
}

/**
 * Get services pending cleanup
 */
function getServicesPendingCleanup(): array
{
    return Capsule::table('mod_yourmodule_data')
        ->where('status', 'pending_cleanup')
        ->get();
}
```

### Bulk Service Termination

```php
<?php
/**
 * Terminate all remaining services
 */
function yourmodule_terminateAllServices(bool $force = false): array
{
    $results = [
        'terminated' => 0,
        'failed' => 0,
        'errors' => [],
    ];
    
    $services = Capsule::table('mod_yourmodule_data')
        ->whereIn('status', ['active', 'suspended', 'pending_cleanup'])
        ->get();
    
    foreach ($services as $service) {
        try {
            $params = [
                'serviceid' => $service->service_id,
                'account_id' => $service->account_id,
            ];
            
            $result = yourmodule_externalTerminate($params);
            
            if ($result['success']) {
                Capsule::table('mod_yourmodule_data')
                    ->where('id', $service->id)
                    ->update(['status' => 'terminated']);
                
                $results['terminated']++;
            } else {
                $results['failed']++;
                $results['errors'][] = $result['error'];
            }
            
        } catch (Exception $e) {
            $results['failed']++;
            $results['errors'][] = "Service {$service->service_id}: " . $e->getMessage();
        }
    }
    
    return $results;
}

/**
 * Call external API for termination
 */
function yourmodule_externalTerminate(array $params): array
{
    $api = new YourModuleAPI(getModuleConfig());
    return $api->terminateAccount($params['account_id']);
}
```

## Cleanup Operations

### File Cleanup

```php
<?php
/**
 * Remove module files
 */
function yourmodule_cleanupFiles(): array
{
    $paths = [
        __DIR__ . '/../cache',
        __DIR__ . '/../logs',
        __DIR__ . '/../temp',
    ];
    
    $cleaned = 0;
    
    foreach ($paths as $path) {
        if (is_dir($path)) {
            $files = glob("{$path}/*");
            foreach ($files as $file) {
                if (is_file($file)) {
                    unlink($file);
                    $cleaned++;
                }
            }
            
            // Remove empty directories
            if (count(glob("{$path}/*")) === 0) {
                rmdir($path);
            }
        }
    }
    
    return [
        'success' => true,
        'files_cleaned' => $cleaned,
    ];
}
```

### Cache Cleanup

```php
<?php
/**
 * Clear all module caches
 */
function yourmodule_clearCache(): void
{
    // WHMCS cache
    \WHMCS\File\Cache::factory('YourModule')->deleteAll();
    
    // OPCache
    if (function_exists('opcache_get_status')) {
        opcache_get_status();
    }
    
    // Application cache
    $cacheDir = \WHMCS\Utility\Di::getInstance()->getApp()->getApplication()->getCachedir();
    $moduleCache = $cacheDir . '/module_yourmodule*';
    
    foreach (glob($moduleCache) as $file) {
        if (is_file($file)) {
            unlink($file);
        }
    }
}
```

## Safe Uninstall Checklist

```php
<?php
/**
 * Pre-uninstall validation
 */
function yourmodule_preUninstallCheck(): array
{
    $checks = [];
    
    // Check active services
    $activeCount = Capsule::table('mod_yourmodule_data')
        ->whereIn('status', ['active', 'suspended'])
        ->count();
    
    $checks['active_services'] = [
        'status' => $activeCount === 0 ? 'pass' : 'warn',
        'message' => $activeCount === 0 
            ? 'No active services' 
            : "{$activeCount} active services will be affected",
    ];
    
    // Check pending invoices
    $pendingInvoices = Capsule::table('tblinvoices')
        ->join('tblinvoiceitems', 'tblinvoices.id', '=', 'tblinvoiceitems.invoiceid')
        ->join('tblhosting', 'tblinvoiceitems.relid', '=', 'tblhosting.id')
        ->join('tblproducts', 'tblhosting.packageid', '=', 'tblproducts.id')
        ->where('tblproducts.servertype', 'yourmodule')
        ->whereIn('tblinvoices.status', ['Unpaid', 'Partial'])
        ->count();
    
    $checks['pending_invoices'] = [
        'status' => $pendingInvoices === 0 ? 'pass' : 'warn',
        'message' => $pendingInvoices === 0 
            ? 'No pending invoices' 
            : "{$pendingInvoices} pending invoices exist",
    ];
    
    // Check backup capability
    $checks['backup_created'] = [
        'status' => 'info',
        'message' => 'Export data before uninstalling',
    ];
    
    return $checks;
}
```

## Force Uninstall

```php
<?php
/**
 * Force uninstallation (admin override)
 */
function yourmodule_forceUninstall(): array
{
    try {
        logActivity('YourModule: Force uninstall initiated');
        
        // Terminate all services immediately
        $terminateResult = yourmodule_terminateAllServices(true);
        
        // Export remaining data
        exportModuleData();
        
        // Drop all tables
        Capsule::schema()->dropIfExists('mod_yourmodule_data');
        Capsule::schema()->dropIfExists('mod_yourmodule_logs');
        Capsule::schema()->dropIfExists('mod_yourmodule_settings');
        
        // Remove configuration
        Capsule::table('tblconfiguration')
            ->where('setting', 'LIKE', 'YourModule%')
            ->delete();
        
        // Clear caches
        yourmodule_clearCache();
        
        // Remove files
        yourmodule_cleanupFiles();
        
        logActivity('YourModule: Force uninstall completed');
        
        return [
            'status' => 'success',
            'description' => 'Module forcefully uninstalled',
            'details' => $terminateResult,
        ];
        
    } catch (Exception $e) {
        return [
            'status' => 'error',
            'description' => 'Force uninstall failed: ' . $e->getMessage(),
        ];
    }
}
```

## Best Practices

1. **Warn about data loss** - Alert admin about data removal
2. **Export first** - Always backup before uninstalling
3. **Terminate services gracefully** - Allow external API cleanup
4. **Check dependencies** - Verify no dependent services
5. **Clean up thoroughly** - Remove all traces of module
6. **Log everything** - Record uninstall actions

## Related Documentation

- [whmcs-module-lifecycle.md](whmcs-module-lifecycle.md)
- [whmcs-module-upgrade.md](whmcs-module-upgrade.md)
