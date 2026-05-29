# WHMCS Two-Factor Authentication

## Overview

Two-factor authentication (2FA) in WHMCS adds an extra layer of security to client logins by requiring a second form of verification beyond the password. WHMCS supports multiple 2FA methods including TOTP apps, email codes, and SMS verification.

## Enabling 2FA

### Admin Configuration

**Configuration > Security > Two-Factor Authentication**

### Basic Settings

```php
[
    'enable_2fa' => true,
    'require_2fa' => false,            // Optional by default
    'allow_user_opt_out' => true,
    'require_admin_2fa' => true,
    'remember_device_days' => 30
]
```

## 2FA Methods

### Available Methods

| Method | Description | Security Level |
|--------|-------------|----------------|
| TOTP (Authenticator App) | Time-based one-time passwords | High |
| Email Code | Code sent to registered email | Medium |
| SMS Code | Code sent via text message | Medium |
| Backup Codes | Single-use backup codes | Low |

### Method Configuration

```php
// TOTP Configuration
[
    'method' => 'totp',
    'enabled' => true,
    'issuer' => 'Your Company',
    'algorithm' => 'SHA1',
    'digits' => 6,
    'period' => 30
]

// Email Code Configuration
[
    'method' => 'email',
    'enabled' => true,
    'code_length' => 6,
    'expiry_minutes' => 10,
    'max_attempts' => 3
]

// SMS Configuration
[
    'method' => 'sms',
    'enabled' => true,
    'provider' => 'twilio',
    'code_length' => 6,
    'expiry_minutes' => 5,
    'max_attempts' => 3
]
```

## TOTP Setup

### Setup Process

**Client Area > Security > Two-Factor Authentication > Enable**

```php
// Step 1: Generate secret
$secret = generateTOTPSecret();  // 20 bytes, base32 encoded

// Step 2: Generate QR code
$otpauth = "otpauth://totp/{$company}:{$email}?secret={$secret}&issuer={$company}&algorithm=SHA1&digits=6&period=30";

// Step 3: Display QR code
// User scans with authenticator app

// Step 4: Verify first code
$userCode = $_POST['code'];
if (verifyTOTP($secret, $userCode)) {
    // Enable 2FA
    save2FAConfig($userid, 'totp', $secret);
}
```

### Authenticator Apps

Compatible apps:
- Google Authenticator (iOS/Android)
- Microsoft Authenticator
- Authy
- 1Password
- LastPass Authenticator

### Backup Code Generation

```php
// Generate backup codes
[
    'count' => 10,
    'length' => 8,
    'format' => 'alphanumeric'
]

// Example backup codes
// XKCD-7842
// PQRS-3156
// MNOP-9876
```

## User 2FA Experience

### Login Flow with 2FA

```php
// Login with 2FA enabled
[
    'step_1' => 'Enter email and password',
    'step_2' => 'Password validated successfully',
    'step_3' => 'Enter 2FA code',
    'step_4' => '2FA code validated',
    'step_5' => 'Login complete'
]
```

### 2FA Input Screen

```html
<!-- 2FA verification screen -->
<form method="post">
    <p>Enter the 6-digit code from your authenticator app</p>
    <input type="text" name="code" maxlength="6" placeholder="000000" required>
    <button type="submit">Verify</button>
</form>

<a href="/resend-code.php">Didn't receive the code?</a>
```

## Admin 2FA Requirements

### Enforce Admin 2FA

```php
// Admin 2FA settings
[
    'require_admin_2fa' => true,
    'admin_roles' => [
        'full admin' => true,
        'billing admin' => true,
        'support admin' => false,
        'read-only' => false
    ],
    'allow_remember_device' => false,
    'force_re_auth' => false
]
```

### Admin Override

```php
// Allow admin to disable user 2FA
[
    'allow_disable_user_2fa' => true,
    'require_password_confirm' => true,
    'audit_log' => true
]
```

## 2FA Recovery

### Locked Out Recovery

```php
// User is locked out of account
[
    'recovery_options' => [
        'use_backup_codes',
        'contact_support',
        'email_verification_override'
    ]
]
```

### Admin Recovery

**Admin: Clients > Select Client > Security > Disable 2FA**

```php
// Admin disables user 2FA
[
    'action' => 'disable_2fa',
    'userid' => 123,
    'require_password' => true,
    'notify_user' => true,
    'audit_log' => true
]
```

### Backup Code Entry

```html
<!-- Backup code input -->
<form method="post">
    <p>Enter one of your backup codes</p>
    <input type="text" name="backup_code" placeholder="XXXX-XXXX" required>
    <button type="submit">Verify</button>
</form>
```

## 2FA Management

### View 2FA Status

**Admin: Clients > Select Client > Security**

```php
// Client 2FA status
[
    'userid' => 123,
    '2fa_enabled' => true,
    'method' => 'totp',
    'enabled_date' => '2024-03-15',
    'last_verified' => '2024-05-15 10:30:00',
    'backup_codes_remaining' => 8
]
```

### Disable 2FA

**Client: Account Settings > Security > Disable 2FA**

```php
// User disables own 2FA
[
    'action' => 'disable_2fa',
    'verify_password' => true,
    'confirm_action' => true
]
```

### Regenerate Backup Codes

```php
// Regenerate backup codes
[
    'action' => 'regenerate_backup_codes',
    'invalidate_old_codes' => true,
    'new_codes_count' => 10,
    'notify_user' => true
]
```

## API Functions

```php
// Enable 2FA for client
$result = localAPI('EnableTwoFactor', [
    'clientid' => 123,
    'method' => 'totp',
    'code' => '123456'
]);

// Disable 2FA
$result = localAPI('DisableTwoFactor', [
    'clientid' => 123
]);

// Check 2FA status
$result = localAPI('GetClientSecurity', [
    'clientid' => 123
]);
```

## Hooks

```php
// Hook: TwoFactorEnabled
add_hook('TwoFactorEnabled', 1, function($vars) {
    // $vars['userid']
    // $vars['method']
});

// Hook: TwoFactorVerify
add_hook('TwoFactorVerify', 1, function($vars) {
    // $vars['userid']
    // $vars['success']
    // Log verification attempts
});
```

## 2FA Reporting

### 2FA Statistics

**Reports > Security > Two-Factor Report**

```php
// Report data
[
    'period' => 'May 2024',
    'total_clients' => 1000,
    '2fa_enabled' => 450,
    '2fa_percentage' => 45,
    'method_breakdown' => [
        'totp' => 400,
        'email' => 40,
        'sms' => 10
    ],
    'successful_verifications' => 5000,
    'failed_attempts' => 50
]
```

## Best Practices

1. **Encourage 2FA**: Promote usage among clients
2. **Secure backup codes**: Advise users to store safely
3. **Multiple methods**: Offer several 2FA options
4. **Admin enforcement**: Require 2FA for admin accounts
5. **Regular review**: Monitor 2FA adoption rates

## Troubleshooting

| Issue | Solution |
|-------|----------|
| Code not working | Check device time sync |
| Lost backup codes | Admin can disable 2FA |
| SMS not received | Try email code method |
| App deleted | Use backup codes to recover |

## Related Documentation

- [Client Authentication](./whmcs-client-authentication.md)
- [Client Passwords](./whmcs-client-passwords.md)
- [Security Settings](./whmcs-security-settings.md)
- [Client Portal](./whmcs-client-portal.md)