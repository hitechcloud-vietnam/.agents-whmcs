# WHMCS ClientEdit Hook Reference

## Overview

The `ClientEdit` hook fires when client account details are modified. This hook triggers whenever a client's profile information is updated through the WHMCS client area or admin panel.

## Hook Parameters

| Parameter | Type | Description |
|-----------|------|-------------|
| `userid` | int | The unique client ID |
| `firstname` | string | Updated first name |
| `lastname` | string | Updated last name |
| `companyname` | string | Updated company name |
| `email` | string | Updated email address |
| `address1` | string | Updated address line 1 |
| `address2` | string | Updated address line 2 |
| `city` | string | Updated city |
| `state` | string | Updated state/region |
| `country` | string | Updated country code |
| `phonenumber` | string | Updated phone number |
| `currency` | int | Currency ID |

## Example Implementation

```php
<?php
add_hook('ClientEdit', 1, function(array $params) {
    // Compare old vs new values for specific fields
    $fieldsToMonitor = ['email', 'companyname', 'phonenumber'];
    
    foreach ($fieldsToMonitor as $field) {
        if ($params[$field] !== $params['old_' . $field] ?? null) {
            logActivity("Client {$params['userid']} changed {$field}: " . 
                "{$params['old_' . $field]} -> {$params[$field]}");
        }
    }
    
    // Sync changes to external systems
    syncClientToExternalCRM($params['userid'], $params);
    
    // Send notification if email changed
    if ($params['email'] !== $params['old_email'] ?? null) {
        verifyNewEmailAddress($params['userid'], $params['email']);
    }
    
    return $params;
});
```

## Advanced: Capturing Old Values

To capture previous values, query the database before the update:

```php
<?php
add_hook('ClientEdit', 0, function(array $params) {
    // This runs first - capture old values
    $result = mysql_fetch_array(
        full_query("SELECT * FROM tblclients WHERE id = " . (int)$params['userid'])
    );
    
    // Store old values for the second hook execution
    $_SESSION['client_edit_old_values'] = $result;
    return $params;
});

add_hook('ClientEdit', 1, function(array $params) {
    // This runs after - compare values
    $oldValues = $_SESSION['client_edit_old_values'] ?? [];
    unset($_SESSION['client_edit_old_values']);
    
    $changed = [];
    foreach ($params as $key => $value) {
        if (isset($oldValues[$key]) && $oldValues[$key] !== $value) {
            $changed[$key] = ['old' => $oldValues[$key], 'new' => $value];
        }
    }
    
    if (!empty($changed)) {
        logActivity("Client {$params['userid']} profile changes: " . 
            json_encode($changed));
    }
    
    return $params;
});
```

## Use Cases

- **Audit Trail**: Track all profile changes
- **CRM Sync**: Keep external systems updated
- **Email Verification**: Handle email change verification
- **Notification System**: Alert staff or third parties of changes
- **Data Validation**: Enforce business rules on changes

## Notes

- Priority 0 executes before priority 1 for capturing old values
- Email changes may require verification flow
- Consider implementing rate limiting on profile updates
- Returns the modified `$params` array for continued processing

## Related Hooks

- `ClientAdd` - New client registration
- `ClientDelete` - Client deletion
- `Login` - Client login events

## See Also

- [WHMCS Hooks Documentation](https://docs.whmcs.com/Hooks)
- [Hook System Overview](../whmcs-hooks-reference.md)