# WHMCS Tenant Isolation Workflow

## Overview
Configure tenant isolation for multi-tenant WHMCS deployments.

## Prerequisites
- WHMCS v8.0+
- Multi-tenant architecture

## Step-by-Step Guide

### Step 1: Tenant Database Schema
```php
<?php
function create_tenant(int $tenantId, string $name): void
{
    \WHMCS\Database\Capsule::statement("CREATE DATABASE IF NOT EXISTS whmcs_tenant_$tenantId");
    
    \WHMCS\Database\Capsule::table('tenants')->insert([
        'id' => $tenantId,
        'name' => $name,
        'database' => 'whmcs_tenant_' . $tenantId,
        'status' => 'active',
    ]);
}
```

### Step 2: Tenant-Aware Query Builder
```php
<?php
class TenantAwareQuery
{
    protected int $tenantId;

    public function __construct()
    {
        $this->tenantId = $_SESSION['tenant_id'] ?? 0;
    }

    public function getClients(): array
    {
        return \WHMCS\Database\Capsule::connection('tenant_' . $this->tenantId)
            ->table('tblclients')
            ->get();
    }
}
```

### Step 3: Tenant Middleware
```php
<?php
add_hook('AfterAuthenticate', 1, function($vars) {
    $client = \Auth::user();
    $_SESSION['tenant_id'] = $client->tenant_id;
    
    // Set database connection
    config_tenant_database($client->tenant_id);
});
```

## Checklist
- Tenant database isolation
- Tenant-aware queries implemented
- Session tenant tracking
- Cross-tenant access prevented
