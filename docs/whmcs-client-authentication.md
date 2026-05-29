# WHMCS Client Authentication

## Overview

Client authentication in WHMCS encompasses all methods used to verify client identity when accessing their account. This includes password-based authentication, two-factor authentication, single sign-on, and session management.

## Authentication Methods

### Standard Authentication

```php
// Username + Password
[
    'username' => 'client@example.com',
    'password' => 'SecurePassword123!',
    'remember_me' => false
]
```

### SSO Integration

```php
// Single Sign-On methods
[
    'sso_providers' => [
        'google' => ['enabled' => true, 'client_id' => 'xxx'],
        'microsoft' => ['enabled' => true, 'client_id' => 'xxx'],
        'facebook' => ['enabled' => false]
    ]
]
```

## Authentication Configuration

### Accessing Settings

**Configuration > Security > Authentication**

### Basic Settings

```php
[
    'allow_remember_me' => true,
    'remember_duration' => 30,           // days
    'session_timeout' => 120,           // minutes
    'max_login_attempts' => 5,
    'lockout_duration' => 30,            // minutes
    'require_email_verification' => true
]
```

## Login Process

### Standard Login Flow

```php
// Login attempt
[
    'step_1' => 'validate_credentials',
    'step_2' => 'check_account_status',
    'step_3' => 'verify_2fa_if_enabled',
    'step_4' => 'create_session',
    'step_5' => 'set_cookie',
    'step_6' => 'redirect_to_dashboard'
]
```

### Login Validation

```php
// Validate login attempt
[
    'check_credentials' => true,
    'check_account_active' => true,
    'check_not_suspended' => true,
    'check_email_verified' => true,
    'check_ip_not_blocked' => true
]
```

## Session Management

### Session Configuration

```php
// Session settings
[
    'session_lifetime' => 120,          // minutes
    'session_storage' => 'database',     // database, files
    'session_secure' => true,           // HTTPS only
    'session_httponly' => true,
    'session_samesite' => 'Lax',         // Strict, Lax, None
    'regenerate_id' => true             // Regenerate session ID on login
]
```

### Session Data Storage

```php
// Store session data
[
    'userid' => 123,
    'email' => 'client@example.com',
    'logged_in_at' => '2024-05-15 10:30:00',
    'last_activity' => '2024-05-15 10:45:00',
    'ip_address' => '192.168.1.1',
    'user_agent' => 'Mozilla/5.0...'
]
```

## Multi-Factor Authentication

### MFA Methods

```php
// Available MFA methods
[
    'totp' => [
        'enabled' => true,
        'app_name' => 'WHMCS'
    ],
    'email_code' => [
        'enabled' => true,
        'code_length' => 6,
        'expiry_minutes' => 10
    ],
    'sms_code' => [
        'enabled' => false,
        'provider' => 'twilio'
    ],
    'security_questions' => [
        'enabled' => false,
        'questions_required' => 3
    ]
]
```

### TOTP Setup

```php
// Time-based OTP
[
    'secret' => 'JBSWY3DPEHPK3PXP',     // Base32 encoded secret
    'issuer' => 'Your Company',
    'algorithm' => 'SHA1',
    'digits' => 6,
    'period' => 30
]

// Generate QR code
$otpauth = 'otpauth://totp/WHMCS:client@example.com?secret=JBSWY3DPEHPK3PXP&issuer=WHMCS';
```

## Single Sign-On (SSO)

### Social Login

```php
// OAuth providers
[
    'google' => [
        'enabled' => true,
        'client_id' => 'xxx',
        'client_secret' => 'xxx',
        'redirect_uri' => '/oauth/google/callback'
    ],
    'microsoft' => [
        'enabled' => true,
        'client_id' => 'xxx',
        'client_secret' => 'xxx'
    ]
]
```

### Link Social Account

```php
// Link SSO to existing account
[
    'action' => 'link_account',
    'provider' => 'google',
    'provider_user_id' => 'xxx',
    'userid' => 123,
    'email' => 'client@example.com'
]
```

## Login Security

### Brute Force Protection

```php
// Login attempt limits
[
    'max_attempts' => 5,
    'lockout_duration' => 30,           // minutes
    'attempt_window' => 15,             // minutes
    'ip_lockout' => true,
    'email_notification' => true,
    'notify_after_attempts' => 3
]
```

### IP Blocking

```php
// Block suspicious IPs
[
    'auto_block' => true,
    'block_after_attempts' => 10,
    'block_duration' => 24,             // hours
    'whitelist_ips' => ['192.168.1.0/24'],
    'blacklist_ips' => []
]
```

## Login Templates

### Custom Login Page

```smarty
<!-- login.tpl -->
<form method="post" action="dologin.php">
    <input type="email" name="email" placeholder="Email" required>
    <input type="password" name="password" placeholder="Password" required>
    <input type="checkbox" name="rememberme"> Remember me
    <button type="submit">Login</button>
</form>

<a href="/password-reset.php">Forgot Password?</a>

<div class="sso-options">
    <a href="/oauth/google">Login with Google</a>
</div>
```

## API Authentication

### API Access

```php
// API authentication
[
    'api_key' => 'your_api_key',
    'api_secret' => 'your_api_secret',
    'allowed_ips' => ['192.168.1.0/24'],
    'rate_limit' => 100                // requests per minute
]
```

### OAuth 2.0

```php
// OAuth for client access
[
    'grant_type' => 'authorization_code',
    'client_id' => 'xxx',
    'client_secret' => 'xxx',
    'redirect_uri' => 'https://app.example.com/callback',
    'scope' => ['profile', 'services', 'invoices']
]
```

## Session Security

### CSRF Protection

```php
// Cross-Site Request Forgery prevention
[
    'csrf_protection' => true,
    'token_name' => 'csrf_token',
    'token_expiry' => 7200              // seconds
]

// Generate token
$csrfToken = bin2hex(random_bytes(32));

// Validate token
if (!hash_equals($_SESSION['csrf_token'], $_POST['csrf_token'])) {
    // Invalid request
}
```

### XSS Protection

```php
// Cross-Site Scripting prevention
[
    'escape_output' => true,
    'content_security_policy' => true,
    'x_frame_options' => 'DENY',
    'x_content_type_options' => 'nosniff'
]
```

## Authentication Events

### Login Logging

```php
// Log authentication events
[
    'event' => 'login_success',
    'userid' => 123,
    'ip' => '192.168.1.1',
    'timestamp' => '2024-05-15 10:30:00',
    'user_agent' => 'Mozilla/5.0...'
]

// Failed login
[
    'event' => 'login_failed',
    'email' => 'client@example.com',
    'ip' => '192.168.1.1',
    'reason' => 'invalid_password',
    'timestamp' => '2024-05-15 10:30:00'
]
```

## Logout Process

```php
// User logout
[
    'action' => 'logout',
    'invalidate_session' => true,
    'clear_remember_token' => true,
    'redirect_to' => '/login.php'
]
```

## Hooks

```php
// Hook: UserLogin
add_hook('UserLogin', 1, function($vars) {
    // $vars['userid']
    // $vars['email']
    // Custom login actions
});

// Hook: UserLogout
add_hook('UserLogout', 1, function($vars) {
    // $vars['userid']
});

// Hook: FailedLoginAttempt
add_hook('FailedLoginAttempt', 1, function($vars) {
    // $vars['email']
    // $vars['ip']
    // Notify admin or block IP
});
```

## Best Practices

1. **HTTPS only**: Ensure all authentication over secure connection
2. **Strong passwords**: Enforce password requirements
3. **MFA**: Encourage or require two-factor authentication
4. **Session timeout**: Set appropriate session duration
5. **Monitor**: Track failed login attempts

## Related Documentation

- [Client Passwords](./whmcs-client-passwords.md)
- [Two-Factor Auth](./whmcs-two-factor-auth.md)
- [Client Portal](./whmcs-client-portal.md)
- [Security Settings](./whmcs-security-settings.md)