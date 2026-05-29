# WHMCS Client Impersonation

## Overview

Client impersonation in WHMCS allows administrators to view and interact with the client area as if they were the client. This feature is useful for troubleshooting, testing user experiences, and resolving client issues directly.

## Impersonation Access

### Enabling Impersonation

**Configuration > Security > Admin Impersonation**

```php
[
    'allow_impersonation' => true,
    'require_confirmation' => true,
    'log_all_actions' => true,
    'notify_client' => false,
    'auto_logout_minutes' => 30
]
```

## Impersonating a Client

### Start Impersonation

**Admin: Clients > Select Client > Impersonate**

```php
// Start impersonation
[
    'userid' => 123,
    'adminid' => 1,
    'started_at' => '2024-05-15 10:30:00',
    'ip_address' => '192.168.1.1',
    'session_token' => 'xxx'
]
```

### Impersonation Banner

When impersonating, admin sees visible banner:

```html
<div class="impersonation-banner">
    <span>You are impersonating: John Doe (client@example.com)</span>
    <a href="/admin/停止impersonation.php">Stop Impersonation</a>
</div>
```

## Impersonation Permissions

### Role-Based Access

```php
// Who can impersonate
[
    'roles' => [
        'admin' => true,
        'owner' => true,
        'manager' => true,
        'support' => false,
        'billing' => false
    ]
]
```

### Permission Check

```php
// Check if admin can impersonate
function canImpersonate($adminId) {
    $admin = getAdmin($adminId);
    return in_array($admin['role'], ['admin', 'owner', 'manager']);
}
```

## Impersonation Actions

### Allowed Actions

```php
// Actions available during impersonation
[
    'view_services' => true,
    'view_invoices' => true,
    'view_domains' => true,
    'view_tickets' => true,
    'submit_tickets' => true,
    'pay_invoices' => true,
    'update_profile' => true
]
```

### Restricted Actions

```php
// Actions blocked during impersonation
[
    'change_password' => false,
    'delete_account' => false,
    'transfer_service' => false,
    'change_billing' => false,
    'api_access' => false
]
```

## Impersonation Logging

### Activity Log

```php
// Log impersonation actions
[
    'log_id' => 456,
    'admin_id' => 1,
    'userid' => 123,
    'action' => 'view_service',
    'details' => 'Viewed service #1',
    'timestamp' => '2024-05-15 10:35:00',
    'ip' => '192.168.1.1'
]
```

### Audit Trail

```php
// Complete impersonation history
[
    'impersonation_id' => 789,
    'admin_id' => 1,
    'admin_name' => 'Admin User',
    'userid' => 123,
    'client_name' => 'John Doe',
    'started' => '2024-05-15 10:30:00',
    'ended' => '2024-05-15 11:00:00',
    'duration_minutes' => 30,
    'actions_count' => 15,
    'closed_by' => 'admin'
]
```

## Impersonation Session

### Session Timeout

```php
// Auto-end impersonation
[
    'session_timeout' => 30,            // minutes
    'max_session' => 60,               // max minutes
    'auto_end_on_inactivity' => true,
    'inactivity_timeout' => 15          // minutes
]
```

### Manual End Session

```php
// End impersonation
[
    'action' => 'end_impersonation',
    'admin_id' => 1,
    'userid' => 123,
    'ended_at' => '2024-05-15 10:45:00',
    'reason' => 'manual'                // manual, timeout, admin_end
]
```

## Impersonation Notifications

### Admin Notifications

```php
// Notify on impersonation start/end
[
    'notify_admin_manager' => true,
    'notify_email' => 'security@example.com',
    'include_client_details' => true
]
```

### Client Notifications (Optional)

```smarty
Subject: Admin Access to Your Account

Dear {$client_name},

An administrator accessed your account on {$date} at {$time}.
Duration: {$duration} minutes

If you did not authorize this, please contact us immediately.

{$company_name}
```

## Security Considerations

### Impersonation Security

```php
// Security measures
[
    'require_password_reentry' => true,
    'session_ip_binding' => true,
    'unique_session_token' => true,
    'limit_concurrent' => true
]
```

### Prevent Abuse

```php
// Controls to prevent misuse
[
    'log_every_action' => true,
    'require_reason' => true,
    'approval_for_sensitive' => true,
    'auto_report' => true
]
```

## Impersonation Reports

### Impersonation History Report

**Reports > Admin > Impersonation Log**

```php
// Report data
[
    'period' => 'May 2024',
    'total_impersonations' => 25,
    'unique_admins' => 5,
    'unique_clients' => 20,
    'avg_duration_minutes' => 15,
    'actions_per_session' => 8
]
```

## API Functions

```php
// Start impersonation
$result = localAPI('StartClientImpersonation', [
    'clientid' => 123
]);

// End impersonation
$result = localAPI('EndClientImpersonation');

// Get impersonation log
$result = localAPI('GetImpersonationLog', [
    'adminid' => 1
]);
```

## Hooks

```php
// Hook: ImpersonationStarted
add_hook('ImpersonationStarted', 1, function($vars) {
    // $vars['adminid']
    // $vars['userid']
    // Log start, notify, etc.
});

// Hook: ImpersonationEnded
add_hook('ImpersonationEnded', 1, function($vars) {
    // $vars['adminid']
    // $vars['userid']
    // $vars['duration']
    // Log end, create summary, etc.
});
```

## Best Practices

1. **Log everything**: Track all impersonation activities
2. **Limit access**: Only allow trusted admins
3. **Short sessions**: Auto-timeout after reasonable period
4. **Regular review**: Audit impersonation logs
5. **Clear purpose**: Require reason for impersonation

## Related Documentation

- [Client Portal](./whmcs-client-portal.md)
- [Client Authentication](./whmcs-client-authentication.md)
- [Security Settings](./whmcs-security-settings.md)
- [Admin Permissions](./whmcs-admin-permissions.md)