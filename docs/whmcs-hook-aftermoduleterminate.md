# WHMCS AfterModuleTerminate Hook Reference

## Overview

The `AfterModuleTerminate` hook fires after a hosting service is terminated through its module. This hook triggers when WHMCS successfully communicates with the server to permanently terminate the service.

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
| `terminated` | bool | Whether termination was successful |

## Example Implementation

```php
<?php
add_hook('AfterModuleTerminate', 1, function(array $params) {
    // Log the termination
    logActivity("Service terminated: {$params['domain']} (ID: {$params['serviceid']})");
    
    // Remove from all external systems
    removeFromAllExternalServices($params['serviceid']);
    
    // Archive service data
    archiveServiceData($params['serviceid']);
    
    return $params;
});
```

## Data Preservation

```php
<?php
add_hook('AfterModuleTerminate', 1, function(array $params) {
    /** @var \WHMCS\Service\Service $model */
    $model = $params['model'];
    
    // 1. Export all service data to JSON
    $exportData = [
        'service_id' => $params['serviceid'],
        'domain' => $params['domain'],
        'username' => $params['username'],
        'terminated_at' => date('Y-m-d H:i:s'),
        'usage_history' => getServiceUsageHistory($params['serviceid']),
        'files' => exportServiceFiles($params['serviceid']),
        'databases' => exportServiceDatabases($params['serviceid']),
        'emails' => exportServiceEmails($params['domain'])
    ];
    
    // 2. Save to backup storage
    saveToBackupStorage('terminated_services', $params['serviceid'], $exportData);
    
    // 3. Mark service as terminated in custom table
    insert_query('tbl_terminated_services', [
        'service_id' => $params['serviceid'],
        'original_domain' => $params['domain'],
        'terminated_date' => date('Y-m-d H:i:s'),
        'data_snapshot' => json_encode($exportData),
        'retention_until' => date('Y-m-d', strtotime('+30 days'))
    ]);
    
    // 4. Log for compliance
    logActivity("Service data archived before termination: {$params['domain']}");
    
    return $params;
});
```

## Cleanup Operations

```php
<?php
add_hook('AfterModuleTerminate', 1, function(array $params) {
    $serviceId = (int)$params['serviceid'];
    $domain = $params['domain'];
    
    // 1. Remove DNS records
    removeDNSRecords($domain);
    
    // 2. Remove SSL certificates
    removeSSLCertificate($domain);
    
    // 3. Remove from monitoring
    removeFromMonitoring($domain);
    
    // 4. Remove from CDN
    removeFromCDN($domain);
    
    // 5. Remove backups (after retention period)
    scheduleBackupDeletion($serviceId, '+30 days');
    
    // 6. Remove monitoring checks
    removeMonitoringChecks($domain);
    
    // 7. Update analytics
    updateAnalytics('service_terminated', $params['pid']);
    
    return $params;
});
```

## Post-Termination Notification

```php
<?php
add_hook('AfterModuleTerminate', 1, function(array $params) {
    // Get client details
    $client = getClientsDetails($params['userid']);
    
    // Send termination confirmation
    sendTemplatedEmail('Service Terminated', $client['email'], [
        'service_domain' => $params['domain'],
        'service_id' => $params['serviceid'],
        'terminated_date' => date('Y-m-d H:i:s')
    ]);
    
    // Notify administrators
    sendAdminNotification('email', [
        'subject' => "Service Terminated: {$params['domain']}",
        'message' => "The service has been terminated.\n\n" .
                     "Service ID: {$params['serviceid']}\n" .
                     "Domain: {$domain}\n" .
                     "Client: {$client['fullname']} ({$params['userid']})"
    ]);
    
    // Create internal ticket for data cleanup verification
    createSupportTicket([
        'userid' => $params['userid'],
        'subject' => "Service Termination Complete: {$params['domain']}",
        'message' => "Service has been terminated. Please verify cleanup tasks are complete.",
        'priority' => 'Low'
    ]);
    
    return $params;
});
```

## Use Cases

- **Data Archival**: Preserve service data before deletion
- **Cleanup Operations**: Remove DNS, SSL, CDN configurations
- **External Sync**: Remove from all external systems
- **Notifications**: Alert staff and customers
- **Compliance**: Maintain audit trails for terminated services

## Notes

- This hook fires after successful module termination
- Service data may still exist in WHMCS database
- Consider GDPR implications when deleting customer data
- Data retention policies should be defined before deletion

## Related Hooks

- `AfterModuleSuspend` - After service suspension
- `AfterModuleCreate` - After service creation
- `ServiceDelete` - When service is deleted from WHMCS

## See Also

- [WHMCS Hooks Documentation](https://docs.whmcs.com/Hooks)
- [Data Retention Policy](../whmcs-data-retention.md)