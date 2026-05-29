# WHMCS ServiceDelete Hook Reference

## Overview

The `ServiceDelete` hook fires when a hosting service is deleted from WHMCS. This hook executes before the deletion, allowing for data preservation, cleanup operations, and external system synchronization.

## Hook Parameters

| Parameter | Type | Description |
|-----------|------|-------------|
| `serviceid` | int | The unique service ID |
| `userid` | int | The client ID |
| `pid` | int | The product ID |
| `domain` | string | The service domain |
| `username` | string | The provisioning username |
| `model` | object | The Service model instance |

## Example Implementation

```php
<?php
add_hook('ServiceDelete', 1, function(array $params) {
    // Log the deletion
    logActivity("Service deleted: {$params['domain']} (ID: {$params['serviceid']})");
    
    // Archive service data
    archiveServiceData($params['serviceid']);
    
    return $params;
});
```

## Data Preservation

```php
<?php
add_hook('ServiceDelete', 1, function(array $params) {
    /** @var \WHMCS\Service\Service $model */
    $model = $params['model'];
    
    // 1. Export complete service data
    $serviceSnapshot = [
        'service_id' => $params['serviceid'],
        'domain' => $params['domain'],
        'username' => $params['username'],
        'product_id' => $params['pid'],
        'created_date' => $model->registrationDate,
        'terminated_date' => date('Y-m-d H:i:s'),
        'usage_data' => getServiceUsageStats($params['serviceid']),
        'custom_fields' => getCustomFieldValues($params['serviceid']),
        'addons' => getServiceAddons($params['serviceid']),
        'config_options' => getConfigOptions($params['serviceid'])
    ];
    
    // 2. Save to archive table
    insert_query('tbl_service_archive', [
        'service_id' => $params['serviceid'],
        'snapshot' => json_encode($serviceSnapshot),
        'archived_at' => date('Y-m-d H:i:s')
    ]);
    
    // 3. Export files list for potential retrieval
    $fileList = getServiceFilesList($params['serviceid']);
    insert_query('tbl_service_files_archive', [
        'service_id' => $params['serviceid'],
        'files' => json_encode($fileList)
    ]);
    
    return $params;
});
```

## Complete Cleanup Workflow

```php
<?php
add_hook('ServiceDelete', 1, function(array $params) {
    $serviceId = (int)$params['serviceid'];
    $domain = $params['domain'];
    
    // 1. Remove from monitoring systems
    removeMonitoringChecks($domain);
    
    // 2. Remove DNS records
    removeDNSRecords($domain);
    
    // 3. Revoke SSL certificates
    removeSSLCertificate($domain);
    
    // 4. Remove from CDN
    removeFromCDN($domain);
    
    // 5. Cancel scheduled tasks
    cancelScheduledBackups($serviceId);
    cancelScheduledScans($serviceId);
    
    // 6. Remove from external integrations
    removeFromSlackWorkspace($domain);
    removeFromAnalytics($domain);
    
    // 7. Log all cleanup operations
    createAuditLog('service_cleanup', $serviceId, [
        'domain' => $domain,
        'operations' => ['monitoring', 'dns', 'ssl', 'cdn', 'backups'],
        'completed_at' => date('Y-m-d H:i:s')
    ]);
    
    return $params;
});
```

## Preventing Service Deletion

```php
<?php
add_hook('ServiceDelete', 1, function(array $params) {
    // Check for active addons
    $activeAddons = full_query("
        SELECT COUNT(*) as count FROM tblhostingaddons 
        WHERE hostingid = " . (int)$params['serviceid'] . " 
        AND status = 'Active'
    ");
    $addonData = mysql_fetch_array($activeAddons);
    
    if ($addonData['count'] > 0) {
        return [
            'success' => false,
            'errorMessage' => 'Cannot delete service with active addons. Remove addons first.'
        ];
    }
    
    // Check for pending invoices
    $pendingInvoices = full_query("
        SELECT COUNT(*) as count FROM tblinvoiceitems 
        WHERE relid = " . (int)$params['serviceid'] . " 
        AND type = 'Hosting'
        AND invoiceid IN (SELECT id FROM tblinvoices WHERE status IN ('Unpaid', 'Payment Pending'))
    ");
    $invoiceData = mysql_fetch_array($pendingInvoices);
    
    if ($invoiceData['count'] > 0) {
        return [
            'success' => false,
            'errorMessage' => 'Cannot delete service with pending invoices'
        ];
    }
    
    return $params;
});
```

## Use Cases

- **Data Archival**: Preserve service data before deletion
- **Cleanup Operations**: Remove from external systems
- **Validation**: Prevent deletion under certain conditions
- **Audit Trail**: Maintain deletion records
- **GDPR Compliance**: Handle data retention requirements

## Notes

- Runs BEFORE the service is deleted from database
- Return false or error array to prevent deletion
- Consider GDPR and data retention policies
- Module termination happens via `AfterModuleTerminate`

## Related Hooks

- `AfterModuleTerminate` - After module termination
- `ClientDelete` - When client is deleted
- `AfterProductCreate` - When service is created

## See Also

- [WHMCS Hooks Documentation](https://docs.whmcs.com/Hooks)
- [Data Retention](../whmcs-data-retention.md)