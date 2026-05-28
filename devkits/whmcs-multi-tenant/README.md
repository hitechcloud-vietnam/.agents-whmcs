# WHMCS Multi-Tenant Module

Multi-tenant architecture for resellers with isolated data and custom branding.

## Features

- Tenant management
- Custom branding per tenant
- Client isolation
- Quota management
- Status control

## Installation

Copy module to `/path/to/whmcs/modules/servers/multitenant/` and activate.

## Usage

```php
// Create tenant
multitenant_CreateTenant(array(
    'name' => 'Reseller Corp',
    'owner_user_id' => $adminId,
    'branding' => array('logo' => 'url', 'colors' => array('#fff')),
    'quotas' => array('client_quota' => 50)
));

// Get tenant
$tenant = multitenant_GetTenant($tenantKey);

// Add client to tenant
multitenant_AddClient($tenantKey, $clientId);

// Get tenant clients
$clients = multitenant_GetClients($tenantKey);

// Set branding
multitenant_SetBranding($tenantKey, array('logo' => '...', 'primary_color' => '#000'));

// Get branding
$branding = multitenant_GetBranding($tenantKey);

// Get tenant by client
$tenant = multitenant_GetTenantByClient($clientId);

// Update status
multitenant_UpdateStatus($tenantKey, 'suspended');
```

## API Functions

| Function | Description |
|----------|-------------|
| `multitenant_CreateTenant()` | Create tenant |
| `multitenant_GetTenant()` | Get tenant by key |
| `multitenant_GetTenants()` | Get all tenants |
| `multitenant_AddClient()` | Add client to tenant |
| `multitenant_RemoveClient()` | Remove client |
| `multitenant_GetClients()` | Get tenant clients |
| `multitenant_GetTenantByClient()` | Get tenant for client |
| `multitenant_SetBranding()` | Set tenant branding |
| `multitenant_GetBranding()` | Get tenant branding |
| `multitenant_UpdateStatus()` | Update tenant status |
