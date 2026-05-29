# WHMCS ClientDelete Hook Reference

## Overview

The `ClientDelete` hook fires when a client account is deleted from WHMCS. This hook executes before the actual deletion, allowing you to perform cleanup tasks, archive data, or prevent deletion under certain conditions.

## Hook Parameters

| Parameter | Type | Description |
|-----------|------|-------------|
| `userid` | int | The unique client ID being deleted |
| `email` | string | Client's email address |
| `firstname` | string | Client's first name |
| `lastname` | string | Client's last name |

## Example Implementation

```php
<?php
add_hook('ClientDelete', 1, function(array $params) {
    // Archive client data before deletion
    archiveClientData($params['userid']);
    
    // Remove from external systems
    removeFromExternalCRM($params['userid']);
    
    // Log the deletion for audit purposes
    logActivity("Client account deleted: {$params['email']} (ID: {$params['userid']})");
    
    // Cancel pending subscriptions
    cancelExternalSubscriptions($params['userid']);
    
    return $params;
});
```

## Preventing Client Deletion

You can prevent deletion by returning a `false` or error message:

```php
<?php
add_hook('ClientDelete', 1, function(array $params) {
    // Check if client has active services
    $result = full_query("
        SELECT COUNT(*) as count FROM tblhosting 
        WHERE userid = " . (int)$params['userid'] . " 
        AND domainstatus = 'Active'
    ");
    $data = mysql_fetch_array($result);
    
    if ($data['count'] > 0) {
        return [
            'success' => false,
            'errorMessage' => 'Cannot delete client with active services'
        ];
    }
    
    // Check for pending invoices
    $result = full_query("
        SELECT COUNT(*) as count FROM tblinvoices 
        WHERE userid = " . (int)$params['userid'] . " 
        AND status IN ('Unpaid', 'Payment Pending')
    ");
    $data = mysql_fetch_array($result);
    
    if ($data['count'] > 0) {
        return [
            'success' => false,
            'errorMessage' => 'Cannot delete client with pending invoices'
        ];
    }
    
    return $params;
});
```

## Complete Deletion Workflow

```php
<?php
add_hook('ClientDelete', 1, function(array $params) {
    $userId = (int)$params['userid'];
    
    // 1. Export all client data to JSON
    $clientData = exportClientToJSON($userId);
    saveToBackupTable($userId, 'deleted_clients', $clientData);
    
    // 2. Remove from email marketing systems
    removeFromMailchimp($params['email']);
    removeFromNewsletter($params['email']);
    
    // 3. Cancel all active module services
    $services = mysql_fetch_all(full_query(
        "SELECT id, packageid FROM tblhosting WHERE userid = $userId"
    ));
    foreach ($services as $service) {
        // Terminate each service first
        TerminateServiceModule($service['id']);
    }
    
    // 4. Cancel domains
    $domains = mysql_fetch_all(full_query(
        "SELECT id FROM tbldomains WHERE userid = $userId"
    ));
    foreach ($domains as $domain) {
        // Handle domain transfer away or cancellation
    }
    
    // 5. Log for GDPR compliance
    logActivity("Client data archived prior to deletion: User ID {$userId}");
    
    return $params;
});
```

## Use Cases

- **GDPR Compliance**: Archive data before deletion
- **Data Backup**: Export client data to external storage
- **Service Cleanup**: Terminate associated services first
- **Third-party Sync**: Remove from external systems
- **Audit Logging**: Track all deletions

## Notes

- This hook executes BEFORE the deletion occurs
- Return `false` or an error array to prevent deletion
- Consider running cleanup in separate hooks for better organization
- Active services must typically be terminated before deletion

## Related Hooks

- `ClientAdd` - New client registration
- `ClientEdit` - Client profile updates
- `ServiceDelete` - Service deletion

## See Also

- [WHMCS Hooks Documentation](https://docs.whmcs.com/Hooks)
- [GDPR Documentation](../whmcs-gdpr-compliance.md)