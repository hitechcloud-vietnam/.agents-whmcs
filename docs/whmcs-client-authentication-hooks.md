# WHMCS Client Authentication Hooks

**Version:** 8.x | **Updated:** 2026-05-29
**Related Skills:** `whmcs-hooks-development`, `whmcs-two-factor-auth`, `whmcs-sso-integration`, `whmcs-security-hardening`

---

## Overview

Client authentication hooks allow you to customize and secure the customer login experience in WHMCS. This includes login validation, registration requirements, two-factor authentication, SSO integration, and session management.

---

## Client Auth Hooks Overview

### Available Client Auth Hooks

| Hook Name | Description | Parameters |
|-----------|-------------|------------|
| `PreLoginClient` | Before login validation | `username` |
| `LoginVerify` | After login verification | `username`, `password`, `result` |
| `ClientLogin` | After successful login | `userid` |
| `ClientLogout` | After logout | `userid` |
| `ClientChangePassword` | After password change | `userid`, `password` |
| `TwoFactorAuthentication` | Two-factor verification | `userid`, `code`, `success` |
| `ClientAdd` | After client registration | `userid`, `firstname`, `lastname`, `email` |
| `PreClientAdd` | Before registration | All registration fields |

---

## Login Validation

### Custom Pre-Login Checks

```php
<?php
/**
 * PreLoginClient hook
 * Validate login before processing
 */

add_hook('PreLoginClient', 1, function($vars) {
    $username = $vars['username'];
    $ip = $_SERVER['REMOTE_ADDR'];

    // Example: Check if account is locked
    $client = Capsule::table('mod_client_security')
        ->where('email', $username)
        ->orWhere('username', $username)
        ->first();

    if ($client && $client->locked_until) {
        if (strtotime($client->locked_until) > time()) {
            return [
                'error' => 'Account is temporarily locked. Please try again later.',
            ];
        }
    }

    // Example: IP-based blocking
    $blockedIps = Capsule::table('mod_ip_blacklist')
        ->pluck('ip_address')
        ->toArray();

    if (in_array($ip, $blockedIps)) {
        logActivity("Blocked login attempt from blacklisted IP: {$ip}");
        return [
            'error' => 'Access denied',
        ];
    }

    // Example: Check for suspicious activity
    $recentAttempts = Capsule::table('mod_login_attempts')
        ->where('ip', $ip)
        ->where('attempted_at', '>', date('Y-m-d H:i:s', strtotime('-1 hour')))
        ->count();

    if ($recentAttempts >= 10) {
        return [
            'error' => 'Too many login attempts. Please try again later.',
        ];
    }

    return $vars;
});
```

### Login Rate Limiting

```php
<?php
<?php
/**
 * Client login rate limiting
 */

add_hook('PreLoginClient', 1, function($vars) {
    $ip = $_SERVER['REMOTE_ADDR'];
    $username = $vars['username'];

    // Check if this IP is temporarily blocked
    $block = Capsule::table('mod_ip_blocks')
        ->where('ip_address', $ip)
        ->where('blocked_until', '>', date('Y-m-d H:i:s'))
        ->first();

    if ($block) {
        $minutes = ceil((strtotime($block->blocked_until) - time()) / 60);
        return [
            'error' => "Too many failed attempts. Try again in {$minutes} minutes.",
        ];
    }

    // Get recent failed attempts from this IP
    $failedAttempts = Capsule::table('mod_login_attempts')
        ->where('ip_address', $ip)
        ->where('success', 0)
        ->where('attempted_at', '>', date('Y-m-d H:i:s', strtotime('-15 minutes')))
        ->count();

    // After 5 failed attempts, block for 15 minutes
    if ($failedAttempts >= 5) {
        Capsule::table('mod_ip_blocks')->insert([
            'ip_address' => $ip,
            'blocked_until' => date('Y-m-d H:i:s', strtotime('+15 minutes')),
            'reason' => 'Too many failed login attempts',
        ]);

        return [
            'error' => 'Too many failed login attempts. Please try again in 15 minutes.',
        ];
    }

    return $vars;
});

/**
 * Log login attempts
 */
add_hook('LoginVerify', 1, function($vars) {
    $result = $vars['result'];

    Capsule::table('mod_login_attempts')->insert([
        'email' => $vars['username'],
        'ip_address' => $_SERVER['REMOTE_ADDR'],
        'user_agent' => $_SERVER['HTTP_USER_AGENT'] ?? '',
        'success' => $result['success'] ? 1 : 0,
        'attempted_at' => date('Y-m-d H:i:s'),
    ]);
});
```

---

## Registration Validation

### Pre-Registration Checks

```php
<?php
/**
 * PreClientAdd hook
 * Validate registration data
 */

add_hook('PreClientAdd', 1, function($vars) {
    // Validate custom fields
    if (empty($vars['customfield']['referral_code'])) {
        // Make referral code optional
    }

    // Check email domain restrictions
    $allowedDomains = Capsule::table('mod_email_restrictions')
        ->where('type', 'allowed')
        ->pluck('domain')
        ->toArray();

    $blockedDomains = Capsule::table('mod_email_restrictions')
        ->where('type', 'blocked')
        ->pluck('domain')
        ->toArray();

    $emailDomain = substr(strrchr($vars['email'], '@'), 1);

    if (!empty($blockedDomains) && in_array($emailDomain, $blockedDomains)) {
        return [
            'error' => 'This email domain is not allowed for registration.',
        ];
    }

    if (!empty($allowedDomains) && !in_array($emailDomain, $allowedDomains)) {
        return [
            'error' => 'Registration is limited to specific email domains.',
        ];
    }

    // Validate password strength
    $password = $vars['password'];
    if (strlen($password) < 8) {
        return [
            'error' => 'Password must be at least 8 characters long.',
        ];
    }

    if (!preg_match('/[A-Z]/', $password)) {
        return [
            'error' => 'Password must contain at least one uppercase letter.',
        ];
    }

    if (!preg_match('/[a-z]/', $password)) {
        return [
            'error' => 'Password must contain at least one lowercase letter.',
        ];
    }

    if (!preg_match('/[0-9]/', $password)) {
        return [
            'error' => 'Password must contain at least one number.',
        ];
    }

    // Validate country/state combination
    if ($vars['country'] === 'US' && !empty($vars['state'])) {
        $validStates = ['AL', 'AK', 'AZ', 'AR', 'CA', /* ... */];
        if (!in_array($vars['state'], $validStates)) {
            return [
                'error' => 'Invalid state selected.',
            ];
        }
    }

    return $vars;
});
```

### Registration Custom Fields

```php
<?php
/**
 * Add custom fields to registration
 */

add_hook('ClientAreaPageRegister', 1, function($vars) {
    // Add referral code field if needed
    return [
        'show_referral_field' => true,
        'referral_field_label' => 'Referral Code (Optional)',
        'referral_field_placeholder' => 'Enter referral code',
    ];
});
```

---

## Two-Factor Authentication

### Custom Client 2FA

```php
<?php
/**
 * Client Two-Factor Authentication hook
 */

add_hook('TwoFactorAuthentication', 1, function($vars) {
    $userId = $vars['userid'];
    $code = $vars['code'];

    // Check if 2FA is enabled for this client
    $twoFactor = Capsule::table('mod_client_2fa')
        ->where('user_id', $userId)
        ->first();

    if (!$twoFactor || !$twoFactor->enabled) {
        return ['success' => true]; // 2FA not enabled
    }

    // Verify code based on method
    switch ($twoFactor->method) {
        case 'totp':
            $result = verifyTotpCode($twoFactor->secret, $code);
            break;

        case 'email':
            $result = verifyEmailCode($userId, $code);
            break;

        case 'sms':
            $result = verifySmsCode($twoFactor->phone, $code);
            break;

        default:
            $result = false;
    }

    if (!$result) {
        logActivity("Client 2FA failed for user ID: {$userId}");

        return [
            'success' => false,
            'error' => 'Invalid verification code',
        ];
    }

    return ['success' => true];
});

/**
 * Verify TOTP code
 */
function verifyTotpCode(string $secret, string $code): bool
{
    // Using OTP library
    $totp = new \OTPHP\TOTP($secret);
    return $totp->verify($code, time(), 1);
}

/**
 * Verify email code
 */
function verifyEmailCode(int $userId, string $code): bool
{
    $verification = Capsule::table('mod_email_codes')
        ->where('user_id', $userId)
        ->where('code', hash('sha256', $code))
        ->where('expires_at', '>', date('Y-m-d H:i:s'))
        ->first();

    if ($verification) {
        // Delete used code
        Capsule::table('mod_email_codes')
            ->where('id', $verification->id)
            ->delete();

        return true;
    }

    return false;
}
```

### Enforcing 2FA

```php
<?php
/**
 * Force 2FA for certain clients
 */

add_hook('PreLoginClient', 1, function($vars) {
    $username = $vars['username'];

    $client = Capsule::table('tblclients')
        ->where('email', $username)
        ->orWhere('username', $username)
        ->first();

    if (!$client) {
        return $vars;
    }

    // Check if 2FA is required for this client
    $require2fa = Capsule::table('mod_client_security')
        ->where('user_id', $client->id)
        ->where('setting', 'require_2fa')
        ->value('value');

    if ($require2fa) {
        $has2fa = Capsule::table('mod_client_2fa')
            ->where('user_id', $client->id)
            ->where('enabled', 1)
            ->exists();

        if (!$has2fa) {
            return [
                'error' => 'Two-factor authentication is required. Please set it up in your account security settings.',
                'require_2fa_setup' => true,
                'setup_url' => 'security.php?action=2fa',
            ];
        }
    }

    return $vars;
});
```

---

## SSO Integration

### Single Sign-On Hooks

```php
<?php
/**
 * SSO token validation
 */

add_hook('PreLoginClient', 1, function($vars) {
    // Check for SSO token
    $ssoToken = $_GET['sso_token'] ?? $_COOKIE['sso_token'] ?? null;

    if (!$ssoToken) {
        return $vars;
    }

    // Validate SSO token
    $tokenData = Capsule::table('mod_sso_tokens')
        ->where('token', hash('sha256', $ssoToken))
        ->where('expires_at', '>', date('Y-m-d H:i:s'))
        ->where('used', 0)
        ->first();

    if (!$tokenData) {
        return $vars; // Fall through to normal login
    }

    // Mark token as used
    Capsule::table('mod_sso_tokens')
        ->where('id', $tokenData->id)
        ->update(['used' => 1, 'used_at' => date('Y-m-d H:i:s')]);

    // Get client
    $client = Capsule::table('tblclients')
        ->where('id', $tokenData->user_id)
        ->first();

    if ($client) {
        // Auto-login the client
        $_SESSION['uid'] = $client->id;
        $_SESSION['upw'] = \Illuminate\Support\Facades\Crypt::encrypt($client->password);
        $_SESSION['userid'] = $client->id;

        header('Location: clientarea.php');
        exit;
    }

    return $vars;
});

/**
 * Generate SSO link for external site
 */
function generateClientSsoLink(int $clientId, string $returnUrl): string
{
    $token = bin2hex(random_bytes(32));
    $expiresAt = date('Y-m-d H:i:s', strtotime('+5 minutes'));

    Capsule::table('mod_sso_tokens')->insert([
        'user_id' => $clientId,
        'token' => hash('sha256', $token),
        'return_url' => $returnUrl,
        'ip_address' => $_SERVER['REMOTE_ADDR'],
        'expires_at' => $expiresAt,
        'created_at' => date('Y-m-d H:i:s'),
    ]);

    $whmcsUrl = Capsule::table('tblconfiguration')
        ->where('setting', 'SystemURL')
        ->value('value');

    return $whmcsUrl . 'dologin.php?sso=' . $token;
}
```

---

## Session Management

### Client Session Handling

```php
<?php
/**
 * Custom client session validation
 */

add_hook('ClientAreaPage', 1, function($vars) {
    $userId = $_SESSION['uid'] ?? null;

    if (!$userId) {
        return;
    }

    // Check for concurrent session limit
    $maxSessions = Capsule::table('mod_client_security')
        ->where('setting', 'max_concurrent_sessions')
        ->value('value') ?? 3;

    $activeSessions = Capsule::table('mod_client_sessions')
        ->where('user_id', $userId)
        ->where('last_activity', '>', date('Y-m-d H:i:s', strtotime('-30 minutes')))
        ->count();

    if ($activeSessions > $maxSessions) {
        // Keep current session, mark others for cleanup
        Capsule::table('mod_client_sessions')
            ->where('user_id', $userId)
            ->where('session_id', '!=', session_id())
            ->where('last_activity', '<', date('Y-m-d H:i:s', strtotime('-30 minutes')))
            ->delete();
    }

    // Update current session
    Capsule::table('mod_client_sessions')
        ->updateOrInsert(
            ['session_id' => session_id()],
            [
                'user_id' => $userId,
                'ip_address' => $_SERVER['REMOTE_ADDR'],
                'user_agent' => $_SERVER['HTTP_USER_AGENT'] ?? '',
                'last_activity' => date('Y-m-d H:i:s'),
            ]
        );
});

/**
 * Session timeout handling
 */
add_hook('ClientAreaPage', 1, function($vars) {
    $lastActivity = $_SESSION['last_client_activity'] ?? 0;
    $timeout = 7200; // 2 hours
    $warnTime = 300; // 5 minutes before timeout

    $idleTime = time() - $lastActivity;

    if ($idleTime > $timeout) {
        // Force logout
        session_destroy();
        header('Location: clientarea.php?loggedout=session');
        exit;
    }

    if ($idleTime > ($timeout - $warnTime)) {
        $_SESSION['session_warning'] = true;
    }

    $_SESSION['last_client_activity'] = time();
});
```

---

## Login Notifications

### Client Login Alerts

```php
<?php
/**
 * Notify client of new login
 */

add_hook('ClientLogin', 1, function($vars) {
    $userId = $vars['userid'];

    $client = Capsule::table('tblclients')
        ->where('id', $userId)
        ->first();

    // Check if login notification is enabled
    $notifyLogin = Capsule::table('mod_client_security')
        ->where('user_id', $userId)
        ->where('setting', 'login_notification')
        ->value('value');

    if ($notifyLogin) {
        send_email([
            'type' => 'general',
            'id' => 0,
            'merge_fields' => [
                'client_first_name' => $client->firstname,
                'client_full_name' => $client->firstname . ' ' . $client->lastname,
                'login_ip' => $_SERVER['REMOTE_ADDR'],
                'login_time' => date('Y-m-d H:i:s'),
                'login_location' => getIpLocation($_SERVER['REMOTE_ADDR']),
            ],
        ], $client->email, 'Login Notification Template');
    }

    // Log to security audit
    Capsule::table('mod_client_audit_log')->insert([
        'user_id' => $userId,
        'action' => 'login',
        'ip_address' => $_SERVER['REMOTE_ADDR'],
        'user_agent' => $_SERVER['HTTP_USER_AGENT'] ?? '',
        'created_at' => date('Y-m-d H:i:s'),
    ]);
});

/**
 * Failed login notification
 */
add_hook('PreLoginClient', 100, function($vars) {
    if (isset($vars['error'])) {
        Capsule::table('mod_client_audit_log')->insert([
            'action' => 'login_failed',
            'email' => $vars['username'],
            'ip_address' => $_SERVER['REMOTE_ADDR'],
            'reason' => $vars['error'],
            'created_at' => date('Y-m-d H:i:s'),
        ]);

        // Check for brute force
        $recentFailures = Capsule::table('mod_client_audit_log')
            ->where('ip_address', $_SERVER['REMOTE_ADDR'])
            ->where('action', 'login_failed')
            ->where('created_at', '>', date('Y-m-d H:i:s', strtotime('-1 hour')))
            ->count();

        if ($recentFailures >= 5) {
            // Could send alert or block IP
            logActivity("Multiple failed login attempts from: " . $_SERVER['REMOTE_ADDR']);
        }
    }
});
```

---

## Best Practices

1. **Secure session handling** - Use secure session cookies
2. **Implement rate limiting** - Prevent brute force attacks
3. **Log all attempts** - Track login success and failures
4. **Send notifications** - Alert users of suspicious activity
5. **Enforce strong passwords** - Validate password strength
6. **Use HTTPS** - Always transmit credentials securely
7. **Clean up sessions** - Remove expired sessions regularly

---

## Related Documentation

- [Hook System Reference](whmcs-hook-system.md)
- [Hooks Reference](hooks-reference.md)
- [SSO Integration](../skills/whmcs-sso-integration.md)
- [Security Best Practices](security-best-practices.md)
