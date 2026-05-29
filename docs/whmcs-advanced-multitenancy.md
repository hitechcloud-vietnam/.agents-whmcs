# WHMCS Multi-tenancy

Complete guide to multi-tenant architecture.

## Overview

Implement multi-tenant WHMCS deployments.

## Multi-tenant Architecture

### Tenant Manager

```php
<?php
/**
 * Multi-tenant manager
 */
class TenantManager
{
    private static ?int $currentTenantId = null;
    private static array $tenants = [];
    
    /**
     * Set current tenant
     */
    public static function setCurrentTenant(int $tenantId): void
    {
        self::$currentTenantId = $tenantId;
        
        // Set tenant-specific configuration
        $config = self::getTenantConfig($tenantId);
        self::applyConfig($config);
    }
    
    /**
     * Get current tenant
     */
    public static function getCurrentTenant(): ?int
    {
        return self::$currentTenantId;
    }
    
    /**
     * Get tenant configuration
     */
    public static function getTenantConfig(int $tenantId): array
    {
        if (!isset(self::$tenants[$tenantId])) {
            $tenant = Capsule::table('mod_tenants')
                ->where('id', $tenantId)
                ->first();
            
            self::$tenants[$tenantId] = $tenant ? (array)$tenant : [];
        }
        
        return self::$tenants[$tenantId];
    }
    
    /**
     * Apply tenant configuration
     */
    private static function applyConfig(array $config): void
    {
        // Override global settings with tenant-specific values
        if (isset($config['company_name'])) {
            define('COMPANY_NAME', $config['company_name']);
        }
        
        if (isset($config['email_domain'])) {
            define('EMAIL_DOMAIN', $config['email_domain']);
        }
    }
    
    /**
     * Create tenant
     */
    public static function createTenant(array $data): int
    {
        return Capsule::table('mod_tenants')->insertGetId([
            'name' => $data['name'],
            'slug' => $data['slug'],
            'config' => json_encode($data['config'] ?? []),
            'status' => 'active',
            'created_at' => date('Y-m-d H:i:s'),
        ]);
    }
}
```

## Tenant Isolation

### Isolated Queries

```php
<?php
/**
 * Tenant-aware query builder
 */
class TenantAwareQuery
{
    /**
     * Get clients for current tenant
     */
    public static function getClients(): Builder
    {
        $tenantId = TenantManager::getCurrentTenant();
        
        return Capsule::table('tblclients')
            ->where('tenant_id', $tenantId);
    }
    
    /**
     * Get services for current tenant
     */
    public static function getServices(): Builder
    {
        $tenantId = TenantManager::getCurrentTenant();
        
        return Capsule::table('tblhosting')
            ->join('tblclients', 'tblhosting.userid', '=', 'tblclients.id')
            ->where('tblclients.tenant_id', $tenantId);
    }
    
    /**
     * Get invoices for current tenant
     */
    public static function getInvoices(): Builder
    {
        $tenantId = TenantManager::getCurrentTenant();
        
        return Capsule::table('tblinvoices')
            ->join('tblclients', 'tblinvoices.userid', '=', 'tblclients.id')
            ->where('tblclients.tenant_id', $tenantId);
    }
}
```

## Tenant Routing

### Request Routing

```php
<?php
/**
 * Tenant routing middleware
 */
function handleTenantRouting(): void
{
    $host = $_SERVER['HTTP_HOST'];
    
    // Extract tenant slug from subdomain
    $parts = explode('.', $host);
    $subdomain = $parts[0];
    
    // Ignore www
    if ($subdomain === 'www') {
        return;
    }
    
    // Look up tenant by slug
    $tenant = Capsule::table('mod_tenants')
        ->where('slug', $subdomain)
        ->where('status', 'active')
        ->first();
    
    if ($tenant) {
        TenantManager::setCurrentTenant($tenant->id);
        
        // Set tenant-specific paths
        define('TENANT_ROOT', ROOTDIR . '/tenants/' . $tenant->slug);
        define('TENANT_TEMPLATE', TENANT_ROOT . '/templates');
        
    } else {
        // Tenant not found
        http_response_code(404);
        echo "Tenant not found";
        exit;
    }
}
```

## Tenant-specific Resources

### Theme Isolation

```php
<?php
/**
 * Tenant-specific theming
 */
class TenantTheme
{
    /**
     * Get theme path for current tenant
     */
    public static function getThemePath(string $template): string
    {
        $tenantId = TenantManager::getCurrentTenant();
        $tenantConfig = TenantManager::getTenantConfig($tenantId);
        
        $customTheme = $tenantConfig['custom_theme'] ?? 'default';
        
        $themePath = ROOTDIR . '/templates/' . $customTheme . '/' . $template;
        
        if (file_exists($themePath)) {
            return $themePath;
        }
        
        // Fall back to default theme
        return ROOTDIR . '/templates/six/' . $template;
    }
    
    /**
     * Get tenant logo
     */
    public static function getLogo(): string
    {
        $tenantId = TenantManager::getCurrentTenant();
        $tenantConfig = TenantManager::getTenantConfig($tenantId);
        
        $logo = $tenantConfig['logo_url'] ?? '';
        
        if ($logo && file_exists(ROOTDIR . '/tenants/logos/' . $logo)) {
            return '/tenants/logos/' . $logo;
        }
        
        return '/assets/img/logo.png';
    }
}
```

## Shared Resources

### Resource Pooling

```php
<?php
/**
 * Shared resource management
 */
class SharedResourcePool
{
    /**
     * Get shared API client
     */
    public static function getAPIClient(): APIClient
    {
        static $client = null;
        
        if ($client === null) {
            $client = new APIClient([
                'base_url' => SHARED_API_URL,
                'api_key' => SHARED_API_KEY,
            ]);
        }
        
        return $client;
    }
    
    /**
     * Get shared database connection
     */
    public static function getDatabase(): PDO
    {
        static $pdo = null;
        
        if ($pdo === null) {
            $pdo = new PDO(
                'mysql:host=' . SHARED_DB_HOST . ';dbname=' . SHARED_DB_NAME,
                SHARED_DB_USER,
                SHARED_DB_PASS
            );
        }
        
        return $pdo;
    }
}
```

## Best Practices

1. **Tenant isolation** - Separate data properly
2. **Resource pooling** - Share expensive resources
3. **Tenant detection** - Handle routing correctly
4. **Configuration** - Allow per-tenant settings
5. **Billing** - Track per-tenant usage
6. **Security** - Prevent cross-tenant access

## Related Documentation

- [whmcs-advanced-security.md](whmcs-advanced-security.md)
- [whmcs-advanced-performance.md](whmcs-advanced-performance.md)
