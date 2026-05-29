# WHMCS Admin Permissions

## Overview
Guide for implementing admin permission controls in WHMCS. Covers permission checking, role management, and access control.

## Permission System

### Permission Checker

```php
<?php
// /includes/hooks/admin_permissions.php

function checkAdminPermission(string $permission, ?int $adminId = null): bool
{
    $adminId = $adminId ?? $_SESSION["adminid"];
    
    $admin = Capsule::table("tbladmins")->where("id", $adminId)->first();
    
    if (!$admin) {
        return false;
    }
    
    // Super admins bypass all permissions
    if ($admin->roleid == 1) {
        return true;
    }
    
    // Get admin permissions
    $permissions = Capsule::table("tbladminperms")
        ->where("roleid", $admin->roleid)
        ->pluck("permid")
        ->toArray();
    
    $permissionId = Capsule::table("tblpermissions")
        ->where("name", $permission)
        ->value("id");
    
    return in_array($permissionId, $permissions);
}

function requireAdminPermission(string $permission): void
{
    if (!checkAdminPermission($permission)) {
        http_response_code(403);
        echo "Access Denied. You don't have permission to perform this action.";
        exit;
    }
}
```

### Permission Hook

```php
add_hook("AdminPermissionCheck", 1, function(array $params) {
    $permission = $params["permission"];
    $adminId = $params["admin_id"] ?? $_SESSION["adminid"];
    
    return [
        "allowed" => checkAdminPermission($permission, $adminId)
    ];
});
```

### Custom Permission Definitions

```php
add_hook("AdminPermissionDefinition", 1, function(array $params) {
    return [
        "permissions" => [
            "custom_reports" => [
                "name" => "Custom Reports",
                "description" => "Access custom reporting features"
            ],
            "api_management" => [
                "name" => "API Management",
                "description" => "Manage API keys and access"
            ],
            "mass_operations" => [
                "name" => "Mass Operations",
                "description" => "Perform bulk operations"
            ],
            "system_settings" => [
                "name" => "System Settings",
                "description" => "Modify system configuration"
            ]
        ]
    ];
});
```

## Role Management

```php
function getAdminRole(int $roleId): ?object
{
    return Capsule::table("tbladminroles")->where("id", $roleId)->first();
}

function getAdminPermissionsByRole(int $roleId): array
{
    return Capsule::table("tbladminperms")
        ->where("roleid", $roleId)
        ->pluck("permid")
        ->toArray();
}

function updateAdminRolePermissions(int $roleId, array $permissionIds): bool
{
    // Start transaction
    Capsule::connection()->transaction(function() use ($roleId, $permissionIds) {
        // Remove existing permissions
        Capsule::table("tbladminperms")
            ->where("roleid", $roleId)
            ->delete();
        
        // Add new permissions
        foreach ($permissionIds as $permId) {
            Capsule::table("tbladminperms")->insert([
                "roleid" => $roleId,
                "permid" => $permId
            ]);
        }
    });
    
    return true;
}
```

## Permission Template

```smarty
<!-- /admin/templates/admin_permissions.tpl -->
<div class="permissions-container">
    <h2>Role Permissions</h2>
    <p>Configure permissions for the <strong>{$role.name}</strong> role.</p>
    
    <form method="post" action="adminroles.php?id={$role.id}">
        <input type="hidden" name="token" value="{$token}">
        <input type="hidden" name="action" value="update_permissions">
        
        <div class="permissions-grid">
            {foreach $permission_groups as $group => $permissions}
                <div class="permission-group">
                    <h3>{$group}</h3>
                    
                    {foreach $permissions as $perm}
                        <div class="permission-item">
                            <label>
                                <input type="checkbox" 
                                       name="permissions[]" 
                                       value="{$perm.id}"
                                       {if in_array($perm.id, $role_permissions)}checked{/if}>
                                <span class="permission-name">{$perm.name}</span>
                                <span class="permission-desc">{$perm.description}</span>
                            </label>
                        </div>
                    {/foreach}
                </div>
            {/foreach}
        </div>
        
        <div class="form-actions">
            <button type="submit" class="btn btn-primary">
                Save Permissions
            </button>
        </div>
    </form>
</div>
```

## Best Practices

1. **Least Privilege**: Grant minimum necessary permissions
2. **Role-Based**: Use roles instead of individual permissions
3. **Audit Trail**: Log permission changes
4. **Documentation**: Document all custom permissions
5. **Testing**: Test permission boundaries
6. **Separation**: Separate sensitive permissions
7. **Inheritance**: Allow permission inheritance
8. **Review**: Regular permission reviews
