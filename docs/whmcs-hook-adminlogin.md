# WHMCS AdminLogin Hook Reference

## Overview

The `AdminLogin` hook fires when an admin successfully logs into the WHMCS admin area. This hook enables admin-specific session management, security monitoring, and audit logging.

## Hook Parameters

| Parameter | Type | Description |
|-----------|------|-------------|
| `adminid` | int | Admin user ID |
| `username` | string | Admin username |
| `password` | string | Password (may be masked) |
| `loginkey` | string | Session login key |
| `authz` | array | Authorization roles |

## Example Implementation

```php
<?php
add_hook('AdminLogin', 1, function(array $params) {
    // Log admin login
    logActivity("Admin login: {$params['username']} from IP: " . $_SERVER['REMOTE_ADDR']);
    
    // Track login details
    full_query("
        INSERT INTO tbladmin_logins (admin_id, ip, user_agent, timestamp, success) 
        VALUES ({$params['adminid']}, '" . db_escape($_SERVER['REMOTE_ADDR']) . "', 
                '" . db_escape($_SERVER['HTTP_USER_AGENT']) . "', NOW(), 1)
    ");
    
    return $params;
});
```

## Admin Session Management

```php
<?php
add_hook('AdminLogin', 1, function(array $params) {
    // 1. Set admin session preferences
    $_SESSION['admin_initial_login'] = isFirstAdminLogin($params['adminid']);
    $_SESSION['admin_login_timestamp'] = time();
    $_SESSION['admin_last_activity'] = time();
    
    // 2. Update admin last login
    update_query('tbladmins', [
        'lastlogin' => date('Y-m-d H:i:s'),
        'last_login_ip' => db_escape($_SERVER['REMOTE_ADDR'])
    ], ['id' => $params['adminid']]);
    
    // 3. Reset failed login attempts
    update_query('tbladmins', [
        'failedloginattempts' => 0
    ], ['id' => $params['adminid']]);
    
    // 4. Load admin permissions cache
    $_SESSION['admin_permissions'] = getAdminPermissions($params['adminid']);
    
    return $params;
});
```

## Security Monitoring

```php
<?php
add_hook('AdminLogin', 1, function(array $params) {
    // 1. Check for suspicious login location
    $lastLogin = getLastAdminLogin($params['adminid']);
    $currentIP = $_SERVER['REMOTE_ADDR'];
    $currentCountry = getIPCountry($currentIP);
    
    if ($lastLogin && $lastLogin['country'] !== $currentCountry) {
        $timeSinceLast = time() - strtotime($lastLogin['timestamp']);
        
        // Impossible travel detection
        if ($timeSinceLast < 3600) {
            // Alert security team
            sendSecurityAlert([
                'type' => 'impossible_travel',
                'admin' => $params['username'],
                'previous_location' => $lastLogin['country'],
                'current_location' => $currentCountry,
                'previous_ip' => $lastLogin['ip']
            ]);
        }
    }
    
    // 2. Check if admin is logging in from new IP
    $knownIPs = getAdminKnownIPs($params['adminid']);
    if (!in_array($currentIP, $knownIPs)) {
        // First time from this IP - could require approval or notify
        insert_query('tbl_admin_new_ips', [
            'admin_id' => $params['adminid'],
            'ip' => $currentIP,
            'user_agent' => $_SERVER['HTTP_USER_AGENT'],
            'first_seen' => date('Y-m-d H:i:s')
        ]);
    }
    
    // 3. Verify admin still has required role
    if (!hasActiveRole($params['adminid'])) {
        logActivity("Login denied: Admin {$params['username']} has inactive role");
        return ['success' => false, 'errorMessage' => 'Account role is inactive'];
    }
    
    return $params;
});
```

## Audit and Compliance

```php
<?php
add_hook('AdminLogin', 1, function(array $params) {
    // 1. Create audit log entry
    insert_query('tbl_admin_audit_log', [
        'admin_id' => $params['adminid'],
        'action' => 'login',
        'ip_address' => $_SERVER['REMOTE_ADDR'],
        'user_agent' => $_SERVER['HTTP_USER_AGENT'],
        'timestamp' => date('Y-m-d H:i:s'),
        'session_id' => session_id()
    ]);
    
    // 2. Update admin statistics
    $stats = getAdminStats($params['adminid']);
    update_query('tbladmins', [
        'total_logins' => ($stats['total_logins'] ?? 0) + 1
    ], ['id' => $params['adminid']]);
    
    // 3. Check if admin needs to accept updated terms
    if (requiresTermsAcceptance($params['adminid'])) {
        $_SESSION['require_terms_acceptance'] = true;
    }
    
    // 4. Load admin dashboard preferences
    $prefs = getAdminPreferences($params['adminid']);
    $_SESSION['admin_dashboard_layout'] = $prefs['dashboard_layout'] ?? 'default';
    $_SESSION['admin_notifications'] = $prefs['notifications_enabled'] ?? true;
    
    return $params;
});
```

## Use Cases

- **Session Management**: Custom admin session handling
- **Security Monitoring**: Detect suspicious logins
- **Audit Logging**: Track all admin access
- **Compliance**: GDPR, SOC2 requirements
- **Role Verification**: Check active permissions

## Notes

- Runs for admin area logins only
- Use with `AdminLoginFail` for complete coverage
- Consider IP allowlisting for admin access
- Implement MFA requirements for admin logins

## Related Hooks

- `Login` - For client login
- `AdminLoginFail` - For failed admin login
- `AdminHeadOutput` - For admin area hooks

## See Also

- [WHMCS Hooks Documentation](https://docs.whmcs.com/Hooks)
- [Admin Security](../whmcs-admin-security.md)