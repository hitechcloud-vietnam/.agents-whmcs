# WHMCS User Permissions

## Overview
Guide for implementing custom permission systems in WHMCS. Covers permission checks, roles, and access control.

## Permission System

### Permission Definitions

```php
<?php
// /includes/hooks/permissions.php

// Define custom permissions
define("CUSTOM_PERMISSIONS", [
    "view_billing" => "View billing information",
    "manage_services" => "Manage own services",
    "download_invoices" => "Download invoices",
    "update_payment" => "Update payment methods",
    "submit_tickets" => "Submit support tickets",
    "manage_contacts" => "Manage sub-accounts",
    "api_access" => "Access API",
    "view_reports" => "View usage reports",
]);

add_hook("PermissionDefinition", 1, function(array $params) {
    return [
        "permissions" => CUSTOM_PERMISSIONS
    ];
});
```

### Check Permission

```php
function hasPermission(int $clientId, string $permission): bool
{
    $client = Capsule::table("tblclients")->where("id", $clientId)->first();
    
    if (!$client) {
        return false;
    }
    
    // Check client-specific permissions
    $clientPerms = Capsule::table("mod_client_permissions")
        ->where("client_id", $clientId)
        ->pluck("permission")
        ->toArray();
    
    if (in_array($permission, $clientPerms)) {
        return true;
    }
    
    // Check group permissions
    $groupPerms = Capsule::table("mod_group_permissions")
        ->where("group_id", $client->groupid)
        ->pluck("permission")
        ->toArray();
    
    if (in_array($permission, $groupPerms)) {
        return true;
    }
    
    // Check against default permissions
    $defaultPerms = getDefaultPermissions($client->groupid);
    return in_array($permission, $defaultPerms);
}

add_hook("CheckPermission", 1, function(array $params) {
    $clientId = $params["client_id"];
    $permission = $params["permission"];
    
    return ["allowed" => hasPermission($clientId, $permission)];
});
```

### Grant/Revoke Permission

```php
add_hook("GrantPermission", 1, function(array $params) {
    $clientId = $params["client_id"];
    $permission = $params["permission"];
    
    // Check if already granted
    $existing = Capsule::table("mod_client_permissions")
        ->where("client_id", $clientId)
        ->where("permission", $permission)
        ->first();
    
    if ($existing) {
        return ["success" => true, "message" => "Permission already granted."];
    }
    
    Capsule::table("mod_client_permissions")->insert([
        "client_id" => $clientId,
        "permission" => $permission,
        "granted_by" => $params["granted_by"] ?? 0,
        "granted_at" => date("Y-m-d H:i:s")
    ]);
    
    // Log action
    logActivity("Permission '{$permission}' granted to client #{$clientId}");
    
    return ["success" => true];
});

add_hook("RevokePermission", 1, function(array $params) {
    $clientId = $params["client_id"];
    $permission = $params["permission"];
    
    $deleted = Capsule::table("mod_client_permissions")
        ->where("client_id", $clientId)
        ->where("permission", $permission)
        ->delete();
    
    if ($deleted) {
        logActivity("Permission '{$permission}' revoked from client #{$clientId}");
    }
    
    return ["success" => true];
});
```

### Permission Check Middleware

```php
function checkPermissionOrDie(string $permission): void
{
    if (!isLoggedIn()) {
        header("Location: login.php");
        exit;
    }
    
    $clientId = $_SESSION["uid"];
    
    if (!hasPermission($clientId, $permission)) {
        // Show access denied
        $smarty = new \WHMCS\Smarty\Front();
        $smarty->assign("error_title", "Access Denied");
        $smarty->assign("error_message", "You don't have permission to access this resource.");
        $smarty->display("error.tpl");
        exit;
    }
}

// Usage in pages
add_hook("ClientAreaPageBilling", 1, function(array $params) {
    if (!hasPermission($params["userid"], "view_billing")) {
        unset($params["sidebar"]["billing"]);
    }
    return $params;
});
```

### Feature-Based Access

```php
function hasFeatureAccess(int $clientId, string $feature): bool
{
    // Check if feature is enabled globally
    if (!getFeatureConfig($feature, "enabled")) {
        return false;
    }
    
    // Check if client has access to feature
    $clientFeatures = Capsule::table("mod_client_features")
        ->where("client_id", $clientId)
        ->where("feature", $feature)
        ->where("enabled", 1)
        ->first();
    
    if ($clientFeatures) {
        // Check expiration
        if ($clientFeatures->expires_at && 
            strtotime($clientFeatures->expires_at) < time()) {
            return false;
        }
        return true;
    }
    
    // Check against plan/tier
    $client = Capsule::table("tblclients")->where("id", $clientId)->first();
    $planFeatures = Capsule::table("mod_plan_features")
        ->where("plan_id", $client->groupid)
        ->where("feature", $feature)
        ->where("enabled", 1)
        ->first();
    
    return (bool)$planFeatures;
}

function requireFeature(string $feature): void
{
    if (!isLoggedIn()) {
        redir("login.php");
    }
    
    if (!hasFeatureAccess($_SESSION["uid"], $feature)) {
        $upgradeUrl = "upgrade.php?feature=" . urlencode($feature);
        $_SESSION["UpgradeRequired"] = $feature;
        redir($upgradeUrl);
    }
}
```

## Database Schema

```php
// Custom permissions table
Capsule::schema()->create('mod_client_permissions', function($t) {
    $t->increments('id');
    $t->integer('client_id');
    $t->string('permission', 100);
    $t->integer('granted_by');
    $t->timestamp('granted_at');
    $t->unique(['client_id', 'permission']);
});

Capsule::schema()->create('mod_group_permissions', function($t) {
    $t->increments('id');
    $t->integer('group_id');
    $t->string('permission', 100);
    $t->unique(['group_id', 'permission']);
});

Capsule::schema()->create('mod_client_features', function($t) {
    $t->increments('id');
    $t->integer('client_id');
    $t->string('feature', 100);
    $t->boolean('enabled')->default(true);
    $t->timestamp('expires_at')->nullable();
    $t->unique(['client_id', 'feature']);
});
```

## Best Practices

1. **Granular Permissions**: Define specific, narrow permissions
2. **Role-Based**: Assign permissions via roles/groups
3. **Audit Trail**: Log all permission changes
4. **Default Deny**: Start with no permissions, grant explicitly
5. **Feature Flags**: Combine with feature flags for flexibility
6. **Expiration**: Support time-limited permissions
7. **Inheritance**: Allow permission inheritance from groups
8. **Override**: Allow client-specific overrides
