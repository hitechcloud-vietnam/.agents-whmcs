# WHMCS AfterModuleSuspend Hook Reference

## Overview

The `AfterModuleSuspend` hook fires after a hosting service is successfully suspended through its module. This hook triggers when WHMCS successfully communicates with the server to suspend the service.

## Hook Parameters

| Parameter | Type | Description |
|-----------|------|-------------|
| `serviceid` | int | The unique service ID |
| `userid` | int | The client ID |
| `pid` | int | The product ID |
| `domain` | string | The service domain |
| `username` | string | The provisioning username |
| `model` | object | The Service model instance |
| `params` | array | Raw module params array |

## Example Implementation

```php
<?php
add_hook('AfterModuleSuspend', 1, function(array $params) {
    // Log the suspension
    logActivity("Service suspended: {$params['domain']} (ID: {$params['serviceid']})");
    
    // Update local database status
    update_query('tblhosting', [
        'suspendreason' => 'Suspended due to non-payment'
    ], ['id' => $params['serviceid']]);
    
    // Remove from load balancer
    removeFromLoadBalancer($params['domain']);
    
    // Update DNS
    updateDNSForSuspendedService($params['domain']);
    
    return $params;
});
```

## Monitoring Suspension Status

```php
<?php
add_hook('AfterModuleSuspend', 1, function(array $params) {
    /** @var \WHMCS\Service\Service $model */
    $model = $params['model'];
    
    // Record suspension timestamp
    $model->suspendTime = date('Y-m-d H:i:s');
    $model->save();
    
    // Send notification to admin
    sendAdminNotification('email', [
        'subject' => "Service Suspended: {$params['domain']}",
        'message' => "Service ID: {$params['serviceid']}\n" .
                     "Client ID: {$params['userid']}\n" .
                     "Domain: {$params['domain']}"
    ]);
    
    // Update monitoring system
    updateMonitoringStatus($params['domain'], 'suspended');
    
    return $params;
});
```

## Complete Suspension Workflow

```php
<?php
add_hook('AfterModuleSuspend', 1, function(array $params) {
    $serviceId = (int)$params['serviceid'];
    
    // 1. Revoke API access
    revokeServiceAPIKeys($serviceId);
    
    // 2. Disable CDN if applicable
    disableServiceCDN($params['domain']);
    
    // 3. Update SSL certificate status
    updateSSLCertificateStatus($params['domain'], 'suspended');
    
    // 4. Remove from caching layer
    purgeServiceCache($params['domain']);
    
    // 5. Create audit log entry
    createAuditLog('suspend', $serviceId, [
        'reason' => 'module_suspend',
        'timestamp' => date('Y-m-d H:i:s')
    ]);
    
    // 6. Notify client
    $client = getClientsDetails($params['userid']);
    sendTemplatedEmail('Service Suspended', $client['email'], [
        'service_domain' => $params['domain']
    ]);
    
    return $params;
});
```

## Detecting Suspension Reason

```php
<?php
add_hook('AfterModuleSuspend', 1, function(array $params) {
    /** @var \WHMCS\Service\Service $model */
    $model = $params['model'];
    
    // Check why suspension occurred
    $pendingInvoices = getUnpaidInvoicesForService($params['serviceid']);
    
    if (!empty($pendingInvoices)) {
        $reason = 'unpaid_invoices';
        $invoiceIds = array_column($pendingInvoices, 'id');
        
        logActivity("Service suspended for non-payment. Invoices: " . 
            implode(', ', $invoiceIds));
    } else {
        $reason = 'manual';
    }
    
    // Store suspension reason in custom field
    saveCustomFieldValue($model, 'Suspension Reason', $reason);
    
    return $params;
});
```

## Use Cases

- **Load Balancer Management**: Remove from active pool
- **DNS Updates**: Point to suspension page
- **CDN/Cache Management**: Disable CDN, purge cache
- **API Key Revocation**: Invalidate service API keys
- **Notification System**: Alert staff of suspensions

## Notes

- This hook fires only on SUCCESSFUL suspension
- Use `ModuleSuspendFailed` for handling failures
- The module must support suspension for this hook to fire
- Consider combining with `DailyCronJob` for automated suspensions

## Related Hooks

- `AfterModuleUnsuspend` - After service is unsuspended
- `AfterModuleTerminate` - After service termination
- `DailyCronJob` - For automated suspension checks

## See Also

- [WHMCS Hooks Documentation](https://docs.whmcs.com/Hooks)
- [Module Suspend Function](../whmcs-module-development.md)