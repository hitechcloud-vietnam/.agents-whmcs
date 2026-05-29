# WHMCS Service Hooks

## Overview

Service hooks allow you to execute custom code during hosting service lifecycle events including creation, suspension, termination, and updates.

## Available Service Hooks

### Service Created Hook

```php
<?php
// Triggered when a new service/hosting account is created
add_hook('ServiceCreated', 1, function(array $vars) {
    $serviceId = $vars['service_id'];
    $userId = $vars['user_id'];
    $productId = $vars['pid'];
    $domain = $vars['domain'];
    
    // Initialize service configuration
    initializeServiceConfig($serviceId);
    
    // Setup monitoring
    setupServiceMonitoring($serviceId, $domain);
    
    // Configure backups
    configureServiceBackups($serviceId, $productId);
    
    // Set up SSL if applicable
    setupInitialSSL($serviceId, $domain);
    
    // Configure DNS
    configureServiceDNS($serviceId, $domain);
    
    return ['success' => true, 'service_id' => $serviceId];
});

function setupServiceMonitoring(int $serviceId, string $domain): void
{
    Capsule::table('mod_service_monitoring')->insert([
        'service_id' => $serviceId,
        'domain' => $domain,
        'check_interval' => 60,
        'last_check' => null,
        'status' => 'active',
        'created_at' => date('Y-m-d H:i:s'),
    ]);
}
```

### Service Updated Hook

```php
<?php
// Triggered when service details are updated
add_hook('ServiceUpdated', 1, function(array $vars) {
    $serviceId = $vars['service_id'];
    $changes = $vars['changes'] ?? [];
    
    // Log changes for audit
    logServiceChanges($serviceId, $changes);
    
    // Update server configuration
    if (isset($changes['bwlimit'])) {
        updateBandwidthLimit($serviceId, $changes['bwlimit']);
    }
    
    // Update resource allocation
    if (isset($changes['disklimit'])) {
        updateDiskLimit($serviceId, $changes['disklimit']);
    }
    
    // Update custom fields on server
    syncCustomFieldsToServer($serviceId, $changes);
    
    return ['success' => true];
});

function logServiceChanges(int $serviceId, array $changes): void
{
    Capsule::table('mod_service_change_log')->insert([
        'service_id' => $serviceId,
        'changes' => json_encode($changes),
        'changed_by' => $_SESSION['adminid'] ?? 0,
        'ip_address' => $_SERVER['REMOTE_ADDR'] ?? '',
        'created_at' => date('Y-m-d H:i:s'),
    ]);
}
```

### Service Suspended Hook

```php
<?php
// Triggered when a service is suspended
add_hook('ServiceSuspended', 1, function(array $vars) {
    $serviceId = $vars['service_id'];
    $suspendReason = $vars['suspend_reason'] ?? 'Payment overdue';
    
    // Log suspension
    logServiceSuspension($serviceId, $suspendReason);
    
    // Disable monitoring alerts
    disableServiceMonitoring($serviceId);
    
    // Update DNS to suspended page
    redirectToSuspendedPage($serviceId);
    
    // Send suspension notification
    sendSuspensionNotification($serviceId, $suspendReason);
    
    // Schedule automatic unsuspend check
    scheduleUnsuspendCheck($serviceId);
    
    return ['success' => true];
});

function logServiceSuspension(int $serviceId, string $reason): void
{
    Capsule::table('mod_service_audit')->insert([
        'service_id' => $serviceId,
        'action' => 'suspended',
        'reason' => $reason,
        'suspended_at' => date('Y-m-d H:i:s'),
    ]);
}
```

### Service Unsuspended Hook

```php
<?php
// Triggered when a service is unsuspended
add_hook('ServiceUnsuspended', 1, function(array $vars) {
    $serviceId = $vars['service_id'];
    
    // Re-enable monitoring
    enableServiceMonitoring($serviceId);
    
    // Restore DNS configuration
    restoreDNSConfiguration($serviceId);
    
    // Resume automated tasks
    resumeServiceTasks($serviceId);
    
    // Send unsuspension notification
    sendUnsuspensionNotification($serviceId);
    
    // Record unsuspension for billing
    recordUnsuspension($serviceId);
    
    return ['success' => true];
});
```

### Service Terminated Hook

```php
<?php
// Triggered when a service is terminated
add_hook('ServiceTerminated', 1, function(array $vars) {
    $serviceId = $vars['service_id'];
    $terminateReason = $vars['terminate_reason'] ?? 'Manual termination';
    
    // Backup data before termination
    $backupResult = createTerminationBackup($serviceId);
    
    // Send termination notice
    sendTerminationNotification($serviceId, $terminateReason);
    
    // Archive service data
    archiveServiceData($serviceId);
    
    // Clean up monitoring
    removeServiceMonitoring($serviceId);
    
    // Cancel scheduled tasks
    cancelScheduledTasks($serviceId);
    
    // Release resources
    releaseServiceResources($serviceId);
    
    return [
        'success' => true,
        'backup_created' => $backupResult['success'] ?? false,
    ];
});

function createTerminationBackup(int $serviceId): array
{
    $service = Capsule::table('tblhosting')->where('id', $serviceId)->first();
    $backupPath = "/backups/services/{$serviceId}/termination_" . date('Ymd');
    
    return [
        'success' => true,
        'backup_path' => $backupPath,
    ];
}
```

### Service Renewed Hook

```php
<?php
// Triggered when a service is renewed
add_hook('ServiceRenewed', 1, function(array $vars) {
    $serviceId = $vars['service_id'];
    $newDueDate = $vars['nextduedate'];
    $billingCycle = $vars['billingcycle'];
    
    // Update service expiration
    updateServiceExpiration($serviceId, $newDueDate);
    
    // Extend SSL certificates
    extendSSLCertificates($serviceId);
    
    // Reset usage counters
    resetUsageCounters($serviceId);
    
    // Update monitoring
    extendMonitoringSubscription($serviceId);
    
    // Send renewal confirmation
    sendRenewalConfirmation($serviceId);
    
    return ['success' => true];
});
```

## Comprehensive Service Handler

```php
<?php
class ServiceHookHandler {
    
    public function register(): void
    {
        add_hook('ServiceCreated', 1, [$this, 'handleServiceCreated']);
        add_hook('ServiceUpdated', 1, [$this, 'handleServiceUpdated']);
        add_hook('ServiceSuspended', 1, [$this, 'handleSuspended']);
        add_hook('ServiceUnsuspended', 1, [$this, 'handleUnsuspended']);
        add_hook('ServiceTerminated', 1, [$this, 'handleTerminated']);
        add_hook('ServiceRenewed', 1, [$this, 'handleRenewed']);
    }
    
    public function handleServiceCreated(array $vars): array
    {
        $this->initializeService($vars['service_id']);
        $this->setupMonitoring($vars['service_id']);
        $this->scheduleTasks($vars['service_id']);
        return ['success' => true];
    }
    
    public function handleServiceUpdated(array $vars): array
    {
        $this->logChanges($vars['service_id'], $vars['changes'] ?? []);
        $this->syncToServer($vars['service_id'], $vars['changes'] ?? []);
        return ['success' => true];
    }
    
    public function handleSuspended(array $vars): array
    {
        $this->logSuspension($vars['service_id'], $vars['suspend_reason'] ?? '');
        $this->disableMonitoring($vars['service_id']);
        $this->notifySuspension($vars['service_id']);
        return ['success' => true];
    }
    
    public function handleUnsuspended(array $vars): array
    {
        $this->logUnsuspension($vars['service_id']);
        $this->enableMonitoring($vars['service_id']);
        $this->notifyUnsuspension($vars['service_id']);
        return ['success' => true];
    }
    
    public function handleTerminated(array $vars): array
    {
        $this->createBackup($vars['service_id']);
        $this->archiveData($vars['service_id']);
        $this->notifyTermination($vars['service_id']);
        return ['success' => true];
    }
    
    public function handleRenewed(array $vars): array
    {
        $this->updateExpiration($vars['service_id'], $vars['nextduedate']);
        $this->resetUsage($vars['service_id']);
        $this->notifyRenewal($vars['service_id']);
        return ['success' => true];
    }
    
    private function initializeService(int $serviceId): void
    {
        Capsule::table('mod_service_meta')->insert([
            'service_id' => $serviceId,
            'created_at' => date('Y-m-d H:i:s'),
        ]);
    }
    
    private function setupMonitoring(int $serviceId): void
    {
        Capsule::table('mod_monitoring')->insert([
            'service_id' => $serviceId,
            'enabled' => true,
        ]);
    }
    
    private function scheduleTasks(int $serviceId): void
    {
        // Schedule maintenance tasks
    }
    
    private function logChanges(int $serviceId, array $changes): void
    {
        Capsule::table('mod_service_audit')->insert([
            'service_id' => $serviceId,
            'action' => 'updated',
            'changes' => json_encode($changes),
            'created_at' => date('Y-m-d H:i:s'),
        ]);
    }
    
    private function syncToServer(int $serviceId, array $changes): void
    {
        // Sync changes to server
    }
    
    private function logSuspension(int $serviceId, string $reason): void
    {
        Capsule::table('mod_service_audit')->insert([
            'service_id' => $serviceId,
            'action' => 'suspended',
            'reason' => $reason,
            'created_at' => date('Y-m-d H:i:s'),
        ]);
    }
    
    private function logUnsuspension(int $serviceId): void
    {
        Capsule::table('mod_service_audit')->insert([
            'service_id' => $serviceId,
            'action' => 'unsuspended',
            'created_at' => date('Y-m-d H:i:s'),
        ]);
    }
    
    private function disableMonitoring(int $serviceId): void
    {
        Capsule::table('mod_monitoring')
            ->where('service_id', $serviceId)
            ->update(['enabled' => false]);
    }
    
    private function enableMonitoring(int $serviceId): void
    {
        Capsule::table('mod_monitoring')
            ->where('service_id', $serviceId)
            ->update(['enabled' => true]);
    }
    
    private function notifySuspension(int $serviceId): void
    {
        // Send suspension notification
    }
    
    private function notifyUnsuspension(int $serviceId): void
    {
        // Send unsuspension notification
    }
    
    private function createBackup(int $serviceId): void
    {
        // Create final backup
    }
    
    private function archiveData(int $serviceId): void
    {
        // Archive service data
    }
    
    private function notifyTermination(int $serviceId): void
    {
        // Send termination notification
    }
    
    private function updateExpiration(int $serviceId, string $newDate): void
    {
        Capsule::table('tblhosting')
            ->where('id', $serviceId)
            ->update(['nextduedate' => $newDate]);
    }
    
    private function resetUsage(int $serviceId): void
    {
        Capsule::table('mod_service_usage')
            ->where('service_id', $serviceId)
            ->update(['reset_at' => date('Y-m-d H:i:s')]);
    }
    
    private function notifyRenewal(int $serviceId): void
    {
        // Send renewal notification
    }
}

$handler = new ServiceHookHandler();
$handler->register();
```

## Best Practices

1. **Handle provisioning asynchronously** - Don't block on external calls
2. **Maintain audit logs** - Track all service state changes
3. **Use transactions** - Ensure data consistency
4. **Implement rollback** - Be able to undo changes if needed
5. **Queue notifications** - Send emails in background

## Related Documentation

- [WHMCS Order Hooks](/docs/whmcs-order-hooks.md)
- [WHMCS Provisioning Module Development](/docs/whmcs-provisioning-dev.md)