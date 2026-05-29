# WHMCS ClientAdd Hook Reference

## Overview

The `ClientAdd` hook fires when a new client is successfully created in WHMCS. This hook allows you to execute custom code whenever a client account is added to the system.

## Hook Parameters

| Parameter | Type | Description |
|-----------|------|-------------|
| `userid` | int | The unique client ID |
| `firstname` | string | Client's first name |
| `lastname` | string | Client's last name |
| `companyname` | string | Company name (if provided) |
| `email` | string | Client's email address |
| `address1` | string | Address line 1 |
| `address2` | string | Address line 2 |
| `city` | string | City |
| `state` | string | State/Region |
| `country` | string | Country code (2 chars) |
| `phonenumber` | string | Phone number |
| `password` | string | Encrypted password |
| `currency` | int | Currency ID |

## Example Implementation

```php
<?php
use WHMCS\View\Helper\Hook;

add_hook('ClientAdd', 1, function(array $params) {
    // Log new client registration
    logActivity("New client registered: {$params['email']}");
    
    // Send welcome email via external service
    $clientName = $params['firstname'] . ' ' . $params['lastname'];
    sendWelcomeEmail($params['email'], $clientName);
    
    // Create account in external CRM
    syncToCRM($params);
    
    // Assign default client group
    update_query('tblclients', ['groupid' => 1], ['id' => $params['userid']]);
    
    return $params;
});
```

## Use Cases

- **CRM Integration**: Sync new clients to external CRM systems
- **Welcome Emails**: Trigger welcome email sequences
- **Analytics**: Track new client signups
- **Auto-Tagging**: Assign default tags or groups
- **Audit Logging**: Log all new client registrations

## Notes

- This hook executes AFTER the client is added to the database
- The `userid` is already assigned and can be used for related queries
- Password is hashed and should not be accessed directly
- Return the `$params` array if you want to allow other hooks to process

## Related Hooks

- `ClientEdit` - When client details are modified
- `ClientDelete` - When a client is deleted
- `AfterCartCheckout` - After checkout completion

## See Also

- [WHMCS Hooks Documentation](https://docs.whmcs.com/Hooks)
- [Hook System Overview](../whmcs-hooks-reference.md)