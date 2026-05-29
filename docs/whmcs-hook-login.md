# WHMCS Login Hook Reference

## Overview

The `Login` hook fires when a client successfully logs into WHMCS. This hook triggers after successful authentication and can be used for tracking, personalization, and security enhancements.

## Hook Parameters

| Parameter | Type | Description |
|-----------|------|-------------|
| `userid` | int | The client user ID |
| `email` | string | Client's email address |
| `password` | string | (Potentially masked) password |
| `RememberMe` | bool | Whether "Remember Me" was used |
| `loginkey` | string | Login session key |
| `twoFASuccess` | bool | Whether 2FA was verified |

## Example Implementation

```php
<?php
add_hook('Login', 1, function(array $params) {
    // Log the login
    logActivity("Client login: {$params['email']}");
    
    // Get client details
    $client = getClientsDetails($params['userid']);
    
    return $params;
});
```

## Session Management

```php
<?php
add_hook('Login', 1, function(array $params) {
    // 1. Set custom session data
    $_SESSION['client_first_login'] = isFirstLogin($params['userid']);
    $_SESSION['client_tier'] = getClientTier($params['userid']);
    $_SESSION['login_timestamp'] = time();
    
    // 2. Update last login
    update_query('tblclients', [
        'lastlogin' => date('Y-m-d H:i:s'),
        'loginattempts' => 0
    ], ['id' => $params['userid']]);
    
    // 3. Set session expiry based on remember me
    if (!$params['RememberMe']) {
        ini_set('session.cookie_lifetime', 0);
    }
    
    return $params;
});
```

## Security Enhancement

```php
<?php
add_hook('Login', 1, function(array $params) {
    // 1. Track login for fraud detection
    recordLoginAttempt([
        'userid' => $params['userid'],
        'email' => $params['email'],
        'ip' => $_SERVER['REMOTE_ADDR'],
        'user_agent' => $_SERVER['HTTP_USER_AGENT'],
        'timestamp' => date('Y-m-d H:i:s'),
        'success' => true
    ]);
    
    // 2. Check for suspicious activity
    $recentLogins = getRecentLoginCount($params['userid'], '1 hour');
    if ($recentLogins > 5) {
        // Flag account for review
        flagAccountForReview($params['userid'], 'multiple_logins');
        
        // Send security alert
        $client = getClientsDetails($params['userid']);
        sendSecurityAlert($client['email'], [
            'type' => 'unusual_login_frequency',
            'ip' => $_SERVER['REMOTE_ADDR']
        ]);
    }
    
    // 3. Require re-auth for sensitive actions
    $_SESSION['twofa_verified'] = $params['twoFASuccess'] ?? false;
    $_SESSION['last_password_check'] = time();
    
    return $params;
});
```

## Personalization

```php
<?php
add_hook('Login', 1, function(array $params) {
    // 1. Load client preferences
    $prefs = getClientPreferences($params['userid']);
    $_SESSION['client_theme'] = $prefs['theme'] ?? 'default';
    $_SESSION['client_language'] = $prefs['language'] ?? 'english';
    $_SESSION['client_timezone'] = $prefs['timezone'] ?? 'UTC';
    
    // 2. Get unread notifications count
    $_SESSION['unread_notifications'] = getUnreadNotificationCount($params['userid']);
    
    // 3. Set homepage based on client type
    $client = getClientsDetails($params['userid']);
    if ($client['groupid'] == 2) { // Premium clients
        $_SESSION['redirect_after_login'] = 'dashboard-premium';
    } else {
        $_SESSION['redirect_after_login'] = 'clientarea';
    }
    
    // 4. Load cart contents for returning
    restoreAbandonedCart($params['userid']);
    
    return $params;
});
```

## Third-Party Integration

```php
<?php
add_hook('Login', 1, function(array $params) {
    // 1. Sync to CRM
    updateCRMLoginStatus($params['userid'], [
        'logged_in' => true,
        'login_time' => date('c'),
        'ip' => $_SERVER['REMOTE_ADDR']
    ]);
    
    // 2. Update marketing platform
    updateMarketingPlatform($params['userid'], 'login');
    
    // 3. Start session in external services
    if (hasExternalServiceLink($params['userid'])) {
        syncSessionToExternal($params['userid']);
    }
    
    // 4. Track in analytics
    trackAnalyticsEvent('login', [
        'user_id' => $params['userid'],
        'method' => $params['twoFASuccess'] ? '2fa' : 'password'
    ]);
    
    return $params;
});
```

## Use Cases

- **Session Management**: Set custom session data
- **Security**: Track logins, detect fraud
- **Personalization**: Load client preferences
- **Integration**: Sync with external systems
- **Analytics**: Track login patterns

## Notes

- Fires only on successful login
- Use `LoginFail` for failed login handling
- Can access session via `$_SESSION`
- Consider 2FA verification status

## Related Hooks

- `LoginFail` - When login fails
- `AdminLogin` - For admin login
- `Logout` - When client logs out

## See Also

- [WHMCS Hooks Documentation](https://docs.whmcs.com/Hooks)
- [Security Configuration](../whmcs-security-setup.md)