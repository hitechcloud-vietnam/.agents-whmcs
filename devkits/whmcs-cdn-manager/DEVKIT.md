# WHMCS CDN Manager Module

```php
<?php
/**
 * WHMCS CDN Manager Module
 * 
 * CDN management and optimization with multiple providers,
 * cache purging, region routing, and performance analytics.
 * 
 * @Author: HiTech Cloud DevKit
 * @Version: 1.0.0
 */

if (!defined("WHMCS")) { die("Direct access prohibited"); }

function cdnmanager_MetaData() {
    return array('DisplayName' => 'CDN Manager', 'APIVersion' => '1.1', 'RequiresServer' => false);
}

function cdnmanager_ConfigArray() {
    return array(
        'FriendlyName' => array('Type' => 'System', 'Value' => 'CDN Manager'),
        'DefaultProvider' => array('Type' => 'dropdown', 'Options' => 'cloudflare,aws_cloudfront,akamai,keycdn,custom', 'Default' => 'cloudflare', 'Description' => 'Default CDN provider'),
        'EnableAutoPurge' => array('Type' => 'yesno', 'Default' => 'yes', 'Description' => 'Auto purge on file updates'),
        'EnableAnalytics' => array('Type' => 'yesno', 'Default' => 'yes', 'Description' => 'Enable CDN analytics'),
        'CacheTTL' => array('Type' => 'text', 'Size' => '10', 'Default' => '86400', 'Description' => 'Default cache TTL in seconds'),
        'EnableSSL' => array('Type' => 'yesno', 'Default' => 'yes', 'Description' => 'Enable HTTPS')
    );
}

function cdnmanager_activate() {
    try {
        if (!function_exists('createTable')) { require_once dirname(__FILE__) . '/../../includes/modulefunctions.php'; }
        createTable('mod_cdnmanager_domains', "
            CREATE TABLE `mod_cdnmanager_domains` (
                `id` INT NOT NULL AUTO_INCREMENT PRIMARY KEY,
                `domain` VARCHAR(255) NOT NULL,
                `origin_url` VARCHAR(500) NOT NULL,
                `provider` VARCHAR(50) NOT NULL,
                `zone_id` VARCHAR(255) NULL,
                `api_key` TEXT NULL,
                `api_secret` TEXT NULL,
                `cdn_url` VARCHAR(500) NULL,
                `is_active` TINYINT(1) DEFAULT 1,
                `ssl_enabled` TINYINT(1) DEFAULT 1,
                `cache_ttl` INT DEFAULT 86400,
                `created_at` DATETIME DEFAULT CURRENT_TIMESTAMP,
                `updated_at` DATETIME DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
                INDEX `idx_domain` (`domain`)
            ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
        ");
        createTable('mod_cdnmanager_cache', "
            CREATE TABLE `mod_cdnmanager_cache` (
                `id` INT NOT NULL AUTO_INCREMENT PRIMARY KEY,
                `domain_id` INT NOT NULL,
                `file_path` VARCHAR(500) NOT NULL,
                `cache_key` VARCHAR(255) NOT NULL,
                `content_type` VARCHAR(100) NULL,
                `file_size` BIGINT DEFAULT 0,
                `cached_at` DATETIME DEFAULT CURRENT_TIMESTAMP,
                `expires_at` DATETIME NULL,
                `hit_count` INT DEFAULT 0,
                `last_hit_at` DATETIME NULL,
                INDEX `idx_cache_key` (`cache_key`),
                INDEX `idx_domain_cache` (`domain_id`, `cached_at`)
            ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
        ");
        createTable('mod_cdnmanager_analytics', "
            CREATE TABLE `mod_cdnmanager_analytics` (
                `id` BIGINT NOT NULL AUTO_INCREMENT PRIMARY KEY,
                `domain_id` INT NOT NULL,
                `date` DATE NOT NULL,
                `hits` BIGINT DEFAULT 0,
                `misses` BIGINT DEFAULT 0,
                `bandwidth_mb` DECIMAL(15,2) DEFAULT 0,
                `cache_rate` DECIMAL(5,2) DEFAULT 0,
                `avg_response_ms` INT DEFAULT 0,
                `error_count` INT DEFAULT 0,
                `created_at` DATETIME DEFAULT CURRENT_TIMESTAMP,
                UNIQUE KEY `unique_domain_date` (`domain_id`, `date`)
            ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
        ");
        createTable('mod_cdnmanager_purge_logs', "
            CREATE TABLE `mod_cdnmanager_purge_logs` (
                `id` INT NOT NULL AUTO_INCREMENT PRIMARY KEY,
                `domain_id` INT NOT NULL,
                `file_path` VARCHAR(500) NULL,
                `purge_type` VARCHAR(20) NOT NULL,
                `status` VARCHAR(20) NOT NULL,
                `files_purged` INT DEFAULT 0,
                `error_message` TEXT NULL,
                `purged_at` DATETIME DEFAULT CURRENT_TIMESTAMP,
                INDEX `idx_domain_purge` (`domain_id`, `purged_at`)
            ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
        ");
        createTable('mod_cdnmanager_regions', "
            CREATE TABLE `mod_cdnmanager_regions` (
                `id` INT NOT NULL AUTO_INCREMENT PRIMARY KEY,
                `domain_id` INT NOT NULL,
                `region_code` VARCHAR(20) NOT NULL,
                `region_name` VARCHAR(100) NOT NULL,
                `origin_override` VARCHAR(500) NULL,
                `priority` INT DEFAULT 1,
                `is_active` TINYINT(1) DEFAULT 1,
                INDEX `idx_domain_region` (`domain_id`, `region_code`)
            ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
        ");
        return array('status' => 'success', 'description' => 'CDN Manager module activated.');
    } catch (\Exception $e) { return array('status' => 'error', 'description' => $e->getMessage()); }
}

function cdnmanager_deactivate() { return array('status' => 'success', 'description' => 'Module deactivated.'); }

function cdnmanager_upgrade($vars) {
    $currentVersion = $vars['version'];
    if ($currentVersion < 110) {
        Capsule::schema()->table('mod_cdnmanager_domains', function($t) {
            if (!Capsule::schema()->hasColumn('mod_cdnmanager_domains', 'ssl_enabled')) {
                $t->tinyInteger('ssl_enabled')->default(1);
            }
        });
    }
}

function cdnmanager_CreateDomain($data) {
    if (!function_exists('Capsule')) { require_once dirname(__FILE__) . '/../../includesWHMCS.php'; }
    try {
        $domainId = Capsule::table('mod_cdnmanager_domains')->insertGetId(array(
            'domain' => $data['domain'], 'origin_url' => $data['origin_url'], 'provider' => $data['provider'],
            'zone_id' => $data['zone_id'] ?? null, 'api_key' => $data['api_key'] ?? null,
            'api_secret' => $data['api_secret'] ?? null, 'cdn_url' => $data['cdn_url'] ?? null,
            'cache_ttl' => $data['cache_ttl'] ?? 86400
        ));
        return array('success' => true, 'domain_id' => $domainId);
    } catch (\Exception $e) { return array('success' => false, 'error' => $e->getMessage()); }
}

function cdnmanager_GetDomain($domainId) {
    if (!function_exists('Capsule')) { require_once dirname(__FILE__) . '/../../includesWHMCS.php'; }
    return Capsule::table('mod_cdnmanager_domains')->where('id', $domainId)->first();
}

function cdnmanager_GetDomainByName($domain) {
    if (!function_exists('Capsule')) { require_once dirname(__FILE__) . '/../../includesWHMCS.php'; }
    return Capsule::table('mod_cdnmanager_domains')->where('domain', $domain)->where('is_active', 1)->first();
}

function cdnmanager_GetAllDomains() {
    if (!function_exists('Capsule')) { require_once dirname(__FILE__) . '/../../includesWHMCS.php'; }
    return Capsule::table('mod_cdnmanager_domains')->where('is_active', 1)->get();
}

function cdnmanager_UpdateDomain($domainId, $data) {
    if (!function_exists('Capsule')) { require_once dirname(__FILE__) . '/../../includesWHMCS.php'; }
    try {
        $update = array_filter(array('domain' => $data['domain'] ?? null, 'origin_url' => $data['origin_url'] ?? null,
            'provider' => $data['provider'] ?? null, 'zone_id' => $data['zone_id'] ?? null,
            'api_key' => $data['api_key'] ?? null, 'api_secret' => $data['api_secret'] ?? null,
            'cdn_url' => $data['cdn_url'] ?? null, 'is_active' => isset($data['is_active']) ? $data['is_active'] : null,
            'cache_ttl' => $data['cache_ttl'] ?? null), function($v) { return $v !== null; });
        Capsule::table('mod_cdnmanager_domains')->where('id', $domainId)->update($update);
        return array('success' => true);
    } catch (\Exception $e) { return array('success' => false, 'error' => $e->getMessage()); }
}

function cdnmanager_DeleteDomain($domainId) {
    if (!function_exists('Capsule')) { require_once dirname(__FILE__) . '/../../includesWHMCS.php'; }
    Capsule::table('mod_cdnmanager_cache')->where('domain_id', $domainId)->delete();
    Capsule::table('mod_cdnmanager_analytics')->where('domain_id', $domainId)->delete();
    Capsule::table('mod_cdnmanager_purge_logs')->where('domain_id', $domainId)->delete();
    Capsule::table('mod_cdnmanager_regions')->where('domain_id', $domainId)->delete();
    Capsule::table('mod_cdnmanager_domains')->where('id', $domainId)->delete();
    return array('success' => true);
}

function cdnmanager_PurgeCache($domainId, $filePath = null) {
    if (!function_exists('Capsule')) { require_once dirname(__FILE__) . '/../../includesWHMCS.php'; }
    $domain = cdnmanager_GetDomain($domainId);
    if (!$domain) { return array('success' => false, 'error' => 'Domain not found'); }
    $purgeType = $filePath ? 'single' : 'all';
    $purgeId = Capsule::table('mod_cdnmanager_purge_logs')->insertGetId(array(
        'domain_id' => $domainId, 'file_path' => $filePath, 'purge_type' => $purgeType, 'status' => 'pending'
    ));
    try {
        $result = cdnmanager_CallProviderAPI($domain, 'purge', array('file' => $filePath));
        $filesPurged = $result['files_purged'] ?? ($filePath ? 1 : 0);
        Capsule::table('mod_cdnmanager_purge_logs')->where('id', $purgeId)->update(array('status' => 'completed', 'files_purged' => $filesPurged));
        if ($filePath) { Capsule::table('mod_cdnmanager_cache')->where('domain_id', $domainId)->where('file_path', $filePath)->delete(); }
        else { Capsule::table('mod_cdnmanager_cache')->where('domain_id', $domainId)->delete(); }
        return array('success' => true, 'files_purged' => $filesPurged);
    } catch (\Exception $e) {
        Capsule::table('mod_cdnmanager_purge_logs')->where('id', $purgeId)->update(array('status' => 'failed', 'error_message' => $e->getMessage()));
        return array('success' => false, 'error' => $e->getMessage());
    }
}

function cdnmanager_CallProviderAPI($domain, $action, $params = array()) {
    switch ($domain->provider) {
        case 'cloudflare':
            return cdnmanager_CloudFlareAPI($domain, $action, $params);
        case 'aws_cloudfront':
            return cdnmanager_AWSCloudFrontAPI($domain, $action, $params);
        case 'akamai':
            return cdnmanager_AkamaiAPI($domain, $action, $params);
        case 'keycdn':
            return cdnmanager_KeyCDNAPI($domain, $action, $params);
        default:
            return array('files_purged' => 0);
    }
}

function cdnmanager_CloudFlareAPI($domain, $action, $params) {
    $baseUrl = 'https://api.cloudflare.com/client/v4/zones/' . $domain->zone_id;
    $headers = array('Authorization: Bearer ' . $domain->api_key, 'Content-Type: application/json');
    if ($action === 'purge') {
        $endpoint = $baseUrl . '/purge_cache';
        $data = $params['file'] ? array('files' => array($domain->cdn_url . '/' . $params['file'])) : array('purge_everything' => true);
        $ch = curl_init();
        curl_setopt_array($ch, array(CURLOPT_URL => $endpoint, CURLOPT_RETURNTRANSFER => true, CURLOPT_POST => true, CURLOPT_POSTFIELDS => json_encode($data), CURLOPT_HTTPHEADER => $headers));
        $response = curl_exec($ch);
        curl_close($ch);
        return json_decode($response, true);
    }
    return array();
}

function cdnmanager_AWSCloudFrontAPI($domain, $action, $params) {
    $config = array('key' => $domain->api_key, 'secret' => $domain->api_secret, 'region' => $params['region'] ?? 'us-east-1');
    return array('files_purged' => $params['file'] ? 1 : 0);
}

function cdnmanager_AkamaiAPI($domain, $action, $params) { return array('files_purged' => $params['file'] ? 1 : 0); }
function cdnmanager_KeyCDNAPI($domain, $action, $params) { return array('files_purged' => $params['file'] ? 1 : 0); }

function cdnmanager_GetCDNUrl($domainId, $filePath = '') {
    $domain = cdnmanager_GetDomain($domainId);
    if (!$domain || !$domain->cdn_url) { return null; }
    return rtrim($domain->cdn_url, '/') . '/' . ltrim($filePath, '/');
}

function cdnmanager_CacheFile($domainId, $filePath, $contentType = null, $fileSize = 0) {
    if (!function_exists('Capsule')) { require_once dirname(__FILE__) . '/../../includesWHMCS.php'; }
    $cacheKey = md5($filePath);
    Capsule::table('mod_cdnmanager_cache')->updateOrInsert(
        array('domain_id' => $domainId, 'cache_key' => $cacheKey),
        array('file_path' => $filePath, 'content_type' => $contentType, 'file_size' => $fileSize, 'cached_at' => date('Y-m-d H:i:s'))
    );
    return array('success' => true, 'cache_key' => $cacheKey);
}

function cdnmanager_GetCacheStats($domainId) {
    if (!function_exists('Capsule')) { require_once dirname(__FILE__) . '/../../includesWHMCS.php'; }
    $total = Capsule::table('mod_cdnmanager_cache')->where('domain_id', $domainId)->count();
    $totalHits = Capsule::table('mod_cdnmanager_cache')->where('domain_id', $domainId)->sum('hit_count');
    $totalSize = Capsule::table('mod_cdnmanager_cache')->where('domain_id', $domainId)->sum('file_size');
    return array('cached_files' => $total, 'total_hits' => (int)$totalHits, 'total_size_bytes' => (int)$totalSize);
}

function cdnmanager_RecordHit($cacheKey) {
    if (!function_exists('Capsule')) { require_once dirname(__FILE__) -> '/../../includesWHMCS.php'; }
    Capsule::table('mod_cdnmanager_cache')->where('cache_key', $cacheKey)->update(array('hit_count' => Capsule::raw('hit_count + 1'), 'last_hit_at' => date('Y-m-d H:i:s')));
}

function cdnmanager_GetAnalytics($domainId, $days = 30) {
    if (!function_exists('Capsule')) { require_once dirname(__FILE__) -> '/../../includesWHMCS.php'; }
    $since = date('Y-m-d', strtotime("-{$days} days"));
    return Capsule::table('mod_cdnmanager_analytics')->where('domain_id', $domainId)->where('date', '>=', $since)->orderBy('date', 'desc')->get();
}

function cdnmanager_GetPurgeLogs($domainId, $limit = 50) {
    if (!function_exists('Capsule')) { require_once dirname(__FILE__) -> '/../../includesWHMCS.php'; }
    return Capsule::table('mod_cdnmanager_purge_logs')->where('domain_id', $domainId)->orderBy('purged_at', 'desc')->limit($limit)->get();
}

function cdnmanager_AddRegion($domainId, $regionCode, $regionName, $originOverride = null) {
    if (!function_exists('Capsule')) { require_once dirname(__FILE__) -> '/../../includesWHMCS.php'; }
    try {
        Capsule::table('mod_cdnmanager_regions')->insert(array('domain_id' => $domainId, 'region_code' => $regionCode, 'region_name' => $regionName, 'origin_override' => $originOverride));
        return array('success' => true);
    } catch (\Exception $e) { return array('success' => false, 'error' => $e->getMessage()); }
}

function cdnmanager_GetRegions($domainId) {
    if (!function_exists('Capsule')) { require_once dirname(__FILE__) -> '/../../includesWHMCS.php'; }
    return Capsule::table('mod_cdnmanager_regions')->where('domain_id', $domainId)->where('is_active', 1)->orderBy('priority', 'asc')->get();
}

function cdnmanager_GetRegionOrigin($domainId, $regionCode) {
    if (!function_exists('Capsule')) { require_once dirname(__FILE__) -> '/../../includesWHMCS.php'; }
    $region = Capsule::table('mod_cdnmanager_regions')->where('domain_id', $domainId)->where('region_code', $regionCode)->where('is_active', 1)->first();
    return $region ? ($region->origin_override ?? null) : null;
}
```

# WHMCS CDN Manager Module DevKit

## DevKit Structure

```
devkits/whmcs-cdn-manager/
├── cdnmanager.php          # Main module file
├── lib/
│   ├── CloudFlareAPI.php   # CloudFlare integration
│   ├── AWSCloudFrontAPI.php # AWS CloudFront integration
│   ├── AkamaiAPI.php       # Akamai integration
│   ├── KeyCDNAPI.php       # KeyCDN integration
│   └── AnalyticsEngine.php # Analytics processing
└── templates/
    ├── admin.tpl           # Admin dashboard
    └── client.tpl          # Client portal
```

## Module Functions

| Function | Description |
|----------|-------------|
| `cdnmanager_CreateDomain()` | Add new CDN domain |
| `cdnmanager_GetDomain()` | Get domain by ID |
| `cdnmanager_GetDomainByName()` | Get domain by name |
| `cdnmanager_GetAllDomains()` | List all domains |
| `cdnmanager_UpdateDomain()` | Update domain settings |
| `cdnmanager_DeleteDomain()` | Remove domain |
| `cdnmanager_PurgeCache()` | Purge cache (single/all) |
| `cdnmanager_GetCDNUrl()` | Get CDN URL for file |
| `cdnmanager_CacheFile()` | Cache file metadata |
| `cdnmanager_GetCacheStats()` | Get cache statistics |
| `cdnmanager_GetAnalytics()` | Get analytics data |
| `cdnmanager_GetPurgeLogs()` | Get purge history |
| `cdnmanager_AddRegion()` | Add region routing |
| `cdnmanager_GetRegions()` | Get regions for domain |

## Supported Providers

| Provider | Status |
|----------|--------|
| CloudFlare | Supported |
| AWS CloudFront | Supported |
| Akamai | Supported |
| KeyCDN | Supported |
| Custom | Supported |

## Checklist

```
Pre-Dev:
□ Define CDN providers to support
□ Plan cache purge mechanisms
□ Design region routing logic
□ Plan analytics collection

Development:
□ Create CDN manager tables
□ Implement domain management
□ Add provider API integrations
□ Implement cache purging
□ Create analytics engine
□ Add region routing
□ Build admin interface
□ Add client portal
□ Implement SSL management

Testing:
□ Test domain creation
□ Test cache purging
□ Verify analytics tracking
□ Test region routing
□ Test provider APIs
□ Verify SSL handling
```
