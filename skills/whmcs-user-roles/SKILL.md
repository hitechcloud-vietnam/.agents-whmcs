# WHMCS User Roles

## Overview
Guide for implementing role-based access control (RBAC) for WHMCS users. Covers role definitions, assignment, and hierarchy.

## Role System

### Define Roles

```php
<?php
// /includes/hooks/user_roles.php

define("USER_ROLES", [
    "standard" => [
        "name" => "Standard User",
        "permissions" => [
            "view_own_services",
            "view_own_invoices",
            "submit_tickets",
            "update_profile",
            "manage_contacts"
        ],
        "max_services" => 10,
        "max_contacts" => 2
    ],
    "premium" => [
        "name" => "Premium User",
        "permissions" => [
            "view_own_services",
            "view_own_invoices",
            "download_invoices",
            "submit_tickets",
            "update_profile",
            "manage_contacts",
            "view_billing",
            "api_access",
            "view_reports"
        ],
        "max_services" => 100,
        "max_contacts" => 10
    ],
    "enterprise" => [
        "name" => "Enterprise User",
        "permissions" => [
            "view_own_services",
            "view_own_invoices",
            "download_invoices",
            "submit_tickets",
            "update_profile",
            "manage_contacts",
            "view_billing",
            "api_access",
            "view_reports",
            "bulk_operations",
            "custom_branding",
            "dedicated_support"
        ],
        "max_services" => -1, // Unlimited
        "max_contacts" => -1
    ]
]);

add_hook("UserRoleDefinition", 1, function(array $params) {
    return ["roles" => USER_ROLES];
});
```

### Assign Role

```php
add_hook("AssignUserRole", 1, function(array $params) {
    $clientId = $params["client_id"];
    $role = $params["role"];
    
    if (!isset(USER_ROLES[$role])) {
        return ["error" => "Invalid role specified."];
    }
    
    // Update client
    Capsule::table("tblclients")
        ->where("id", $clientId)
        ->update([
            "role" => $role,
            "role_assigned_at" => date("Y-m-d H:i:s"),
            "role_assigned_by" => $params["assigned_by"] ?? 0
        ]);
    
    // Apply role permissions
    applyRolePermissions($clientId, $role);
    
    // Log assignment
    Capsule::table("mod_role_assignments")->insert([
        "client_id" => $clientId,
        "role" => $role,
        "assigned_by" => $params["assigned_by"] ?? 0,
        "assigned_at" => date("Y-m-d H:i:s")
    ]);
    
    // Notify client
    send_email("RoleAssigned", $clientId, [
        "role_name" => USER_ROLES[$role]["name"]
    ]);
    
    return ["success" => true];
});

function applyRolePermissions(int $clientId, string $role): void
{
    // Clear existing permissions
    Capsule::table("mod_client_permissions")
        ->where("client_id", $clientId)
        ->delete();
    
    // Apply role permissions
    foreach (USER_ROLES[$role]["permissions"] as $permission) {
        Capsule::table("mod_client_permissions")->insert([
            "client_id" => $clientId,
            "permission" => $permission,
            "granted_by" => 0, // System
            "granted_at" => date("Y-m-d H:i:s"),
            "source" => "role:" . $role
        ]);
    }
}
```

### Get User Role

```php
function getUserRole(int $clientId): ?array
{
    $client = Capsule::table("tblclients")->where("id", $clientId)->first();
    
    if (!$client || empty($client->role)) {
        return USER_ROLES["standard"]; // Default role
    }
    
    return USER_ROLES[$client->role] ?? USER_ROLES["standard"];
}

function getUserPermissions(int $clientId): array
{
    // Get role permissions
    $role = getUserRole($clientId);
    $permissions = $role["permissions"];
    
    // Add individual permission overrides
    $overrides = Capsule::table("mod_client_permissions")
        ->where("client_id", $clientId)
        ->where("source", "like", "override:%")
        ->pluck("permission")
        ->toArray();
    
    return array_unique(array_merge($permissions, $overrides));
}

add_hook("GetUserPermissions", 1, function(array $params) {
    return [
        "permissions" => getUserPermissions($params["client_id"])
    ];
});
```

### Check Role Permission

```php
function hasRolePermission(int $clientId, string $permission): bool
{
    $permissions = getUserPermissions($clientId);
    return in_array($permission, $permissions);
}

function requireRolePermission(string $permission): void
{
    if (!isLoggedIn()) {
        redir("login.php");
    }
    
    if (!hasRolePermission($_SESSION["uid"], $permission)) {
        $role = getUserRole($_SESSION["uid"]);
        $_SESSION["UpgradeRequired"] = [
            "permission" => $permission,
            "current_role" => $role["name"]
        ];
        redir("upgrade-account.php?required=" . urlencode($permission));
    }
}

// Role upgrade/downgrade
add_hook("ChangeUserRole", 1, function(array $params) {
    $clientId = $params["client_id"];
    $newRole = $params["new_role"];
    $reason = $params["reason"] ?? "";
    
    $oldRole = Capsule::table("tblclients")
        ->where("id", $clientId)
        ->value("role");
    
    // Apply new role
    add_hook("AssignUserRole", 1, function($p) use ($newRole) {
        // Role assignment logic
    });
    
    // Log change
    Capsule::table("mod_role_changes")->insert([
        "client_id" => $clientId,
        "old_role" => $oldRole,
        "new_role" => $newRole,
        "reason" => $reason,
        "changed_at" => date("Y-m-d H:i:s"),
        "changed_by" => $params["changed_by"] ?? 0
    ]);
    
    return ["success" => true];
});
```

## Role Template

```smarty
<!-- /templates/clientarea_roles.tpl -->
<div class="role-management">
    <h2>Account Tier</h2>
    
    <div class="current-role">
        <span class="role-badge role-{$current_role}">
            {$role_name}
        </span>
        <p class="role-description">{$role_description}</p>
    </div>
    
    <div class="role-features">
        <h3>Your Features</h3>
        <ul class="feature-list">
            {foreach $role_permissions as $permission}
                <li>
                    <i class="fa fa-check"></i>
                    {$permission}
                </li>
            {/foreach}
        </ul>
    </div>
    
    <div class="upgrade-options">
        <h3>Available Upgrades</h3>
        {foreach $available_roles as $role => $details}
            <div class="upgrade-card {if $role eq $current_role}current{/if}">
                <h4>{$details.name}</h4>
                <ul>
                    {foreach $details.added_permissions as $perm}
                        <li><i class="fa fa-plus"></i> {$perm}</li>
                    {/foreach}
                </ul>
                <a href="upgrade.php?role={$role}" 
                   class="btn btn-primary {if $role eq $current_role}disabled{/if}">
                    {if $role eq $current_role}
                        Current Plan
                    {else}
                        Upgrade to {$details.name}
                    {/if}
                </a>
            </div>
        {/foreach}
    </div>
</div>
```

## Best Practices

1. **Clear Definitions**: Well-defined permission sets per role
2. **Hierarchy**: Logical role hierarchy (standard < premium < enterprise)
3. **Migration**: Smooth upgrade paths between roles
4. **Limits**: Define limits per role (services, contacts, etc.)
5. **Audit**: Log all role changes
6. **Notifications**: Notify users of role changes
7. **Overrides**: Allow permission overrides when needed
8. **Documentation**: Clear documentation of each role's capabilities
