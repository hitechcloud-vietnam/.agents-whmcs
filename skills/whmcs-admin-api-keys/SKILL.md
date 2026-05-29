# WHMCS Admin API Keys

## Overview
Guide for managing admin API keys in WHMCS. Covers API key generation, access control, and key rotation.

## Admin API Key Management

### Generate Admin API Key

```php
<?php
// /includes/hooks/admin_api_keys.php

add_hook("AdminGenerateAPIKey", 1, function(array $params) {
    $adminId = $params["admin_id"];
    $name = $params["name"];
    $scopes = $params["scopes"] ?? [];
    $expiresAt = $params["expires_at"] ?? null;
    
    // Generate credentials
    $apiKey = "whmcs_admin_" . bin2hex(random_bytes(16));
    $apiSecret = bin2hex(random_bytes(32));
    
    $keyHash = hash("sha256", $apiKey);
    $secretHash = password_hash($apiSecret, PASSWORD_DEFAULT);
    
    // Store key
    $keyId = Capsule::table("mod_admin_api_keys")->insertGetId([
        "admin_id" => $adminId,
        "name" => $name,
        "key_hash" => $keyHash,
        "scopes" => json_encode($scopes),
        "created_at" => date("Y-m-d H:i:s"),
        "expires_at" => $expiresAt,
        "last_used_at" => null,
        "active" => 1
    ]);
    
    // Store secret (hashed)
    Capsule::table("mod_admin_api_key_secrets")->insert([
        "key_id" => $keyId,
        "secret_hash" => $secretHash
    ]);
    
    // Log creation
    logAdminActivity("api_key_created", $keyId, [
        "name" => $name,
        "admin_id" => $adminId
    ]);
    
    // Return credentials (show secret only once)
    return [
        "success" => true,
        "api_key" => $apiKey,
        "api_secret" => $apiSecret,
        "key_id" => $keyId,
        "warning" => "Store the secret securely. It will not be shown again."
    ];
});
```

### Validate Admin API Key

```php
function validateAdminAPIKey(string $apiKey, ?string $apiSecret = null): ?array
{
    $keyHash = hash("sha256", $apiKey);
    
    $key = Capsule::table("mod_admin_api_keys")
        ->where("key_hash", $keyHash)
        ->where("active", 1)
        ->first();
    
    if (!$key) {
        return null;
    }
    
    // Check expiration
    if ($key->expires_at && strtotime($key->expires_at) < time()) {
        return null;
    }
    
    // Verify secret
    if ($apiSecret) {
        $secret = Capsule::table("mod_admin_api_key_secrets")
            ->where("key_id", $key->id)
            ->first();
        
        if (!$secret || !password_verify($apiSecret, $secret->secret_hash)) {
            return null;
        }
    }
    
    // Update last used
    Capsule::table("mod_admin_api_keys")
        ->where("id", $key->id)
        ->update(["last_used_at" => date("Y-m-d H:i:s")]);
    
    // Get admin info
    $admin = Capsule::table("tbladmins")->where("id", $key->admin_id)->first();
    
    return [
        "key_id" => $key->id,
        "admin_id" => $key->admin_id,
        "admin_username" => $admin->username,
        "scopes" => json_decode($key->scopes, true)
    ];
}
```

### Revoke Admin API Key

```php
add_hook("AdminRevokeAPIKey", 1, function(array $params) {
    $keyId = $params["key_id"];
    $adminId = $params["admin_id"] ?? $_SESSION["adminid"];
    
    // Verify ownership
    $key = Capsule::table("mod_admin_api_keys")
        ->where("id", $keyId)
        ->where("admin_id", $adminId)
        ->first();
    
    if (!$key) {
        return ["error" => "API key not found"];
    }
    
    Capsule::table("mod_admin_api_keys")
        ->where("id", $keyId)
        ->update([
            "active" => 0,
            "revoked_at" => date("Y-m-d H:i:s")
        ]);
    
    logAdminActivity("api_key_revoked", $keyId, [
        "name" => $key->name
    ]);
    
    return ["success" => true];
});
```

## Admin API Keys Template

```smarty
<!-- /admin/templates/admin_api_keys.tpl -->
<div class="admin-api-keys-container">
    <div class="page-header">
        <h2>API Keys</h2>
        <a href="create_api_key.php" class="btn btn-primary">
            <i class="fa fa-plus"></i> Generate New Key
        </a>
    </div>
    
    <div class="api-keys-list">
        {foreach $api_keys as $key}
            <div class="api-key-card">
                <div class="key-info">
                    <h4>
                        {$key.name}
                        {if !$key.active}
                            <span class="badge badge-danger">Revoked</span>
                        {/if}
                    </h4>
                    <div class="key-meta">
                        <span><i class="fa fa-calendar"></i> Created: {$key.created_at}</span>
                        <span><i class="fa fa-clock-o"></i> Last used: {$key.last_used_at|default:'Never'}</span>
                        {if $key.expires_at}
                            <span><i class="fa fa-exclamation-circle"></i> Expires: {$key.expires_at}</span>
                        {/if}
                    </div>
                    <div class="key-scopes">
                        <strong>Scopes:</strong>
                        {foreach $key.scopes as $scope}
                            <span class="scope-badge">{$scope}</span>
                        {/foreach}
                    </div>
                </div>
                <div class="key-actions">
                    {if $key.active}
                        <a href="rotate_api_key.php?id={$key.id}" class="btn btn-sm btn-default">
                            Rotate
                        </a>
                        <a href="revoke_api_key.php?id={$key.id}" class="btn btn-sm btn-danger">
                            Revoke
                        </a>
                    {/if}
                </div>
            </div>
        {/foreach}
    </div>
</div>
```

## Best Practices

1. **Secret Display**: Show secret only once at creation
2. **Rotation**: Support key rotation without downtime
3. **Scopes**: Limit access with granular scopes
4. **Expiration**: Set expiration for temporary access
5. **Audit Trail**: Log all key operations
6. **Rate Limiting**: Apply rate limits to API usage
7. **IP Restrictions**: Optionally restrict to certain IPs
8. **Revocation**: Quick revocation capability
