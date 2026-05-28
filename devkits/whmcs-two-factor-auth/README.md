# WHMCS Two-Factor Authentication Module

Multi-factor authentication module with TOTP (Google Authenticator), SMS, Email, and backup codes support.

## Features

- TOTP authenticator app support (Google Authenticator, Authy, etc.)
- Email verification codes
- SMS verification codes
- Backup codes for account recovery
- Account lockout after failed attempts
- Per-user 2FA settings
- Admin and client support
- QR code generation for easy setup

## Installation

1. Copy the module to your WHMCS installation:
   ```
   /path/to/whmcs/modules/servers/twofactorauth/
   ```

2. Activate through WHMCS Admin:
   - Go to **Setup > Addon Modules**
   - Find "Two-Factor Authentication"
   - Click **Activate**
   - Configure settings

## Configuration

| Setting | Default | Description |
|---------|---------|-------------|
| EnableTOTP | yes | Enable authenticator apps |
| EnableEmail | yes | Enable email verification |
| EnableSMS | no | Enable SMS verification |
| EnableBackupCodes | yes | Enable backup codes |
| BackupCodeCount | 10 | Number of backup codes |
| CodeExpiry | 300 | Code expiry (seconds) |
| MaxAttempts | 5 | Max attempts before lockout |
| LockoutDuration | 900 | Lockout duration (seconds) |

## Usage

### Setting Up TOTP

```php
// Generate TOTP secret for user
$secret = twofactorauth_GenerateTOTPSecret();

// Get QR code URL for authenticator app
$qrUrl = twofactorauth_GetTOTPQRUrl($secret, $userEmail, 'YourCompany');

// Complete setup after user verifies code
$isValid = twofactorauth_ValidateSetup($userId, $clientType, $secret, $code);

if ($isValid) {
    twofactorauth_SetupUser($userId, $clientType, 'totp', array('secret' => $secret));
}
```

### Managing User 2FA

```php
// Check if user has 2FA enabled
$enabled = twofactorauth_IsEnabled($userId, 'client');

// Get user 2FA settings
$settings = twofactorauth_GetUserSettings($userId, 'client');
// Returns: method, is_enabled, totp_secret, backup_codes, etc.

// Disable user 2FA (admin function)
twofactorauth_DisableUser($userId, 'client');
```

### Verification

```php
// Verify TOTP code
$result = twofactorauth_VerifyCode($userId, $userType, $code, 'totp');

if ($result['success']) {
    // Code is valid, allow login
} else {
    echo "Error: " . $result['error'];
    echo "Remaining attempts: " . $result['remaining_attempts'];
}

// Verify email/SMS code
$result = twofactorauth_VerifyCode($userId, $userType, $code, 'email');

// Verify backup code
$result = twofactorauth_VerifyCode($userId, $userType, $code, 'backup');
```

### Sending Verification Codes

```php
// Send email verification code
$result = twofactorauth_SendCode($userId, $userType, 'email');

// Send SMS verification code
$result = twofactorauth_SendCode($userId, $userType, 'sms');
```

### Backup Codes

```php
// Get remaining backup codes count
$remaining = twofactorauth_GetRemainingBackupCodes($userId, $clientType);

// Regenerate backup codes (old ones become invalid)
$result = twofactorauth_RegenerateBackupCodes($userId, $clientType);
$newCodes = $result['codes']; // Array of new backup codes
```

### Available Methods

```php
// Get available 2FA methods based on configuration
$methods = twofactorauth_GetAvailableMethods();
// Returns: array('totp' => true, 'email' => true, 'sms' => false, 'backup' => true)
```

### Lockout Management

```php
// Check if user is locked out
$locked = twofactorauth_IsLockedOut($userId, $userType);

// Get failed attempts count
$attempts = twofactorauth_GetFailedAttempts($userId, $userType);
```

## TOTP Setup Flow

1. Generate secret and show QR code to user
2. User scans QR code with authenticator app
3. User enters 6-digit code from app
4. Validate code and save secret

```php
// Step 1: Generate and display
$secret = twofactorauth_GenerateTOTPSecret();
$qrUrl = twofactorauth_GetTOTPQRUrl($secret, $userEmail);
echo "<img src='{$qrUrl}' alt='QR Code'>";

// Step 2: User submits code from app
$code = $_POST['verification_code'];

// Step 3: Validate
if (twofactorauth_VerifyTOTP($secret, $code)) {
    twofactorauth_SetupUser($userId, 'client', 'totp', array('secret' => $secret));
    echo "2FA enabled successfully!";
}
```

## Authenticator App Support

Works with any TOTP-compatible app:
- Google Authenticator
- Authy
- Microsoft Authenticator
- 1Password
- Bitwarden

## Database Tables

- `mod_twofactorauth_user_settings` - User 2FA settings
- `mod_twofactorauth_codes` - Verification codes
- `mod_twofactorauth_attempts` - Login attempts

## API Functions

| Function | Description |
|----------|-------------|
| `twofactorauth_GenerateTOTPSecret()` | Generate TOTP secret |
| `twofactorauth_GenerateTOTPCode()` | Generate TOTP code |
| `twofactorauth_VerifyTOTP()` | Verify TOTP code |
| `twofactorauth_GetTOTPQRUrl()` | Get QR code URL |
| `twofactorauth_GenerateBackupCodes()` | Generate backup codes |
| `twofactorauth_SetupUser()` | Setup user 2FA |
| `twofactorauth_GetUserSettings()` | Get user settings |
| `twofactorauth_IsEnabled()` | Check if enabled |
| `twofactorauth_DisableUser()` | Disable user 2FA |
| `twofactorauth_SendCode()` | Send verification code |
| `twofactorauth_VerifyCode()` | Verify code |
| `twofactorauth_VerifyBackupCode()` | Verify backup code |
| `twofactorauth_RegenerateBackupCodes()` | Regenerate backup codes |
| `twofactorauth_GetRemainingBackupCodes()` | Get remaining codes |
| `twofactorauth_GetAvailableMethods()` | Get available methods |

## Version History

- **1.0.0** - Initial release
  - TOTP support
  - Email verification
  - SMS verification
  - Backup codes
  - Account lockout
  - User management
