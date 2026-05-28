# WHMCS Auth Manager Module

Authentication manager with MFA, SSO, and security features.

## Features

- Multi-factor authentication (TOTP, SMS, Email, Backup codes)
- Single Sign-On (SSO) integration
- Password policies
- Session management
- IP whitelisting/blacklisting
- Login attempt limiting
- Account lockout
- Security audit logs
- Password reset workflows
- API key authentication
- OAuth 2.0 support
- LDAP/Active Directory integration
- Custom login flows
- Brute force protection
- Security notifications

## Installation

1. Copy module to `/path/to/whmcs/modules/addons/authmanager/`
2. Activate in WHMCS Admin > System Settings > Module Addons
3. Configure authentication methods

## Usage

```php
// Enable MFA for user
$result = authmanager_EnableMFA($userId, 'totp', array(
    'issuer' => 'MyCompany'
));
// Returns: secret, qr_code, backup_codes

// Verify MFA token
$result = authmanager_VerifyMFA($userId, $token);
// Returns: success, remaining_attempts

// Disable MFA
authmanager_DisableMFA($userId);

// Setup SSO
authmanager_SetupSSO($userId, array(
    'provider' => 'google',
    'email' => 'user@example.com'
));

// Initiate SSO login
$result = authmanager_InitiateSSOLogin('google');
// Returns: redirect_url, state, code_verifier

// Handle SSO callback
$result = authmanager_HandleSSOCallback($provider, $code, $state);

// Create API session
$result = authmanager_CreateAPISession($userId, array(
    'scopes' => array('read', 'write'),
    'expires_in' => 3600
));
// Returns: session_id, access_token, refresh_token

// Validate API token
$result = authmanager_ValidateAPIToken($accessToken);
// Returns: valid, user_id, scopes, expires_at

// Refresh API token
$result = authmanager_RefreshAPIToken($refreshToken);

// Revoke API session
authmanager_RevokeSession($sessionId);

// Get active sessions
$sessions = authmanager_GetActiveSessions($userId);

// Revoke all sessions
authmanager_RevokeAllSessions($userId);

// Set password policy
authmanager_SetPasswordPolicy(array(
    'min_length' => 12,
    'require_uppercase' => true,
    'require_lowercase' => true,
    'require_numbers' => true,
    'require_special' => true,
    'expire_days' => 90,
    'history_count' => 5
));

// Validate password
$result = authmanager_ValidatePassword($userId, $password);
// Returns: valid, errors[]

// Add IP to whitelist
authmanager_AddIPWhitelist($userId, '192.168.1.0/24');

// Check IP access
$result = authmanager_CheckIPAccess($userId, $_SERVER['REMOTE_ADDR']);
// Returns: allowed, reason

// Get login history
$history = authmanager_GetLoginHistory($userId, 30);

// Get security alerts
$alerts = authmanager_GetSecurityAlerts($userId);
```

## Configuration Options

| Option | Type | Default | Description |
|--------|------|---------|-------------|
| EnableMFA | yesno | yes | Enable MFA |
| DefaultMFA | dropdown | totp | Default MFA method |
| EnforceMFA | yesno | no | Enforce MFA for all |
| EnableSSO | yesno | yes | Enable SSO |
| SSOProviders | text | google,microsoft,facebook | SSO providers |
| PasswordMinLength | text | 12 | Minimum password length |
| PasswordHistory | text | 5 | Password history count |
| SessionTimeout | text | 3600 | Session timeout (seconds) |
| MaxLoginAttempts | text | 5 | Max failed attempts |
| LockoutDuration | text | 900 | Lockout duration (seconds) |
| EnableIPWhitelist | yesno | no | Enable IP whitelist |

## MFA Methods

| Method | Description |
|--------|-------------|
| totp | Time-based OTP (Google Authenticator) |
| sms | SMS verification code |
| email | Email verification code |
| backup | Backup codes |
| security_key | FIDO2 security key |

## SSO Providers

| Provider | Description |
|----------|-------------|
| google | Google Workspace |
| microsoft | Microsoft/Azure AD |
| facebook | Facebook |
| github | GitHub |
| oidc | Generic OpenID Connect |

## Database Tables

- `mod_authmanager_mfa` - MFA configurations
- `mod_authmanager_mfa_secrets` - MFA secrets
- `mod_authmanager_sso_links` - SSO links
- `mod_authmanager_sessions` - API sessions
- `mod_authmanager_password_history` - Password history
- `mod_authmanager_ip_rules` - IP rules
- `mod_authmanager_login_history` - Login history
- `mod_authmanager_security_alerts` - Security alerts
- `mod_authmanager_policies` - Policies

## API Functions

| Function | Description |
|----------|-------------|
| `authmanager_EnableMF` | Enable MFA |
| `authmanager_VerifyMFA()` | Verify MFA token |
| `authmanager_DisableMFA()` | Disable MFA |
| `authmanager_SetupSSO()` | Setup SSO |
| `authmanager_InitiateSSOLogin()` | Start SSO flow |
| `authmanager_HandleSSOCallback()` | Handle SSO callback |
| `authmanager_CreateAPISession()` | Create API session |
| `authmanager_ValidateAPIToken()` | Validate token |
| `authmanager_RefreshAPIToken()` | Refresh token |
| `authmanager_RevokeSession()` | Revoke session |
| `authmanager_GetActiveSessions()` | Get sessions |
| `authmanager_RevokeAllSessions()` | Revoke all sessions |
| `authmanager_SetPasswordPolicy()` | Set password rules |
| `authmanager_ValidatePassword()` | Validate password |
| `authmanager_AddIPWhitelist()` | Add IP whitelist |
| `authmanager_CheckIPAccess()` | Check IP access |
| `authmanager_GetLoginHistory()` | Get login log |
| `authmanager_GetSecurityAlerts()` | Get alerts |
