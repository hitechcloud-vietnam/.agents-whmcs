# WHMCS Multi-Tenant Module Skill
# Version: 1.0 | Updated: 2026-05-28

## Purpose

Guide for building multi-tenant WHMCS modules.

## When to Use

- Reseller modules
- White-label solutions
- Multi-brand hosting platforms

## Multi-Tenant Patterns

```php
<?php
class TenantManager {
    private int $tenantId;

    public function __construct(int $tenantId = null) {
        $this->tenantId = $tenantId ?? $this->resolveCurrentTenant();
    }

    private function resolveCurrentTenant(): int {
        // Resolve from reseller account
        if (isReseller()) {
            return Capsule::table('mod_tenant_resellers')
                ->where('user_id', $_SESSION['uid'])
                ->value('tenant_id');
        }
        return 0; // Master tenant
    }

    public function getTenantData(): array {
        return Capsule::table('mod_tenants')
            ->where('id', $this->tenantId)
            ->first();
    }

    public function query(string $table, array $conditions = []): object {
        return Capsule::table($table)
            ->where('tenant_id', $this->tenantId)
            ->where($conditions);
    }

    public function getConfig(string $key, $default = null) {
        $value = Capsule::table('mod_tenant_settings')
            ->where('tenant_id', $this->tenantId)
            ->where('setting_key', $key)
            ->value('setting_value');

        return $value ?? $default;
    }

    public function setConfig(string $key, $value): void {
        Capsule::table('mod_tenant_settings')
            ->updateOrInsert(
                ['tenant_id' => $this->tenantId, 'setting_key' => $key],
                ['setting_value' => $value]
            );
    }

    public function applyBranding(): void {
        $tenant = $this->getTenantData();

        // Override WHMCS branding
        Capsule::table('tblconfiguration')
            ->where('setting', 'CompanyName')
            ->update(['value' => $tenant->brand_name]);

        // Set custom logo path
        $_SESSION['tenant_logo'] = $tenant->logo_url;
    }
}
```

### Branding Hook
```php
add_hook('ClientAreaHeadOutput', 1, function($vars) {
    $tenant = new TenantManager();
    $logo = $tenant->getConfig('logo_url');

    if ($logo) {
        return '<link rel="stylesheet" href="' . $tenant->getConfig('custom_css') . '">';
    }
});
```

---

**Related Skills:**
- whmcs-reseller-module
- whmcs-configuration-management
- whmcs-template-styling
