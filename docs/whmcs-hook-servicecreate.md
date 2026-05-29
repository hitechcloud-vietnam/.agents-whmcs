# WHMCS ServiceCreate Hook Reference

## Overview

The `ServiceCreate` hook fires when a new hosting service is created in WHMCS. This hook can be used to add custom processing when services are provisioned through orders or manual admin creation.

## Hook Parameters

| Parameter | Type | Description |
|-----------|------|-------------|
| `serviceid` | int | The unique service ID |
| `userid` | int | The client ID |
| `pid` | int | The product ID |
| `domain` | string | The service domain |
| `username` | string | The provisioning username |
| `password` | string | The provisioning password (encrypted) |
| `billingcycle` | string | Billing cycle (Monthly, Quarterly, etc.) |
| `amount` | float | Service price |
| `customfields` | array | Custom field values |
| `configoptions` | array | Configurable options |
| `model` | object | The Service model instance |

## Example Implementation

```php
<?php
add_hook('ServiceCreate', 1, function(array $params) {
    // Log the service creation
    logActivity("Service created: {$params['domain']} (ID: {$params['serviceid']})");
    
    // Apply custom pricing based on product
    if ($params['pid'] == 10) {
        update_query('tblhosting', [
            'subscriptionid' => generateSubscriptionId()
        ], ['id' => $params['serviceid']]);
    }
    
    return $params;
});
```

## Custom Field Processing

```php
<?php
add_hook('ServiceCreate', 1, function(array $params) {
    /** @var \WHMCS\Service\Service $model */
    $model = $params['model'];
    
    // Process custom fields
    $customFields = $params['customfields'] ?? [];
    
    foreach ($customFields as $fieldId => $value) {
        // Example: Generate control panel URL
        if ($fieldId == 'control_panel') {
            $cpUrl = generateControlPanelURL($params['domain'], $value);
            saveCustomField($model, 'CP URL', $cpUrl);
        }
        
        // Example: Set server based on location
        if ($fieldId == 'datacenter_location') {
            $assignedServer = assignServerByLocation($value);
            $model->server = $assignedServer;
            $model->save();
        }
    }
    
    return $params;
});
```

## Service Configuration

```php
<?php
add_hook('ServiceCreate', 1, function(array $params) {
    $serviceId = (int)$params['serviceid'];
    
    // 1. Set up backup schedule
    setupBackupSchedule($serviceId, 'daily');
    
    // 2. Configure monitoring
    setupMonitoring([
        'domain' => $params['domain'],
        'service_id' => $serviceId,
        'checks' => ['http', 'ping', 'ssl']
    ]);
    
    // 3. Create default email aliases
    createDefaultAliases($params['domain'], $params['userid']);
    
    // 4. Set resource limits based on config options
    if (isset($params['configoptions']['storage'])) {
        $storageLimit = $params['configoptions']['storage'];
        setStorageQuota($serviceId, $storageLimit);
    }
    
    // 5. Initialize support tier
    $supportTier = determineSupportTier($params['pid']);
    saveCustomField($model, 'Support Tier', $supportTier);
    
    return $params;
});
```

## Validation and Assignment

```php
<?php
add_hook('ServiceCreate', 1, function(array $params) {
    // Assign to specific server based on product
    $serverGroup = getServerGroupForProduct($params['pid']);
    $availableServer = getAvailableServer($serverGroup);
    
    if ($availableServer) {
        update_query('tblhosting', [
            'server' => $availableServer['id']
        ], ['id' => $params['serviceid']]);
    } else {
        logActivity("No available server in group {$serverGroup} for service {$params['serviceid']}");
    }
    
    // Check capacity limits
    $currentCount = getServiceCountForProduct($params['pid']);
    $maxCapacity = getProductMaxCapacity($params['pid']);
    
    if ($currentCount >= $maxCapacity) {
        logActivity("Product {$params['pid']} at capacity: {$currentCount}/{$maxCapacity}");
        // Send alert to admin
        sendAdminNotification('system', [
            'subject' => 'Product Capacity Warning',
            'message' => "Product ID {$params['pid']} is at {$currentCount}/{$maxCapacity} capacity"
        ]);
    }
    
    return $params;
});
```

## Use Cases

- **Server Assignment**: Auto-assign servers based on criteria
- **Custom Configuration**: Set up additional service options
- **Resource Limits**: Configure based on config options
- **Monitoring Setup**: Add to monitoring systems
- **Custom Fields**: Generate or process custom field values

## Notes

- Runs after service is created but may be before provisioning
- Use `AfterProductCreate` for post-provisioning tasks
- Module provisioning happens separately via `AfterModuleCreate`
- Can modify service data before final save

## Related Hooks

- `AfterProductCreate` - After product is created
- `AfterModuleCreate` - After module provisioning
- `ServiceDelete` - When service is deleted

## See Also

- [WHMCS Hooks Documentation](https://docs.whmcs.com/Hooks)
- [Service Management](../whmcs-service-management.md)