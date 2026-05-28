# WHMCS Service Lifecycle Hooks

**Version:** 8.x | **Updated:** 2026-05-29
**Related Skills:** `whmcs-hooks-development`, `whmcs-lifecycle-management`, `whmcs-service-billing`

---

## Overview

Service lifecycle hooks allow you to automate actions during the lifecycle of hosting services. This includes provisioning, suspension, termination, upgrades, and custom service-related events.

---

## Service Hooks Overview

### Available Service Hooks

| Hook Name | Description | Parameters |
|-----------|-------------|------------|
| `AfterModuleCreate` | Service provisioned | `serviceid`, `userid` |
| `AfterModuleSuspend` | Service suspended | `serviceid`, `userid` |
| `AfterModuleUnsuspend` | Service reactivated | `serviceid`, `userid` |
| `AfterModuleTerminate` | Service terminated | `serviceid`, `userid` |
| `AfterModuleChangePackage` | Package upgraded/downgraded | `serviceid`, `params` |
| `AfterModuleChangePassword` | Password changed | `serviceid`, `newpassword` |
| `PreServiceDelete` | Before termination | `userid`, `serviceid` |
| `ServiceEdit` | Service details edited | `serviceid`, `params` |
| `ServiceView` | Service details viewed | `serviceid` |

---

## Provisioning Hooks

### After Service Creation

```php
<?php
/**
 * AfterModuleCreate hook
 * Execute after successful service provisioning
 */

add_hook('AfterModuleCreate', 1, function($vars) {
    $serviceId = $vars['serviceid'];
    $userId = $vars['userid'];

    // Get service details
    $service = Capsule::table('tblhosting')
        ->where('id', $serviceId)
        ->first();

    $client = Capsule::table('tblclients')
        ->where('id', $userId)
        ->first();

    // Example 1: Send welcome email with server details
    send_email([
        'type' => 'product',
        'id' => $service->packageid,
        'customvars' => [
            'service_id' => $serviceId,
            'server_ip' => $service->dedicatedip,
            'server_username' => $service->username,
            'control_panel_url' => getControlPanelUrl($service),
        ],
    ], $userId);

    // Example 2: Create DNS records
    $product = Capsule::table('tblproducts')
        ->where('id', $service->packageid)
        ->first();

    if ($product->autosetup == 'on') {
        createDnsRecords($service->domain, $service->dedicatedip);
    }

    // Example 3: Sync to external CRM
    $crmData = [
        'service_id' => $serviceId,
        'client_email' => $client->email,
        'product_name' => $product->name,
        'status' => 'active',
        'next_due' => $service->nextduedate,
    ];
    syncToCrm($crmData);

    // Example 4: Create billing record
    Capsule::table('mod_service_provisioning_log')->insert([
        'service_id' => $serviceId,
        'action' => 'create',
        'timestamp' => date('Y-m-d H:i:s'),
        'details' => json_encode([
            'server_ip' => $service->dedicatedip,
            'username' => $service->username,
        ]),
    ]);
});

/**
 * Get control panel URL based on product type
 */
function getControlPanelUrl($service): string
{
    $product = Capsule::table('tblproducts')
        ->where('id', $service->packageid)
        ->first();

    $server = Capsule::table('tblservers')
        ->where('id', $service->server)
        ->first();

    return sprintf(
        'https://%s:%s@%s:%d',
        $server->username,
        $server->password,
        $server->hostname,
        $server->secureport ?: 2087
    );
}
```

### Pre-Provisioning Validation

```php
<?php
/**
 * Custom pre-provisioning checks
 */

add_hook('OrderProductValidation', 1, function($vars) {
    $productId = $vars['pid'];
    $userId = $_SESSION['uid'];

    // Check for existing similar services
    $existingService = Capsule::table('tblhosting')
        ->where('userid', $userId)
        ->where('domainstatus', 'Active')
        ->where('packageid', $productId)
        ->first();

    if ($existingService) {
        return [
            'error' => 'You already have an active service with this product.',
            'existing_service_id' => $existingService->id,
        ];
    }

    // Check resource limits
    $clientServices = Capsule::table('tblhosting')
        ->where('userid', $userId)
        ->where('domainstatus', 'Active')
        ->count();

    $maxServices = Capsule::table('tblclients')
        ->where('id', $userId)
        ->value('max_services');

    if ($maxServices && $clientServices >= $maxServices) {
        return [
            'error' => 'You have reached your maximum number of services.',
            'upgrade_url' => 'upgrade.php',
        ];
    }

    return $vars;
});
```

---

## Suspension Hooks

### After Service Suspension

```php
<?php
/**
 * AfterModuleSuspend hook
 * Execute after service suspension
 */

add_hook('AfterModuleSuspend', 1, function($vars) {
    $serviceId = $vars['serviceid'];
    $userId = $vars['userid'];

    // Get service details
    $service = Capsule::table('tblhosting')
        ->where('id', $serviceId)
        ->first();

    $client = Capsule::table('tblclients')
        ->where('id', $userId)
        ->first();

    // Example 1: Update DNS to suspended landing page
    updateDnsSuspendedPage($service->domain);

    // Example 2: Notify client
    send_email([
        'type' => 'product',
        'id' => $service->packageid,
        'customvars' => [
            'service_id' => $serviceId,
            'suspension_reason' => getSuspensionReason($serviceId),
            'reactivate_url' => getReactivateUrl($serviceId),
        ],
    ], $userId);

    // Example 3: Update external systems
    updateExternalService($serviceId, 'suspended');

    // Example 4: Log suspension
    Capsule::table('mod_service_audit')->insert([
        'service_id' => $serviceId,
        'action' => 'suspend',
        'performed_by' => $_SESSION['adminid'] ?? 'system',
        'timestamp' => date('Y-m-d H:i:s'),
        'reason' => getSuspensionReason($serviceId),
    ]);

    // Example 5: Release resources (if applicable)
    releaseLoadBalancerNode($serviceId);
    freeBackupSlots($serviceId);
});

/**
 * Get suspension reason from WHMCS
 */
function getSuspensionReason(int $serviceId): string
{
    $log = Capsule::table('tblactivitylog')
        ->where('description', 'like', '%Suspend%')
        ->where('description', 'like', '%' . $serviceId . '%')
        ->orderBy('id', 'desc')
        ->first();

    if ($log) {
        return $log->description;
    }

    return 'Service suspended due to non-payment';
}
```

### Pre-Suspension Hooks

```php
<?php
/**
 * Custom pre-suspension validation
 */

add_hook('PreServiceSuspend', 1, function($vars) {
    $serviceId = $vars['serviceid'];

    // Check for pending orders
    $pendingOrders = Capsule::table('tblorders')
        ->where('userid', $vars['userid'])
        ->whereIn('status', ['Pending', 'Active'])
        ->where('id', '!=', $serviceId)
        ->count();

    if ($pendingOrders > 0) {
        return [
            'error' => 'Cannot suspend: Customer has pending orders.',
        ];
    }

    // Check for active SLA
    $hasSla = Capsule::table('mod_service_sla')
        ->where('service_id', $serviceId)
        ->where('status', 'active')
        ->where('immunity_until', '>', date('Y-m-d H:i:s'))
        ->exists();

    if ($hasSla) {
        return [
            'error' => 'Cannot suspend: Service has active SLA immunity.',
            'immunity_expires' => getSlaImmunityExpiry($serviceId),
        ];
    }

    return $vars;
});
```

---

## Termination Hooks

### Before Service Deletion

```php
<?php
/**
 * PreServiceDelete hook
 * Execute before service termination
 */

add_hook('PreServiceDelete', 1, function($vars) {
    $serviceId = $vars['serviceid'];

    // Example 1: Create final backup
    createServiceBackup($serviceId, 'final');

    // Example 2: Export service data
    $exportPath = exportServiceData($serviceId);
    saveBackupLocation($serviceId, $exportPath);

    // Example 3: Send final data export email
    $service = Capsule::table('tblhosting')
        ->where('id', $serviceId)
        ->first();

    send_email([
        'type' => 'product',
        'id' => $service->packageid,
        'customvars' => [
            'data_export_url' => $exportPath,
            'export_expires' => date('Y-m-d H:i:s', strtotime('+7 days')),
        ],
    ], $vars['userid']);

    // Example 4: Cancel scheduled tasks
    cancelScheduledTasks($serviceId);

    // Example 5: Release domain (if applicable)
    if ($service->domain) {
        markDomainForDeletion($service->domain);
    }

    return $vars; // Return to continue with termination
});

/**
 * Allow force termination (skip confirmation)
 */
add_hook('PreServiceDelete', 1, function($vars) {
    // Check if admin is forcing termination
    if ($_SESSION['adminid'] && $_GET['force'] === '1') {
        $admin = Capsule::table('tbladmins')
            ->where('id', $_SESSION['adminid'])
            ->first();

        if ($admin->roleid == 1) { // Full admin
            return [
                'skip_confirmation' => true,
            ];
        }
    }

    return $vars;
});
```

### After Termination

```php
<?php
/**
 * AfterModuleTerminate hook
 * Execute after service termination
 */

add_hook('AfterModuleTerminate', 1, function($vars) {
    $serviceId = $vars['serviceid'];

    // Example 1: Update external systems
    updateExternalService($serviceId, 'terminated');

    // Example 2: Archive service data
    archiveServiceData($serviceId);

    // Example 3: Release SSL certificates
    revokeSslCertificates($serviceId);

    // Example 4: Update monitoring systems
    removeFromMonitoring($serviceId);

    // Example 5: Log termination
    Capsule::table('mod_service_audit')->insert([
        'service_id' => $serviceId,
        'action' => 'terminate',
        'performed_by' => $_SESSION['adminid'] ?? 'system',
        'timestamp' => date('Y-m-d H:i:s'),
        'data' => json_encode($_POST),
    ]);

    // Example 6: Remove DNS records (after grace period)
    scheduleDnsCleanup($serviceId, '+7 days');
});
```

---

## Upgrade/Downgrade Hooks

### After Package Change

```php
<?php
/**
 * AfterModuleChangePackage hook
 * Execute after service upgrade/downgrade
 */

add_hook('AfterModuleChangePackage', 1, function($vars) {
    $serviceId = $vars['serviceid'];
    $params = $vars['params'];

    // Get updated service
    $service = Capsule::table('tblhosting')
        ->where('id', $serviceId)
        ->first();

    $newProduct = Capsule::table('tblproducts')
        ->where('id', $service->packageid)
        ->first();

    // Example 1: Apply new resource limits
    applyNewResourceLimits($serviceId, $newProduct);

    // Example 2: Resize server resources
    resizeServerResources($serviceId, [
        'cpu' => $newProduct->configoption1,
        'ram' => $newProduct->configoption2,
        'disk' => $newProduct->configoption3,
    ]);

    // Example 3: Send upgrade confirmation
    send_email([
        'type' => 'product',
        'id' => $service->packageid,
        'customvars' => [
            'new_plan' => $newProduct->name,
            'new_price' => $service->amount,
        ],
    ], $service->userid);

    // Example 4: Update billing (prorate)
    calculateProration($serviceId, $params);

    // Example 5: Log the change
    Capsule::table('mod_service_changes')->insert([
        'service_id' => $serviceId,
        'change_type' => 'package_change',
        'old_product_id' => $params['old_pid'],
        'new_product_id' => $params['new_pid'],
        'timestamp' => date('Y-m-d H:i:s'),
        'admin_id' => $_SESSION['adminid'] ?? null,
    ]);
});

/**
 * Prorate billing calculation
 */
function calculateProration(int $serviceId, array $params): void
{
    $service = Capsule::table('tblhosting')->where('id', $serviceId)->first();

    $oldPrice = $params['old_recurring'] ?? 0;
    $newPrice = $service->amount;
    $billingCycle = $params['billing_cycle'] ?? 'monthly';

    // Calculate prorated amounts
    $daysInCycle = 30; // Simplified
    $daysRemaining = daysRemainingInCycle($serviceId);

    $oldProrate = ($oldPrice / $daysInCycle) * $daysRemaining;
    $newProrate = ($newPrice / $daysInCycle) * $daysRemaining;

    $difference = $newProrate - $oldProrate;

    if ($difference != 0) {
        // Apply credit or charge
        if ($difference > 0) {
            addInvoiceItem($serviceId, 'Upgrade Proration', $difference);
        } else {
            addCredit($service->userid, abs($difference), 'Downgrade credit');
        }
    }
}
```

---

## Password Change Hooks

### After Password Change

```php
<?php
/**
 * AfterModuleChangePassword hook
 * Execute after service password change
 */

add_hook('AfterModuleChangePassword', 1, function($vars) {
    $serviceId = $vars['serviceid'];
    $newPassword = $vars['newpassword'];

    // Example 1: Log password change (without password)
    Capsule::table('mod_service_audit')->insert([
        'service_id' => $serviceId,
        'action' => 'password_change',
        'timestamp' => date('Y-m-d H:i:s'),
        'admin_id' => $_SESSION['adminid'] ?? null,
        'ip_address' => $_SERVER['REMOTE_ADDR'],
    ]);

    // Example 2: Update external systems
    updateRemotePassword($serviceId, $newPassword);

    // Example 3: Update control panel
    updateControlPanelPassword($serviceId, $newPassword);

    // Example 4: Notify client
    $service = Capsule::table('tblhosting')->where('id', $serviceId)->first();
    send_email([
        'type' => 'product',
        'id' => $service->packageid,
        'customvars' => [
            'service_domain' => $service->domain,
        ],
    ], $service->userid);

    // Example 5: Security alert for admin changes
    if ($_SESSION['adminid']) {
        $admin = Capsule::table('tbladmins')
            ->where('id', $_SESSION['adminid'])
            ->first();

        logActivity("Admin {$admin->username} changed password for service #{$serviceId}");
    }
});
```

---

## Service View Hooks

### Custom Service Page Data

```php
<?php
/**
 * ServiceView hook
 * Add data to service details page
 */

add_hook('ServiceView', 1, function($vars) {
    $serviceId = $vars['serviceid'];

    // Example 1: Get server status
    $serverStatus = getServerStatus($serviceId);

    // Example 2: Get usage metrics
    $usage = getServiceUsage($serviceId);

    // Example 3: Get recent activity
    $activity = getRecentServiceActivity($serviceId);

    return [
        'server_online' => $serverStatus['online'],
        'server_load' => $serverStatus['load'],
        'disk_usage' => $usage['disk'],
        'bandwidth_usage' => $usage['bandwidth'],
        'recent_activity' => $activity,
        'custom_buttons' => [
            [
                'label' => 'Reboot Server',
                'url' => 'cmd.php?action=reboot&id=' . $serviceId,
                'confirm' => 'Are you sure you want to reboot?',
            ],
            [
                'label' => 'Console Access',
                'url' => 'cmd.php?action=console&id=' . $serviceId,
                'target' => '_blank',
            ],
        ],
    ];
});

/**
 * Get server status from provider
 */
function getServerStatus(int $serviceId): array
{
    $service = Capsule::table('tblhosting')
        ->where('id', $serviceId)
        ->first();

    // API call to get real-time status
    $api = new ProviderApi($service->server);
    $status = $api->getStatus($service->dedicatedip);

    return [
        'online' => $status['online'] ?? false,
        'load' => $status['load'] ?? '0.00',
        'uptime' => $status['uptime'] ?? 0,
    ];
}
```

---

## Scheduled Service Hooks

### Daily Service Maintenance

```php
<?php
/**
 * DailyCronJob hook
 * Execute daily service maintenance tasks
 */

add_hook('DailyCronJob', 1, function($vars) {
    // Example 1: Check expiring services
    $expiringServices = Capsule::table('tblhosting')
        ->where('nextduedate', date('Y-m-d', strtotime('+3 days')))
        ->where('domainstatus', 'Active')
        ->get();

    foreach ($expiringServices as $service) {
        // Send reminder
        sendExpirationReminder($service);
    }

    // Example 2: Process overdue services
    $overdueServices = Capsule::table('tblhosting')
        ->where('domainstatus', 'Active')
        ->where('nextduedate', '<', date('Y-m-d', strtotime('-7 days')))
        ->get();

    foreach ($overdueServices as $service) {
        // Grace period has passed, initiate suspension
        if (!$service->suspension_date) {
            initiateServiceSuspension($service);
        }
    }

    // Example 3: Clean up terminated services
    Capsule::table('tblhosting')
        ->where('domainstatus', 'Terminated')
        ->where('termination_date', '<', date('Y-m-d', strtotime('-30 days')))
        ->delete();

    // Example 4: Generate usage reports
    generateBandwidthReports();
    generateDiskUsageReports();
});

/**
 * Send expiration reminder
 */
function sendExpirationReminder($service): void
{
    $client = Capsule::table('tblclients')
        ->where('id', $service->userid)
        ->first();

    send_email([
        'type' => 'product',
        'id' => $service->packageid,
        'customvars' => [
            'service_domain' => $service->domain,
            'expiry_date' => $service->nextduedate,
            'renewal_url' => getRenewalUrl($service->id),
        ],
    ], $client->id);
}
```

---

## Best Practices

1. **Always log actions** - Track all service lifecycle events
2. **Handle failures gracefully** - Use try-catch and transaction rollback
3. **Async heavy operations** - Queue long-running tasks
4. **Notify appropriately** - Keep clients informed
5. **Maintain audit trail** - Record who did what and when
6. **Test thoroughly** - Test each lifecycle scenario
7. **Consider reversibility** - Plan for rollback scenarios

---

## Related Documentation

- [Hook System Reference](whmcs-hook-system.md)
- [Hooks Reference](hooks-reference.md)
- [Lifecycle Management](../skills/whmcs-lifecycle-management.md)
- [Service Billing](../skills/whmcs-service-billing.md)
