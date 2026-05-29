# WHMCS AfterProductCreate Hook Reference

## Overview

The `AfterProductCreate` hook fires after a new hosting product/service is created in WHMCS. This hook triggers when provisioning a new service through the admin area or during the order fulfillment process.

## Hook Parameters

| Parameter | Type | Description |
|-----------|------|-------------|
| `serviceid` | int | The unique service ID |
| `userid` | int | The client ID who owns this service |
| `pid` | int | The product ID |
| `domain` | string | The service domain/subdomain |
| `username` | string | The provisioning username |
| `password` | string | The provisioning password (encrypted) |
| `model` | object | The Service model instance |
| `customfields` | array | Custom field values |
| `configoptions` | array | Configurable options selected |

## Example Implementation

```php
<?php
add_hook('AfterProductCreate', 1, function(array $params) {
    // Log the service creation
    logActivity("New service created: {$params['domain']} (Service ID: {$params['serviceid']})");
    
    // Set custom expiry date based on product
    if ($params['pid'] == 5) { // Example product ID
        update_query('tblhosting', [
            'nextduedate' => date('Y-m-d', strtotime('+1 year'))
        ], ['id' => $params['serviceid']]);
    }
    
    // Notify external systems
    notifyProvisioningSystem($params);
    
    // Set first payment as completed if using trial
    if (hasTrialPeriod($params['pid'])) {
        update_query('tblhosting', [
            'firstpaymentamount' => 0.00
        ], ['id' => $params['serviceid']]);
    }
    
    return $params;
});
```

## Working with the Service Model

```php
<?php
add_hook('AfterProductCreate', 1, function(array $params) {
    /** @var \WHMCS\Service\Service $model */
    $model = $params['model'];
    
    // Access service properties
    $domain = $model->domain;
    $client = $model->client;
    $product = $model->product;
    
    // Add service notes
    $model->notes .= "\n[Auto] Service provisioned on " . date('Y-m-d H:i:s');
    $model->save();
    
    // Get custom fields
    $customFieldValues = [];
    foreach ($model->customFields as $field) {
        $customFieldValues[$field->id] = $field->value;
    }
    
    // Update service with additional data
    $model->subscriptionId = generateSubscriptionId();
    $model->save();
    
    return $params;
});
```

## Post-Creation Automation

```php
<?php
add_hook('AfterProductCreate', 1, function(array $params) {
    $serviceId = (int)$params['serviceid'];
    
    // 1. Setup monitoring
    setupServiceMonitoring($params['domain'], $serviceId);
    
    // 2. Create support ticket for setup tasks
    createTicket('setup', $params['userid'], $serviceId);
    
    // 3. Add to backup rotation
    addToBackupSchedule($serviceId);
    
    // 4. Register SSL if applicable
    if ($params['configoptions']['ssl'] ?? false) {
        provisionFreeSSL($params['domain']);
    }
    
    // 5. Setup email accounts
    if ($params['configoptions']['email_accounts'] ?? 0) {
        createEmailAccounts($params['domain'], $params['configoptions']['email_accounts']);
    }
    
    return $params;
});
```

## Use Cases

- **Service Monitoring**: Add new services to monitoring systems
- **Automated Setup**: Complete post-provisioning tasks
- **Notifications**: Alert staff or customers of new services
- **Custom Expiry**: Set custom renewal dates
- **Integration**: Sync with billing or inventory systems

## Notes

- This hook runs AFTER the service is created but may run BEFORE module provisioning
- Use `AfterModuleCreate` for module-specific post-provisioning tasks
- The `model` object provides full access to the service entity
- Password is encrypted - use decryption only when necessary

## Related Hooks

- `ServiceCreate` - Alternative hook for service creation
- `AfterModuleCreate` - After module provisioning completes
- `AfterProductTerminate` - After service termination

## See Also

- [WHMCS Hooks Documentation](https://docs.whmcs.com/Hooks)
- [Module Provisioning](../whmcs-module-development.md)