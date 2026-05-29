# WHMCS Access Control Workflow

## Overview
This workflow covers implementing role-based access control (RBAC) for WHMCS modules.

## Step 1: Access Control Service

```php
<?php
// src/Service/AccessControlService.php

namespace WHMCS\Module\Addon\YourModule\Service;

use WHMCS\Database\Capsule;

class AccessControlService
{
    private $roles = [
        'admin' => ['*'],
        'billing_admin' => ['invoices.*', 'clients.view', 'reports.view'],
        'support_admin' => ['tickets.*', 'clients.view'],
        'developer' => ['modules.*', 'api.*', 'settings.view']
    ];

    public function hasPermission(int $adminId, string $permission): bool
    {
        $admin = Capsule::table('tbladmins')->where('id', $adminId)->first();

        if (!$admin) {
            return false;
        }

        $role = $this->getAdminRole($adminId);
        $permissions = $this->roles[$role] ?? [];

        // Check for wildcard permission
        if (in_array('*', $permissions)) {
            return true;
        }

        // Check for exact match
        if (in_array($permission, $permissions)) {
            return true;
        }

        // Check for wildcard match (e.g., 'invoices.*' matches 'invoices.view')
        $permissionParts = explode('.', $permission);
        $category = $permissionParts[0];

        if (in_array("$category.*", $permissions)) {
            return true;
        }

        return false;
    }

    public function getAdminRole(int $adminId): string
    {
        $roleMapping = Capsule::table('mod_admin_roles')
            ->where('admin_id', $adminId)
            ->first();

        return $roleMapping ? $roleMapping->role : 'viewer';
    }

    public function setAdminRole(int $adminId, string $role): void
    {
        Capsule::table('mod_admin_roles')
            ->updateOrInsert(
                ['admin_id' => $adminId],
                ['role' => $role, 'updated_at' => date('Y-m-d H:i:s')]
            );
    }

    public function getPermissionsForRole(string $role): array
    {
        return $this->roles[$role] ?? [];
    }

    public function checkModuleAccess(int $adminId, string $moduleName): bool
    {
        return $this->hasPermission($adminId, "modules.$moduleName");
    }

    public function checkClientAccess(int $adminId, int $clientId): bool
    {
        // Check if admin has client view permission
        if (!$this->hasPermission($adminId, 'clients.view')) {
            return false;
        }

        // Check if client is assigned to admin
        $assignment = Capsule::table('mod_admin_client_assignments')
            ->where('admin_id', $adminId)
            ->where('client_id', $clientId)
            ->first();

        // If no assignment records, allow all (full access)
        // If assignment exists, client must be assigned
        return !$assignment || $assignment !== null;
    }
}
```

## Step 2: Permission Middleware

```php
<?php
// src/Middleware/RequirePermission.php

namespace WHMCS\Module\Addon\YourModule\Middleware;

class RequirePermission
{
    public function handle($permission, callable $next)
    {
        $adminId = $_SESSION['adminid'] ?? null;

        if (!$adminId) {
            header('Location: /admin/login.php');
            exit;
        }

        $accessService = new \WHMCS\Module\Addon\YourModule\Service\AccessControlService();

        if (!$accessService->hasPermission($adminId, $permission)) {
            http_response_code(403);
            echo json_encode(['error' => 'Access denied']);
            exit;
        }

        return $next();
    }
}
```

## Verification Checklist

- [ ] Access control service implemented
- [ ] Roles defined correctly
- [ ] Permission checking working
- [ ] Client access control working
- [ ] Middleware implemented
- [ ] Test permissions verified
