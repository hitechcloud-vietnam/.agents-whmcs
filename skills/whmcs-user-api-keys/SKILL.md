# WHMCS User API Keys

## Overview
Guide for implementing user API key management in WHMCS. Covers key generation, rotation, and access control.

## API Key Management

### Generate API Key

```php
<?php
// /includes/hooks/api_keys.php

add_hook("CreateAPIKey", 1, function(array $params) {
    $userId = $params["user_id"];
    $name = $params["name"];
    $scopes = $params["scopes"] ?? ["read"];
    $expiresAt = $params["expires_at"] ?? null;
    
    // Generate API key
    $apiKey = "whmcs_" . bin2hex(random_bytes(16));
    $apiSecret = bin2hex(random_bytes(32));
    $keyHash = hash("sha256", $apiKey);
    $secretHash = password_hash($apiSecret, PASSWORD_DEFAULT);
    
    // Store key metadata
    $keyId = Capsule::table("mod_api_keys")->insertGetId([
        "user_id" => $userId,
        "name" => $name,
        "key_hash" => $keyHash,
        "scopes" => json_encode($scopes),
        "created_at" => date("Y-m-d H:i:s"),
        "expires_at" => $expiresAt,
        "last_used_at" => null,
        "active" => 1
    ]);
    
    // Store secret hash separately (not retrievable)
    Capsule::table("mod_api_key_secrets")->insert([
        "key_id" => $keyId,
        "secret_hash" => $secretHash,
        "created_at" => date("Y-m-d H:i:s")
    ]);
    
    // Log creation
    logUserActivity($userId, "api_key_created", "account", [
        "key_name" => $name,
        "key_id" => $keyId
    ]);
    
    return [
        "success" => true,
        "api_key" => $apiKey,
        "api_secret" => $apiSecret,
        "key_id" => $keyId,
        "warning" => "Store the secret securely. It will not be shown again."
    ];
});
```

### Validate API Key

```php
function validateAPIKey(string $apiKey, string $apiSecret = null): ?array
{
    $keyHash = hash("sha256", $apiKey);
    
    $apiKeyRecord = Capsule::table("mod_api_keys")
        ->where("key_hash", $keyHash)
        ->where("active", 1)
        ->first();
    
    if (!$apiKeyRecord) {
        return null;
    }
    
    // Check expiration
    if ($apiKeyRecord->expires_at && 
        strtotime($apiKeyRecord->expires_at) < time()) {
        return null;
    }
    
    // Verify secret if provided (for signed requests)
    if ($apiSecret) {
        $secretRecord = Capsule::table("mod_api_key_secrets")
            ->where("key_id", $apiKeyRecord->id)
            ->first();
        
        if (!$secretRecord || !password_verify($apiSecret, $secretRecord->secret_hash)) {
            return null;
        }
    }
    
    // Update last used
    Capsule::table("mod_api_keys")
        ->where("id", $apiKeyRecord->id)
        ->update(["last_used_at" => date("Y-m-d H:i:s")]);
    
    return [
        "user_id" => $apiKeyRecord->user_id,
        "key_id" => $apiKeyRecord->id,
        "scopes" => json_decode($apiKeyRecord->scopes, true),
        "name" => $apiKeyRecord->name
    ];
}

function hasAPIScope(array $keyData, string $requiredScope): bool
{
    $scopes = $keyData["scopes"] ?? [];
    return in_array($requiredScope, $scopes) || in_array("*", $scopes);
}
```

### Rotate API Key

```php
add_hook("RotateAPIKey", 1, function(array $params) {
    $userId = $params["user_id"];
    $keyId = $params["key_id"];
    
    // Verify ownership
    $key = Capsule::table("mod_api_keys")
        ->where("id", $keyId)
        ->where("user_id", $userId)
        ->first();
    
    if (!$key) {
        return ["error" => "API key not found"];
    }
    
    // Generate new credentials
    $apiKey = "whmcs_" . bin2hex(random_bytes(16));
    $apiSecret = bin2hex(random_bytes(32));
    $keyHash = hash("sha256", $apiKey);
    $secretHash = password_hash($apiSecret, PASSWORD_DEFAULT);
    
    // Invalidate old key but keep for grace period
    Capsule::table("mod_api_keys")
        ->where("id", $keyId)
        ->update([
            "rotated_at" => date("Y-m-d H:i:s"),
            "active" => 0
        ]);
    
    // Create new key with same settings
    $newKeyId = Capsule::table("mod_api_keys")->insertGetId([
        "user_id" => $userId,
        "name" => $key->name . " (rotated)",
        "key_hash" => $keyHash,
        "scopes" => $key->scopes,
        "created_at" => date("Y-m-d H:i:s"),
        "expires_at" => $key->expires_at,
        "rotated_from" => $keyId,
        "active" => 1
    ]);
    
    Capsule::table("mod_api_key_secrets")->insert([
        "key_id" => $newKeyId,
        "secret_hash" => $secretHash,
        "created_at" => date("Y-m-d H:i:s")
    ]);
    
    // Log rotation
    logUserActivity($userId, "api_key_rotated", "account", [
        "old_key_id" => $keyId,
        "new_key_id" => $newKeyId
    ]);
    
    return [
        "success" => true,
        "api_key" => $apiKey,
        "api_secret" => $apiSecret,
        "key_id" => $newKeyId
    ];
});
```

### Revoke API Key

```php
add_hook("RevokeAPIKey", 1, function(array $params) {
    $userId = $params["user_id"];
    $keyId = $params["key_id"];
    
    $updated = Capsule::table("mod_api_keys")
        ->where("id", $keyId)
        ->where("user_id", $userId)
        ->update([
            "active" => 0,
            "revoked_at" => date("Y-m-d H:i:s")
        ]);
    
    if ($updated) {
        logUserActivity($userId, "api_key_revoked", "account", [
            "key_id" => $keyId
        ]);
    }
    
    return ["success" => $updated > 0];
});
```

### List User API Keys

```php
function getUserAPIKeys(int $userId): array
{
    return Capsule::table("mod_api_keys")
        ->where("user_id", $userId)
        ->where("active", 1)
        ->get()
        ->map(function($key) {
            $key->scopes = json_decode($key->scopes, true);
            $key->is_expired = $key->expires_at && 
                               strtotime($key->expires_at) < time();
            return $key;
        })
        ->toArray();
}
```

## API Keys Template

```smarty
<!-- /templates/clientarea_api_keys.tpl -->
<div class="api-keys-container">
    <h2>API Keys</h2>
    <p>Manage your API keys for programmatic access to your account.</p>
    
    <div class="api-keys-actions">
        <a href="create-api-key.php" class="btn btn-primary">
            <i class="fa fa-plus"></i> Create New API Key
        </a>
    </div>
    
    <div class="api-keys-list">
        {foreach $api_keys as $key}
            <div class="api-key-card {if $key->is_expired}expired{/if}">
                <div class="key-header">
                    <h3>{$key->name}</h3>
                    <span class="key-status">
                        {if $key->is_expired}
                            <span class="badge badge-danger">Expired</span>
                        {else}
                            <span class="badge badge-success">Active</span>
                        {/if}
                    </span>
                </div>
                
                <div class="key-info">
                    <div class="info-row">
                        <strong>Created:</strong>
                        <span>{$key->created_at}</span>
                    </div>
                    {if $key->expires_at}
                        <div class="info-row">
                            <strong>Expires:</strong>
                            <span>{$key->expires_at}</span>
                        </div>
                    {/if}
                    <div class="info-row">
                        <strong>Last Used:</strong>
                        <span>{$key->last_used_at ?? 'Never'}</span>
                    </div>
                    <div class="info-row">
                        <strong>Scopes:</strong>
                        <span>{$key->scopes|implode:", "}</span>
                    </div>
                </div>
                
                <div class="key-actions">
                    <a href="rotate-api-key.php?id={$key->id}" 
                       class="btn btn-sm btn-secondary">
                        Rotate
                    </a>
                    <a href="revoke-api-key.php?id={$key->id}" 
                       class="btn btn-sm btn-danger">
                        Revoke
                    </a>
                </div>
            </div>
        {/foreach}
    </div>
    
    <div class="api-documentation">
        <h3>API Documentation</h3>
        <p>Learn how to use API keys to access the WHMCS API.</p>
        <a href="api-docs.php" class="btn btn-outline">View Documentation</a>
    </div>
</div>
```

## Best Practices

1. **Secure Generation**: Use cryptographically secure random bytes
2. **Secret Hashing**: Never store secrets in plain text
3. **Scope Limiting**: Grant minimum necessary scopes
4. **Expiration**: Set expiration dates for keys
5. **Rotation**: Support key rotation without downtime
6. **Audit Trail**: Log all key operations
7. **Rate Limiting**: Apply rate limits to API usage
8. **One-way Display**: Show key only once at creation
