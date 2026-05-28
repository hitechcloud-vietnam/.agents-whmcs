# WHMCS API Key Manager Module

Comprehensive API key management with scopes, rate limiting, and usage tracking.

## Features

- API key creation and management
- Scope-based access control
- Rate limiting per key
- Usage tracking and statistics
- Request logging
- Key expiration
- Multiple key support per user

## Installation

Copy module to `/path/to/whmcs/modules/servers/apikeymanager/` and activate.

## Usage

```php
// Create API key
$result = apikeymanager_CreateKey(array(
    'name' => 'Production API Key',
    'user_id' => $clientId,
    'scopes' => array('read', 'billing'),
    'rate_limit' => 100,
    'expires_days' => 365
));
// Returns: key_id, api_key (store securely!)

// Get key details
$key = apikeymanager_GetKey($keyId);

// Get user keys
$keys = apikeymanager_GetUserKeys($userId);

// Validate key and scope
$result = apikeymanager_ValidateKey($apiKey, 'read');
if (!$result['valid']) {
    echo $result['error'];
    if (!empty($result['retry_after'])) {
        header('Retry-After: ' . $result['retry_after']);
    }
}

// Update key
apikeymanager_UpdateKey($keyId, array(
    'name' => 'Updated Name',
    'scopes' => array('read', 'write'),
    'rate_limit' => 200
));

// Revoke key (soft delete)
apikeymanager_RevokeKey($keyId);

// Delete key (permanent)
apikeymanager_DeleteKey($keyId);

// Get available scopes
$scopes = apikeymanager_GetScopes();

// Log API request
apikeymanager_LogRequest($keyId, '/api/endpoint', 'GET', 200, 45);

// Get key statistics
$stats = apikeymanager_GetKeyStats($keyId, 30);
// Returns: total_requests, avg_response_ms, error_count

// Get key logs
$logs = apikeymanager_GetKeyLogs($keyId, 100);

// Clean expired keys
$cleaned = apikeymanager_CleanExpiredKeys();
```

## Default Scopes

| Scope | Description |
|-------|-------------|
| `read` | Read-only access |
| `write` | Create and update data |
| `delete` | Delete data |
| `billing` | Billing information |
| `support` | Support features |
| `admin` | Administrative access |

## API Functions

| Function | Description |
|----------|-------------|
| `apikeymanager_CreateKey()` | Create new API key |
| `apikeymanager_GetKey()` | Get key details |
| `apikeymanager_GetKeyByToken()` | Validate and get key |
| `apikeymanager_GetUserKeys()` | Get user's keys |
| `apikeymanager_ValidateKey()` | Validate key and scope |
| `apikeymanager_UpdateKey()` | Update key settings |
| `apikeymanager_RevokeKey()` | Revoke API key |
| `apikeymanager_DeleteKey()` | Delete API key |
| `apikeymanager_GetScopes()` | Get available scopes |
| `apikeymanager_LogRequest()` | Log API request |
| `apikeymanager_GetKeyStats()` | Get key statistics |
| `apikeymanager_GetKeyLogs()` | Get request logs |
| `apikeymanager_CleanExpiredKeys()` | Clean expired keys |
