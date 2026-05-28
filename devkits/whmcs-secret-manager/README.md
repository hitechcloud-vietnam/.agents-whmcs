# WHMCS Secret Manager Module

Secrets/credentials management with encryption, access control, audit logging, and rotation.

## Features

- AES-256 encryption
- Multiple secret types
- Access control (user/role based)
- Full audit logging
- Secret rotation
- Version history
- Expiration management
- Category organization

## Installation

1. Copy `secretmanager.php` to `/path/to/whmcs/modules/addons/secretmanager/`
2. Activate in WHMCS Admin > System Settings > Module Addons
3. Set master encryption key in configuration

## Usage

```php
// Create secret
$result = secretmanager_CreateSecret(array(
    'secret_key' => 'stripe_api_key',
    'secret_name' => 'Stripe API Key',
    'value' => 'sk_live_xxxxxxxxxxxx',
    'secret_type' => 'apikey',
    'category' => 'payment',
    'rotation_policy' => 'quarterly',
    'created_by' => $adminId,
    'access' => array(
        array('user_id' => $adminId, 'permission' => 'admin'),
        array('user_id' => $billingUserId, 'permission' => 'read')
    )
));

// Get secret (with access control)
$result = secretmanager_GetSecret('stripe_api_key', $adminId);
if ($result['success']) {
    $apiKey = $result['value'];
}

// Update secret
$result = secretmanager_UpdateSecret('stripe_api_key', 'sk_live_new_key', $adminId);

// Delete secret (soft delete)
secretmanager_DeleteSecret('old_secret');

// Hard delete
secretmanager_DeleteSecret('old_secret', true);

// Rotate secret
$result = secretmanager_RotateSecret('stripe_api_key');

// Generate new secret
$newPassword = secretmanager_GenerateSecret('password', 32);
$newApiKey = secretmanager_GenerateSecret('apikey', 64);
$newToken = secretmanager_GenerateSecret('token');

// Grant access
secretmanager_GrantAccess($secretId, array(
    array('user_id' => $userId, 'permission' => 'read', 'granted_by' => $adminId),
    array('role_id' => $adminRoleId, 'permission' => 'admin')
));

// Revoke access
secretmanager_RevokeAccess($secretId, $userId);

// Check access
$hasAccess = secretmanager_HasAccess($secretId, $userId, 'read');

// List secrets
$secrets = secretmanager_GetSecrets();
$secrets = secretmanager_GetSecrets(array('category' => 'payment'));
$secrets = secretmanager_GetSecrets(array('type' => 'apikey'));

// Get secret info (without value)
$info = secretmanager_GetSecretInfo('stripe_api_key');

// Get version history
$versions = secretmanager_GetVersions('stripe_api_key');

// Get audit log
$logs = secretmanager_GetAuditLog($secretId, 100);

// Get all audit logs
$logs = secretmanager_GetAuditLog();

// Set rotation schedule
secretmanager_SetRotationSchedule($secretId, 'quarterly'); // 90 days
secretmanager_SetRotationSchedule($secretId, 'monthly');   // 30 days
secretmanager_SetRotationSchedule($secretId, 'yearly');     // 365 days

// Get rotation schedule
$schedule = secretmanager_GetRotationSchedule($secretId);

// Get secrets due for rotation
$dueRotation = secretmanager_GetDueRotation(7); // Next 7 days

// Get categories
$categories = secretmanager_GetCategories();
```

## Configuration Options

| Option | Type | Default | Description |
|--------|------|---------|-------------|
| EncryptionAlgorithm | dropdown | AES-256-GCM | Encryption method |
| KeyDerivation | dropdown | argon2id | KDF function |
| EnableAuditLog | yesno | yes | Audit logging |
| EnableRotation | yesno | yes | Auto-rotation |
| RotationDays | text | 90 | Rotation period |
| MasterKey | password | - | Master encryption key |

## Secret Types

| Type | Use Case | Default Length |
|------|----------|----------------|
| password | Passwords | 16 chars |
| apikey | API keys | 32 bytes |
| token | Auth tokens | 32 bytes |
| certificate | SSL certs | 2048 bits |
| ssh_key | SSH keys | 32 bytes |

## Access Permissions

| Permission | Description |
|------------|-------------|
| read | View secret value |
| write | Update secret |
| admin | Full access |

## Rotation Policies

| Policy | Interval |
|--------|----------|
| monthly | 30 days |
| quarterly | 90 days |
| yearly | 365 days |

## Audit Actions

| Action | Description |
|--------|-------------|
| create | Secret created |
| read | Secret accessed |
| update | Secret updated |
| delete | Secret deleted |
| access_denied | Access denied |
| rotate | Secret rotated |

## Encryption

Secrets are encrypted using AES-256:
- AES-256-GCM (default, authenticated encryption)
- AES-256-CBC (legacy support)

Key derivation uses:
- Argon2id (default)
- bcrypt
- PBKDF2

## Best Practices

```php
// Use access control
$result = secretmanager_GetSecret('api_key', $currentUserId);
if (!$result['success']) {
    // Log unauthorized access
    logActivity('Unauthorized secret access attempt');
    return;
}

// Always use rotation
secretmanager_CreateSecret(array(
    'secret_key' => 'db_password',
    'value' => $generatedPassword,
    'rotation_policy' => 'quarterly' // Auto-rotation
));

// Set expiration
secretmanager_CreateSecret(array(
    'secret_key' => 'temp_token',
    'value' => $token,
    'expires_days' => 7 // Auto-expire
));
```

## Database Tables

- `mod_secretmanager_secrets` - Secret storage
- `mod_secretmanager_access` - Access control
- `mod_secretmanager_audit` - Audit logs
- `mod_secretmanager_keys` - Key management
- `mod_secretmanager_rotation_schedules` - Rotation schedules
