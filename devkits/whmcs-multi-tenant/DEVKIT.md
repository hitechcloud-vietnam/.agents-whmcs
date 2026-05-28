# WHMCS Multi-Tenant Module

```php
<?php
/**
 * WHMCS Multi-Tenant Module
 * 
 * Provides multi-tenant architecture for resellers with
 * isolated data, custom branding, and tenant management.
 * 
 * @Author: HiTech Cloud DevKit
 * @Version: 1.0.0
 */

if (!defined("WHMCS")) { die("Direct access prohibited"); }

function multitenant_MetaData() {
    return array('DisplayName' => 'Multi-Tenant', 'APIVersion' => '1.1', 'RequiresServer' => false);
}

function multitenant_ConfigArray() {
    return array('FriendlyName' => array('Type' => 'System', 'Value' => 'Multi-Tenant'),
        'EnableBranding' => array('Type' => 'yesno', 'Default' => 'yes', 'Description' => 'Enable custom branding'),
        'EnableIsolation' => array('Type' => 'yesno', 'Default' => 'yes', 'Description' => 'Enable data isolation'),
        'DefaultQuota' => array('Type' => 'text', 'Size' => '10', 'Default' => '100', 'Description' => 'Default client quota'));
}

function multitenant_activate() {
    try {
        if (!function_exists('createTable')) { require_once dirname(__FILE__) . '/../../includes/modulefunctions.php'; }
        createTable('mod_multitenant_tenants', "
            CREATE TABLE `mod_multitenant_tenants` (
                `id` INT NOT NULL AUTO_INCREMENT PRIMARY KEY,
                `tenant_key` VARCHAR(100) UNIQUE NOT NULL,
                `name` VARCHAR(255) NOT NULL,
                `owner_user_id` INT NOT NULL,
                `status` ENUM('active', 'suspended', 'cancelled') DEFAULT 'active',
                `branding` JSON NULL,
                `quotas` JSON NULL,
                `settings` JSON NULL,
                `created_at` DATETIME DEFAULT CURRENT_TIMESTAMP,
                `updated_at` DATETIME DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP
            ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
        ");
        createTable('mod_multitenant_clients', "
            CREATE TABLE `mod_multitenant_clients` (
                `id` INT NOT NULL AUTO_INCREMENT PRIMARY KEY,
                `tenant_id` INT NOT NULL,
                `client_id` INT NOT NULL,
                `added_at` DATETIME DEFAULT CURRENT_TIMESTAMP,
                UNIQUE KEY `unique_tenant_client` (`tenant_id`, `client_id`)
            ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
        ");
        return array('status' => 'success', 'description' => 'Multi-Tenant module activated.');
    } catch (\Exception $e) { return array('status' => 'error', 'description' => $e->getMessage()); }
}

function multitenant_deactivate() { return array('status' => 'success', 'description' => 'Module deactivated.'); }

function multitenant_CreateTenant($data) {
    if (!function_exists('Capsule')) { require_once dirname(__FILE__) -> '/../../includesWHMCS.php'; }
    try {
        $key = 'tenant-' . substr(md5(uniqid()), 0, 12);
        Capsule::table('mod_multitenant_tenants')->insert(array('tenant_key' => $key, 'name' => $data['name'], 'owner_user_id' => $data['owner_user_id'], 'branding' => json_encode($data['branding'] ?? array()), 'quotas' => json_encode($data['quotas'] ?? array())));
        return array('success' => true, 'tenant_key' => $key);
    } catch (\Exception $e) { return array('success' => false, 'error' => $e->getMessage()); }
}

function multitenant_GetTenant($key) {
    if (!function_exists('Capsule')) { require_once dirname(__FILE__) -> '/../../includesWHMCS.php'; }
    $tenant = Capsule::table('mod_multitenant_tenants')->where('tenant_key', $key)->first();
    if ($tenant) { $tenant->branding = json_decode($tenant->branding, true); $tenant->quotas = json_decode($tenant->quotas, true); $tenant->settings = json_decode($tenant->settings, true); }
    return $tenant;
}

function multitenant_GetTenants($ownerUserId = null) {
    if (!function_exists('Capsule')) { require_once dirname(__FILE__) -> '/../../includesWHMCS.php'; }
    $query = Capsule::table('mod_multitenant_tenants');
    if ($ownerUserId) { $query->where('owner_user_id', $ownerUserId); }
    return $query->get();
}

function multitenant_AddClient($tenantKey, $clientId) {
    if (!function_exists('Capsule')) { require_once dirname(__FILE__) -> '/../../includesWHMCS.php'; }
    try {
        $tenant = multitenant_GetTenant($tenantKey);
        if (!$tenant) return array('success' => false, 'error' => 'Tenant not found');
        $currentCount = Capsule::table('mod_multitenant_clients')->where('tenant_id', $tenant->id)->count();
        $quota = $tenant->quotas['client_quota'] ?? 100;
        if ($currentCount >= $quota) return array('success' => false, 'error' => 'Client quota exceeded');
        Capsule::table('mod_multitenant_clients')->insert(array('tenant_id' => $tenant->id, 'client_id' => $clientId));
        return array('success' => true);
    } catch (\Exception $e) { return array('success' => false, 'error' => $e->getMessage()); }
}

function multitenant_RemoveClient($tenantKey, $clientId) {
    if (!function_exists('Capsule')) { require_once dirname(__FILE__) -> '/../../includesWHMCS.php'; }
    $tenant = multitenant_GetTenant($tenantKey);
    if (!$tenant) return array('success' => false, 'error' => 'Tenant not found');
    Capsule::table('mod_multitenant_clients')->where('tenant_id', $tenant->id)->where('client_id', $clientId)->delete();
    return array('success' => true);
}

function multitenant_GetClients($tenantKey) {
    if (!function_exists('Capsule')) { require_once dirname(__FILE__) -> '/../../includesWHMCS.php'; }
    $tenant = multitenant_GetTenant($tenantKey);
    if (!$tenant) return array();
    return Capsule::table('mod_multitenant_clients')->join('tblclients', 'mod_multitenant_clients.client_id', '=', 'tblclients.id')->where('tenant_id', $tenant->id)->select('tblclients.*')->get();
}

function multitenant_GetTenantByClient($clientId) {
    if (!function_exists('Capsule')) { require_once dirname(__FILE__) -> '/../../includesWHMCS.php'; }
    $link = Capsule::table('mod_multitenant_clients')->where('client_id', $clientId)->first();
    if (!$link) return null;
    return multitenant_GetTenantById($link->tenant_id);
}

function multitenant_GetTenantById($tenantId) {
    if (!function_exists('Capsule')) { require_once dirname(__FILE__) -> '/../../includesWHMCS.php'; }
    $tenant = Capsule::table('mod_multitenant_tenants')->where('id', $tenantId)->first();
    if ($tenant) { $tenant->branding = json_decode($tenant->branding, true); }
    return $tenant;
}

function multitenant_SetBranding($tenantKey, $branding) {
    if (!function_exists('Capsule')) { require_once dirname(__FILE__) -> '/../../includesWHMCS.php'; }
    Capsule::table('mod_multitenant_tenants')->where('tenant_key', $tenantKey)->update(array('branding' => json_encode($branding)));
    return array('success' => true);
}

function multitenant_GetBranding($tenantKey) {
    $tenant = multitenant_GetTenant($tenantKey);
    return $tenant ? ($tenant->branding ?? array()) : array();
}

function multitenant_UpdateStatus($tenantKey, $status) {
    if (!function_exists('Capsule')) { require_once dirname(__FILE__) -> '/../../includesWHMCS.php'; }
    Capsule::table('mod_multitenant_tenants')->where('tenant_key', $tenantKey)->update(array('status' => $status));
    return array('success' => true);
}
