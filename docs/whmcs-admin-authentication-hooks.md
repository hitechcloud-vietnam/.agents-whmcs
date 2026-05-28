# WHMCS Admin Authentication Hooks

**Version:** 8.x | **Updated:** 2026-05-29
**Related Skills:** `whmcs-hooks-development`, `whmcs-two-factor-auth`, `whmcs-security-hardening`

---

## Overview

Admin authentication hooks allow you to customize and extend the authentication process for WHMCS admin accounts. This includes custom login validation, two-factor authentication integration, session management, and access control.

---

## Authentication Hooks Overview

### Available Admin Auth Hooks

| Hook Name | Description | Parameters |
|-----------|-------------|------------|
| `AdminLogin` | After successful admin login | `adminid`, `username` |
| `AdminLogout` | After admin logout | `adminid` |
| `PreAdminLogin` | Before login validation | `username`, `password` |
| `LoginAdmin2FA` | Two-factor verification | `adminid`, `code`, `success` |
| `AdminAreaPage` | Admin page load | Various page variables |
| `AdminAreaHeaderOutput` | Add to admin header | - |

---

## Pre-Login Validation

### Custom Login Requirements

```php
<?php
/**
 * Pre-AdminLogin hook
 * Validate admin login before processing
 */

add_hook('PreAdminLogin', 1, function($vars) {
    $username = $vars['username'];
    $password = $vars['password'];

    // Example: Block login from specific IP ranges
    $blockedRanges = [
        '192.168.1.',
        '10.0.0.',
    ];

    $clientIp = $_SERVER['REMOTE_ADDR'];

    foreach ($blockedRanges as $range) {
        if (strpos($clientIp, $range) === 0) {
            return [
                'error' => 'Login not allowed from this network',
            ];
        }
    }

    // Example: Require specific username pattern
    if (!preg_match('/^[a-zA-Z][a-zA-Z0-9_]{2,}$/', $username)) {
        return [
            'error' => 'Invalid username format',
        ];
    }

    // Example: Check login attempt limit
    $attempts = Capsule::table('mod_login_attempts')
        ->where('ip', $clientIp)
        ->where('created_at', '>', date('Y-m-d H:i:s', strtotime('-15 minutes')))
        ->count();

    if ($attempts >= 5) {
        return [
            'error' => 'Too many login attempts. Please try again later.',
        ];
    }

    // Return $vars to continue with default validation
    return $vars;
});
```

### Rate Limiting

```php
<?php
/**
 * Admin login rate limiting
 */

add_hook('PreAdminLogin', 1, function($vars) {
    $ip = $_SERVER['REMOTE_ADDR'];
    $table = 'mod_admin_login_attempts';

    // Get recent attempts
    $recentAttempts = Capsule::table($table)
        ->where('ip', $ip)
        ->where('attempted_at', '>', date('Y-m-d H:i:s', strtotime('-5 minutes')))
        ->get();

    $failedCount = count(array_filter($recentAttempts, function($a) {
        return $a->success == 0;
    }));

    // Lock out after 3 failed attempts
    if ($failedCount >= 3) {
        $lastAttempt = end($recentAttempts);
        $lockoutUntil = date('Y-m-d H:i:s',
            strtotime($lastAttempt->attempted_at) + 900 // 15 min
        );

        if (date('Y-m-d H:i:s') < $lockoutUntil) {
            return [
                'error' => "Account locked. Try again after " .
                    date('H:i', strtotime($lockoutUntil)),
            ];
        }
    }

    return $vars;
});

add_hook('AdminLogin', 1, function($vars) {
    // Log successful login
    Capsule::table('mod_admin_login_attempts')->insert([
        'admin_id' => $vars['adminid'],
        'ip' => $_SERVER['REMOTE_ADDR'],
        'success' => 1,
        'attempted_at' => date('Y-m-d H:i:s'),
    ]);
});
```

---

## Two-Factor Authentication

### Custom 2FA Integration

```php
<?php
/**
 * Custom Two-Factor Authentication hook
 * Integrate with external 2FA provider
 */

add_hook('LoginAdmin2FA', 1, function($vars) {
    $adminId = $vars['adminid'];
    $code = $vars['code'];

    // Get admin 2FA settings
    $admin = Capsule::table('tbladmins')
        ->where('id', $adminId)
        ->first();

    // Check if 2FA is enabled for this admin
    $twoFactorSettings = Capsule::table('mod_admin_2fa')
        ->where('admin_id', $adminId)
        ->first();

    if (!$twoFactorSettings || !$twoFactorSettings->enabled) {
        return ['success' => true];
    }

    // Verify with external provider (e.g., Duo, Authy)
    $result = verifyExternal2FA(
        $twoFactorSettings->provider,
        $twoFactorSettings->config,
        $code,
        $admin->username
    );

    if (!$result['valid']) {
        logActivity("Admin 2FA failed for: " . $admin->username);

        return [
            'success' => false,
            'error' => 'Invalid verification code',
        ];
    }

    return ['success' => true];
});

/**
 * External 2FA verification helper
 */
function verifyExternal2FA(string $provider, array $config, string $code, string $username): array
{
    switch ($provider) {
        case 'duo':
            return verifyDuo($config, $code, $username);

        case 'authy':
            return verifyAuthy($config, $code);

        case 'totp':
            return verifyTotp($config['secret'], $code);

        default:
            return ['valid' => false, 'error' => 'Unknown provider'];
    }
}
```

### Enforcing 2FA

```php
<?php
/**
 * Enforce 2FA for all admins
 */

add_hook('PreAdminLogin', 1, function($vars) {
    $username = $vars['username'];

    // Get admin details
    $admin = Capsule::table('tbladmins')
        ->where('username', $username)
        ->first();

    if (!$admin) {
        return $vars;
    }

    // Check 2FA requirement
    $require2fa = Capsule::table('mod_admin_security')
        ->where('setting', 'require_2fa')
        ->value('value');

    if ($require2fa) {
        $has2fa = Capsule::table('mod_admin_2fa')
            ->where('admin_id', $admin->id)
            ->where('enabled', 1)
            ->exists();

        if (!$has2fa) {
            return [
                'error' => 'Two-factor authentication is required. Please configure 2FA in your profile.',
                'force_2fa_setup' => true,
            ];
        }
    }

    return $vars;
});
```

---

## Session Management

### Custom Session Handling

```php
<?php
/**
 * Custom admin session validation
 */

add_hook('AdminAreaPage', 1, function($vars) {
    $adminId = $_SESSION['adminid'] ?? null;

    if (!$adminId) {
        return;
    }

    // Check for concurrent session limit
    $maxSessions = Capsule::table('mod_admin_security')
        ->where('setting', 'max_concurrent_sessions')
        ->value('value') ?? 1;

    $activeSessions = Capsule::table('mod_admin_sessions')
        ->where('admin_id', $adminId)
        ->where('last_activity', '>', date('Y-m-d H:i:s', strtotime('-30 minutes')))
        ->count();

    if ($activeSessions > $maxSessions) {
        // Terminate oldest sessions
        $oldSessions = Capsule::table('mod_admin_sessions')
            ->where('admin_id', $adminId)
            ->orderBy('last_activity', 'asc')
            ->limit($activeSessions - $maxSessions)
            ->get();

        foreach ($oldSessions as $session) {
            Capsule::table('mod_admin_sessions')
                ->where('id', $session->id)
                ->delete();

            // Optionally notify user
            logActivity("Terminated concurrent admin session: " . $session->session_id);
        }
    }

    // Update session activity
    Capsule::table('mod_admin_sessions')
        ->where('session_id', session_id())
        ->update(['last_activity' => date('Y-m-d H:i:s')]);
});

/**
 * Update session on each page load
 */
add_hook('AdminAreaPage', 50, function($vars) {
    $sessionId = session_id();
    $adminId = $_SESSION['adminid'] ?? null;

    if ($adminId && $sessionId) {
        Capsule::table('mod_admin_sessions')
            ->updateOrInsert(
                ['session_id' => $sessionId],
                [
                    'admin_id' => $adminId,
                    'ip_address' => $_SERVER['REMOTE_ADDR'],
                    'user_agent' => $_SERVER['HTTP_USER_AGENT'] ?? '',
                    'last_activity' => date('Y-m-d H:i:s'),
                ]
            );
    }
});
```

### Session Expiry

```php
<?php
/**
 * Custom session timeout handling
 */

add_hook('AdminAreaPage', 1, function($vars) {
    $lastActivity = $_SESSION['last_admin_activity'] ?? 0;
    $timeout = 3600; // 1 hour in seconds
    $warnTime = 300; // 5 minutes before timeout

    $idleTime = time() - $lastActivity;

    // Check if session expired
    if ($idleTime > $timeout) {
        // Force logout
        unset($_SESSION['adminid']);
        unset($_SESSION['last_admin_activity']);

        header('Location: ' . WHMCS\Config\Setting::getValue('SystemURL') . 'admin/login.php?expired=1');
        exit;
    }

    // Warn user before expiry
    if ($idleTime > ($timeout - $warnTime)) {
        $_SESSION['admin_session_warning'] = true;
    }

    // Update last activity
    $_SESSION['last_admin_activity'] = time();
});

/**
 * JavaScript for session warning
 */
add_hook('AdminAreaHeaderOutput', 1, function($vars) {
    if ($_SESSION['admin_session_warning'] ?? false) {
        return '<script>
            setTimeout(function() {
                if (confirm("Your session will expire soon. Click OK to extend.")) {
                    fetch("' . $vars['baseurl'] . 'admin/session-extend.php");
                    location.reload();
                }
            }, 1000);
        </script>';
    }
});
```

---

## Access Control

### Role-Based Access Hooks

```php
<?php
/**
 * Custom permission checking
 */

add_hook('AdminAreaPage', 1, function($vars) {
    $adminId = $_SESSION['adminid'] ?? 0;
    $currentPage = basename($_SERVER['PHP_SELF'], '.php');

    // Get admin permissions
    $admin = Capsule::table('tbladmins')
        ->where('id', $adminId)
        ->first();

    // Define page permissions
    $pagePermissions = [
        'reports' => ['role_1', 'role_2'],        // Only certain roles
        'users' => ['role_1'],                    // Only admin role
        'bulk-mail' => ['role_1', 'role_2'],
        'configcustomfields' => ['role_1'],
    ];

    if (isset($pagePermissions[$currentPage])) {
        $adminRole = Capsule::table('mod_admin_roles')
            ->where('admin_id', $adminId)
            ->value('role');

        if (!in_array($adminRole, $pagePermissions[$currentPage])) {
            // Redirect to dashboard with error
            header('Location: ' . $_SESSION['adminurl'] . '/index.php?accessdenied=1');
            exit;
        }
    }
});

/**
 * Custom capability check
 */
function checkAdminCapability(int $adminId, string $capability): bool
{
    $permissions = Capsule::table('mod_admin_capabilities')
        ->where('admin_id', $adminId)
        ->pluck('capability')
        ->toArray();

    $rolePermissions = Capsule::table('tbladminroles')
        ->where('id', Capsule::table('tbladmins')
            ->where('id', $adminId)
            ->value('roleid')
        )
        ->value('permissions');

    $rolePermissions = json_decode($rolePermissions, true) ?? [];

    return in_array($capability, $permissions)
        || in_array($capability, $rolePermissions);
}
```

---

## IP Allowlisting

```php
<?php
/**
 * Admin IP allowlisting
 */

add_hook('PreAdminLogin', 1, function($vars) {
    $clientIp = $_SERVER['REMOTE_ADDR'];
    $username = $vars['username'];

    // Get admin details
    $admin = Capsule::table('tbladmins')
        ->where('username', $username)
        ->first();

    if (!$admin) {
        return $vars;
    }

    // Check if IP restriction is enabled for this admin
    $allowedIps = Capsule::table('mod_admin_ip_whitelist')
        ->where('admin_id', $admin->id)
        ->pluck('ip_address')
        ->toArray();

    if (empty($allowedIps)) {
        return $vars; // No restrictions
    }

    // Check if current IP is allowed
    foreach ($allowedIps as $allowedIp) {
        if ($clientIp === $allowedIp) {
            return $vars; // IP allowed
        }

        // CIDR notation support
        if (strpos($allowedIp, '/') !== false) {
            if (ip_in_cidr($clientIp, $allowedIp)) {
                return $vars;
            }
        }
    }

    // Log denied access
    logActivity("Admin login denied - IP not whitelisted: " . $username . " from " . $clientIp);

    return [
        'error' => 'Access denied from your current location',
    ];
});

/**
 * Check if IP is in CIDR range
 */
function ip_in_cidr(string $ip, string $cidr): bool
{
    list($subnet, $mask) = explode('/', $cidr);
    $maskBits = -1 << (32 - $mask);

    return (ip2long($ip) & $maskBits) === (ip2long($subnet) & $maskBits);
}
```

---

## Login Notifications

### Admin Login Notifications

```php
<?php
/**
 * Notify on admin login
 */

add_hook('AdminLogin', 1, function($vars) {
    $adminId = $vars['adminid'];
    $admin = Capsule::table('tbladmins')
        ->where('id', $adminId)
        ->first();

    // Send email notification
    if (Capsule::table('mod_admin_security')
        ->where('setting', 'login_notify')
        ->value('value')
    ) {
        send_email([
            'to' => $admin->email,
            'subject' => 'Admin Login Alert',
            'message' => "
                An admin login was detected on your WHMCS installation.

                Details:
                - Username: {$vars['username']}
                - IP Address: {$_SERVER['REMOTE_ADDR']}
                - Time: " . date('Y-m-d H:i:s') . "
                - User Agent: {$_SERVER['HTTP_USER_AGENT']}

                If this wasn't you, please secure your account immediately.
            ",
        ]);
    }

    // Log to security audit
    Capsule::table('mod_admin_audit_log')->insert([
        'admin_id' => $adminId,
        'action' => 'login',
        'ip_address' => $_SERVER['REMOTE_ADDR'],
        'user_agent' => $_SERVER['HTTP_USER_AGENT'] ?? '',
        'created_at' => date('Y-m-d H:i:s'),
    ]);
});

/**
 * Notify on failed login attempt
 */
add_hook('PreAdminLogin', 100, function($vars) {
    // This hook is called even on failed logins
    if (isset($vars['error'])) {
        $username = $vars['username'];

        // Log failed attempt
        Capsule::table('mod_admin_login_attempts')->insert([
            'username' => $username,
            'ip_address' => $_SERVER['REMOTE_ADDR'],
            'attempted_at' => date('Y-m-d H:i:s'),
            'reason' => $vars['error'],
            'success' => 0,
        ]);

        // Check for suspicious activity
        $recentFailures = Capsule::table('mod_admin_login_attempts')
            ->where('ip_address', $_SERVER['REMOTE_ADDR'])
            ->where('success', 0)
            ->where('attempted_at', '>', date('Y-m-d H:i:s', strtotime('-1 hour')))
            ->count();

        if ($recentFailures >= 5) {
            // Send alert
            logActivity("Multiple failed admin login attempts from: " . $_SERVER['REMOTE_ADDR']);
        }
    }
});
```

---

## Best Practices

1. **Always validate input** - Sanitize all user data
2. **Log all attempts** - Track successful and failed logins
3. **Implement rate limiting** - Prevent brute force attacks
4. **Use secure storage** - Encrypt sensitive data
5. **Return errors carefully** - Don't expose system details
6. **Consider performance** - Keep hooks efficient
7. **Test thoroughly** - Test authentication edge cases

---

## Related Documentation

- [Hook System Reference](whmcs-hook-system.md)
- [Hooks Reference](hooks-reference.md)
- [Two-Factor Authentication](../skills/whmcs-two-factor-auth.md)
- [Security Best Practices](security-best-practices.md)
