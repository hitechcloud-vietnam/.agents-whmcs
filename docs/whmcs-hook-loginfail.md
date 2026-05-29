# WHMCS LoginFail Hook Reference

## Overview

The `LoginFail` hook fires when a client login attempt fails in WHMCS. This hook triggers after failed authentication and is essential for security monitoring, brute-force protection, and fraud detection.

## Hook Parameters

| Parameter | Type | Description |
|-----------|------|-------------|
| `email` | string | Email used in login attempt |
| `password` | string | Password used (may be masked) |
| `ip` | string | IP address of login attempt |
| `user_agent` | string | Browser user agent |

## Example Implementation

```php
<?php
add_hook('LoginFail', 1, function(array $params) {
    // Log the failed attempt
    logActivity("Failed login attempt for: {$params['email']} from IP: {$params['ip']}");
    
    return $params;
});
```

## Brute Force Protection

```php
<?php
add_hook('LoginFail', 1, function(array $params) {
    $ip = $params['ip'];
    $email = $params['email'];
    
    // 1. Track failed attempts in custom table
    $attempts = (int)db_escape($ip . '_' . date('Y-m-d-H'));
    insert_query('tbl_login_attempts', [
        'ip' => $ip,
        'email_attempted' => $email,
        'user_agent' => $params['user_agent'],
        'attempted_at' => date('Y-m-d H:i:s')
    ]);
    
    // 2. Count recent failures from this IP
    $recentFailures = full_query_value("
        SELECT COUNT(*) FROM tbl_login_attempts 
        WHERE ip = '" . db_escape($ip) . "' 
        AND attempted_at > DATE_SUB(NOW(), INTERVAL 1 HOUR)
    ");
    
    // 3. Block IP if too many failures
    if ($recentFailures >= 10) {
        insert_query('tbl_ip_bans', [
            'ip' => $ip,
            'reason' => 'Brute force protection',
            'banned_until' => date('Y-m-d H:i:s', strtotime('+1 hour')),
            'banned_at' => date('Y-m-d H:i:s')
        ]);
        
        logActivity("IP {$ip} temporarily banned for brute force attempts");
    }
    
    // 4. Notify if account has many failures
    $clientId = getClientIdByEmail($email);
    if ($clientId) {
        $clientFailures = full_query_value("
            SELECT COUNT(*) FROM tbl_login_attempts 
            WHERE email_attempted = '" . db_escape($email) . "' 
            AND attempted_at > DATE_SUB(NOW(), INTERVAL 24 HOUR)
        ");
        
        if ($clientFailures >= 5) {
            sendSecurityAlertForAccount($clientId, 'multiple_failed_logins', [
                'ip' => $ip,
                'attempts' => $clientFailures
            ]);
        }
    }
    
    return $params;
});
```

## Account Lockout

```php
<?php
add_hook('LoginFail', 1, function(array $params) {
    $email = $params['email'];
    $clientId = getClientIdByEmail($email);
    
    if (!$clientId) {
        return $params;
    }
    
    // 1. Increment login attempts for this account
    $currentAttempts = getClientLoginAttempts($clientId);
    $newAttempts = $currentAttempts + 1;
    
    if ($newAttempts >= 5) {
        // Lock account
        update_query('tblclients', [
            'status' => 'Locked',
            'locked_reason' => 'excessive_login_attempts',
            'locked_at' => date('Y-m-d H:i:s')
        ], ['id' => $clientId]);
        
        // Notify client
        $client = getClientsDetails($clientId);
        sendTemplatedEmail('Account Locked', $client['email'], [
            'ip' => $params['ip'],
            'reason' => 'Too many failed login attempts'
        ]);
        
        logActivity("Account {$clientId} locked due to failed login attempts");
    } else {
        update_query('tblclients', [
            'loginattempts' => $newAttempts
        ], ['id' => $clientId]);
    }
    
    // 2. Reset attempts after timeout (e.g., 30 minutes)
    // This would typically be done in a cron job
    
    return $params;
});
```

## Fraud Detection

```php
<?php
add_hook('LoginFail', 1, function(array $params) {
    // 1. Record for fraud analysis
    insert_query('tbl_fraud_analysis', [
        'event_type' => 'login_failed',
        'email' => $params['email'],
        'ip' => $params['ip'],
        'user_agent' => $params['user_agent'],
        'timestamp' => date('Y-m-d H:i:s'),
        'data' => json_encode([
            'referer' => $_SERVER['HTTP_REFERER'] ?? 'direct',
            'geo_country' => getIPCountry($params['ip'])
        ])
    ]);
    
    // 2. Check if IP is in known malicious list
    if (isKnownMaliciousIP($params['ip'])) {
        logActivity("Login attempt from known malicious IP: {$params['ip']}");
        alertSecurityTeam("Malicious IP login attempt");
    }
    
    // 3. Check for credential stuffing patterns
    $similarAttempts = getSimilarFailedAttempts($params['ip']);
    if (count($similarAttempts) > 20) {
        // Likely credential stuffing
        addIPToBlocklist($params['ip'], 'credential_stuffing');
    }
    
    // 4. Geo-anomaly detection
    $clientId = getClientIdByEmail($params['email']);
    if ($clientId) {
        $lastLogin = getLastLoginLocation($clientId);
        $currentCountry = getIPCountry($params['ip']);
        
        if ($lastLogin && $lastLogin['country'] !== $currentCountry) {
            $timeSinceLastLogin = time() - strtotime($lastLogin['timestamp']);
            
            // If last login was recently from different country
            if ($timeSinceLastLogin < 3600) { // Less than 1 hour
                flagAccountForReview($clientId, 'impossible_travel');
            }
        }
    }
    
    return $params;
});
```

## Use Cases

- **Brute Force Protection**: Block after multiple failures
- **Account Lockout**: Temporarily lock accounts
- **Fraud Detection**: Detect credential stuffing
- **Security Alerts**: Notify users of suspicious activity
- **Logging**: Comprehensive audit trail

## Notes

- Runs after every failed authentication
- Use with `Login` for complete login tracking
- IP-based blocking requires careful implementation
- Consider implementing CAPTCHA after failures

## Related Hooks

- `Login` - When login succeeds
- `AdminLogin` - For admin login
- `TwoFactorVerification` - For 2FA failures

## See Also

- [WHMCS Hooks Documentation](https://docs.whmcs.com/Hooks)
- [Security Configuration](../whmcs-security-setup.md)