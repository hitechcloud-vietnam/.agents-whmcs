# WHMCS CDN Module DevKit
# Version: 1.0 | Updated: 2026-05-28

## DevKit Structure

```
devkits/whmcs-cdn-module/
├── cdn.php              # CDN controller
├── lib/
│   ├── CdnProvider.php   # CDN provider integration
│   ├── CacheManager.php  # Cache management
│   └── Purger.php        # Cache purging
└── templates/
    ├── admin.tpl         # Admin templates
    └── client.tpl        # Client templates
```

## CDN Module Template

```php
<?php
/**
 * WHMCS CDN Module: {Module}
 * DevKit Template
 * 
 * Installation: Upload to modules/addons/{module}/
 */

if (!defined("WHMCS")) {
    die("Direct access denied");
}

use WHMCS\Database\Capsule;

/**
 * Config function
 */
function {module}_config(): array {
    return [
        'name' => '{CDN Module}',
        'description' => 'CDN management and optimization',
        'version' => '1.0',
        'author' => '{Author}',
    ];
}

/**
 * Activate
 */
function {module}_activate(): array {
    Capsule::schema()->create('mod_{module}_cdn_zones', function($t) {
        $t->increments('id');
        $t->integer('user_id');
        $t->integer('service_id');
        $t->string('zone_id');
        $t->string('cname');
        $t->string('status');
        $t->bigInteger('bandwidth_used')->default(0);
        $t->bigInteger('requests_total')->default(0);
        $t->timestamp('created_at');
        $t->timestamp('updated_at');
    });
    
    Capsule::schema()->create('mod_{module}_cdn_settings', function($t) {
        $t->increments('id');
        $t->string('setting_name');
        $t->text('setting_value');
        $t->timestamp('updated_at');
    });
    
    Capsule::schema()->create('mod_{module}_cdn_cache_logs', function($t) {
        $t->increments('id');
        $t->integer('zone_id');
        $t->string('path');
        $t->string('action'); // purge, cache, expire
        $t->integer('size');
        $t->timestamp('created_at');
    });
    
    return ['status' => 'success', 'description' => 'Module activated'];
}

/**
 * Deactivate
 */
function {module}_deactivate(): array {
    Capsule::schema()->dropIfExists('mod_{module}_cdn_zones');
    Capsule::schema()->dropIfExists('mod_{module}_cdn_settings');
    Capsule::schema()->dropIfExists('mod_{module}_cdn_cache_logs');
    
    return ['status' => 'success'];
}

/**
 * Output function
 */
function {module}_output(array $vars): void {
    $action = $_REQUEST['action'] ?? 'dashboard';
    
    if ($_SERVER['REQUEST_METHOD'] === 'POST') {
        check_token('WHMCS.admin.default');
    }
    
    switch ($action) {
        case 'zones':
            {module}_manageZones();
            break;
        case 'purge':
            {module}_purgeCache();
            break;
        case 'settings':
            {module}_showSettings();
            break;
        case 'analytics':
            {module}_showAnalytics();
            break;
        default:
            {module}_showDashboard();
    }
}

/**
 * Show Dashboard
 */
function {module}_showDashboard(): void {
    $stats = [
        'total_zones' => Capsule::table('mod_{module}_cdn_zones')->count(),
        'active_zones' => Capsule::table('mod_{module}_cdn_zones')
            ->where('status', 'active')->count(),
        'total_bandwidth' => Capsule::table('mod_{module}_cdn_zones')
            ->sum('bandwidth_used'),
        'total_requests' => Capsule::table('mod_{module}_cdn_zones')
            ->sum('requests_total'),
    ];
    
    echo <<<HTML
<div class="cdn-module">
    <div class="row">
        <div class="col-md-12">
            <div class="panel panel-default">
                <div class="panel-heading">
                    <h3 class="panel-title">CDN Management Dashboard</h3>
                </div>
                <div class="panel-body">
                    <div class="row">
                        <div class="col-md-3">
                            <div class="stat-box">
                                <div class="stat-value">{$stats['total_zones']}</div>
                                <div class="stat-label">Total Zones</div>
                            </div>
                        </div>
                        <div class="col-md-3">
                            <div class="stat-box">
                                <div class="stat-value">{$stats['active_zones']}</div>
                                <div class="stat-label">Active Zones</div>
                            </div>
                        </div>
                        <div class="col-md-3">
                            <div class="stat-box">
                                <div class="stat-value">{($stats['total_bandwidth'] / 1073741824)|number_format:2} GB</div>
                                <div class="stat-label">Bandwidth Used</div>
                            </div>
                        </div>
                        <div class="col-md-3">
                            <div class="stat-box">
                                <div class="stat-value">{$stats['total_requests']}|number_format</div>
                                <div class="stat-label">Total Requests</div>
                            </div>
                        </div>
                    </div>
                </div>
            </div>
        </div>
    </div>
    
    <div class="row">
        <div class="col-md-12">
            <div class="btn-group">
                <a href="?module={module}&action=zones" class="btn btn-primary">
                    <i class="fa fa-globe"></i> Manage Zones
                </a>
                <a href="?module={module}&action=purge" class="btn btn-warning">
                    <i class="fa fa-trash"></i> Purge Cache
                </a>
                <a href="?module={module}&action=settings" class="btn btn-default">
                    <i class="fa fa-cog"></i> Settings
                </a>
                <a href="?module={module}&action=analytics" class="btn btn-default">
                    <i class="fa fa-chart-bar"></i> Analytics
                </a>
            </div>
        </div>
    </div>
</div>
HTML;
}
```

## CDN Provider Class

```php
<?php
namespace {Module};

use WHMCS\Database\Capsule;

class CdnProvider {
    
    private string $apiKey;
    private string $apiSecret;
    private string $baseUrl;
    
    public function __construct() {
        $settings = Capsule::table('mod_{module}_cdn_settings')->get();
        foreach ($settings as $setting) {
            $this->{$setting->setting_name} = $setting->setting_value;
        }
    }
    
    public function createZone(string $origin, string $subdomain): array {
        $zoneData = [
            'name' => $subdomain,
            'type' => 'full',
            'origin' => $origin,
            'ttl' => 3600,
            'automatic_https_rewrites' => true,
        ];
        
        $response = $this->apiRequest('POST', '/zones', $zoneData);
        
        return [
            'zone_id' => $response['id'],
            'cname' => $response['name'] . '.' . $this->domain,
            'status' => $response['status'],
        ];
    }
    
    public function getZone(string $zoneId): array {
        return $this->apiRequest('GET', "/zones/{$zoneId}");
    }
    
    public function deleteZone(string $zoneId): bool {
        return $this->apiRequest('DELETE', "/zones/{$zoneId}");
    }
    
    public function purgeCache(string $zoneId, array $files = []): array {
        if (empty($files)) {
            // Purge everything
            return $this->apiRequest('POST', "/zones/{$zoneId}/cache_purge", [
                'purge_everything' => true,
            ]);
        }
        
        return $this->apiRequest('POST', "/zones/{$zoneId}/cache_purge", [
            'files' => $files,
        ]);
    }
    
    public function getAnalytics(string $zoneId, string $period = '30d'): array {
        return $this->apiRequest('GET', "/zones/{$zoneId}/analytics/dashboard", [
            'since' => '-' . $period,
            'until' => 'now',
        ]);
    }
    
    public function setCacheRules(string $zoneId, array $rules): array {
        return $this->apiRequest('PUT', "/zones/{$zoneId}/cache_rules", [
            'rules' => $rules,
        ]);
    }
    
    public function enableSSL(string $zoneId, string $mode = 'flexible'): array {
        return $this->apiRequest('PATCH', "/zones/{$zoneId}/settings/ssl_mode", [
            'value' => $mode,
        ]);
    }
    
    public function getZoneStats(string $zoneId): array {
        $response = $this->apiRequest('GET', "/zones/{$zoneId}");
        
        return [
            'bandwidth' => $response['bandwidth'] ?? 0,
            'requests' => $response['requests'] ?? 0,
            'hits' => $response['cache_hit_rate'] ?? 0,
            'status' => $response['status'] ?? 'unknown',
        ];
    }
    
    private function apiRequest(string $method, string $endpoint, array $data = []): array {
        $ch = curl_init();
        
        $url = $this->baseUrl . $endpoint;
        
        $headers = [
            'Authorization: Bearer ' . $this->apiKey,
            'Content-Type: application/json',
        ];
        
        curl_setopt_array($ch, [
            CURLOPT_URL => $url,
            CURLOPT_RETURNTRANSFER => true,
            CURLOPT_TIMEOUT => 30,
            CURLOPT_HTTPHEADER => $headers,
        ]);
        
        if ($method === 'POST' || $method === 'PUT' || $method === 'PATCH') {
            curl_setopt($ch, CURLOPT_POST, true);
            curl_setopt($ch, CURLOPT_POSTFIELDS, json_encode($data));
        } elseif ($method === 'DELETE') {
            curl_setopt($ch, CURLOPT_CUSTOMREQUEST, 'DELETE');
        }
        
        $response = curl_exec($ch);
        $httpCode = curl_getinfo($ch, CURLINFO_HTTP_CODE);
        curl_close($ch);
        
        if ($httpCode >= 400) {
            throw new \Exception("CDN API Error: HTTP {$httpCode}");
        }
        
        return json_decode($response, true) ?? [];
    }
}
```

## Cache Manager Class

```php
<?php
namespace {Module};

use WHMCS\Database\Capsule;

class CacheManager {
    
    private CdnProvider $provider;
    
    public function __construct() {
        $this->provider = new CdnProvider();
    }
    
    public function purgeZone(int $zoneId, string $type = 'everything'): array {
        $zone = Capsule::table('mod_{module}_cdn_zones')
            ->where('id', $zoneId)
            ->first();
        
        if (!$zone) {
            throw new \Exception('Zone not found');
        }
        
        $result = $this->provider->purgeCache($zone->zone_id);
        
        // Log the purge
        Capsule::table('mod_{module}_cdn_cache_logs')->insert([
            'zone_id' => $zoneId,
            'path' => $type,
            'action' => 'purge',
            'created_at' => date('Y-m-d H:i:s'),
        ]);
        
        logActivity("{Module}: Cache purged for zone #{$zoneId}");
        
        return $result;
    }
    
    public function purgeFile(int $zoneId, string $filePath): array {
        $zone = Capsule::table('mod_{module}_cdn_zones')
            ->where('id', $zoneId)
            ->first();
        
        if (!$zone) {
            throw new \Exception('Zone not found');
        }
        
        $result = $this->provider->purgeCache($zone->zone_id, [$filePath]);
        
        Capsule::table('mod_{module}_cdn_cache_logs')->insert([
            'zone_id' => $zoneId,
            'path' => $filePath,
            'action' => 'purge',
            'created_at' => date('Y-m-d H:i:s'),
        ]);
        
        return $result;
    }
    
    public function setCacheLevel(int $zoneId, string $level): bool {
        $rules = match($level) {
            'aggressive' => [
                ['path' => '*', 'ttl' => 604800, 'priority' => 1],
            ],
            'standard' => [
                ['path' => '*.html', 'ttl' => 3600, 'priority' => 1],
                ['path' => '*.css', 'ttl' => 86400, 'priority' => 2],
                ['path' => '*.js', 'ttl' => 86400, 'priority' => 3],
            ],
            'basic' => [
                ['path' => '*.jpg', 'ttl' => 3600, 'priority' => 1],
                ['path' => '*.png', 'ttl' => 3600, 'priority' => 2],
            ],
            default => [],
        };
        
        $zone = Capsule::table('mod_{module}_cdn_zones')
            ->where('id', $zoneId)
            ->first();
        
        if (!$zone) {
            return false;
        }
        
        $this->provider->setCacheRules($zone->zone_id, $rules);
        
        return true;
    }
    
    public function warmCache(int $zoneId, array $urls): array {
        $results = [];
        
        foreach ($urls as $url) {
            $ch = curl_init();
            curl_setopt_array($ch, [
                CURLOPT_URL => $url,
                CURLOPT_RETURNTRANSFER => true,
                CURLOPT_TIMEOUT => 10,
            ]);
            
            $response = curl_exec($ch);
            $httpCode = curl_getinfo($ch, CURLINFO_HTTP_CODE);
            curl_close($ch);
            
            $results[] = [
                'url' => $url,
                'status' => $httpCode,
                'cached' => $httpCode >= 200 && $httpCode < 400,
            ];
        }
        
        return $results;
    }
}
```

## Service Hooks

```php
<?php
/**
 * CDN Module Hooks
 */

if (!defined("WHMCS")) {
    die("Direct access denied");
}

// Create CDN zone when service is activated
add_hook('AfterModuleCreate', 1, function(array $vars) {
    $service = Capsule::table('tblhosting')
        ->where('id', $vars['serviceid'])
        ->first();
    
    if (!$service) {
        return;
    }
    
    // Check if service should have CDN
    $product = Capsule::table('tblproducts')
        ->where('id', $service->packageid)
        ->first();
    
    if ($product && $product->{module}_enabled) {
        $provider = new \{Module}\CdnProvider();
        $result = $provider->createZone(
            $service->domain,
            'cdn-' . $service->domain
        );
        
        Capsule::table('mod_{module}_cdn_zones')->insert([
            'user_id' => $service->userid,
            'service_id' => $service->id,
            'zone_id' => $result['zone_id'],
            'cname' => $result['cname'],
            'status' => $result['status'],
            'created_at' => date('Y-m-d H:i:s'),
            'updated_at' => date('Y-m-d H:i:s'),
        ]);
        
        logActivity("{Module}: CDN zone created for service #{$service->id}");
    }
});

// Delete CDN zone when service is terminated
add_hook('AfterModuleTerminate', 1, function(array $vars) {
    $zone = Capsule::table('mod_{module}_cdn_zones')
        ->where('service_id', $vars['serviceid'])
        ->first();
    
    if ($zone) {
        $provider = new \{Module}\CdnProvider();
        $provider->deleteZone($zone->zone_id);
        
        Capsule::table('mod_{module}_cdn_zones')
            ->where('id', $zone->id)
            ->delete();
        
        logActivity("{Module}: CDN zone deleted for service #{$vars['serviceid']}");
    }
});
```

## Checklist

```
Pre-Dev:
□ Identify CDN provider (CloudFlare, Fastly, etc.)
□ Get API documentation
□ Plan zone creation workflow
□ Design cache management
□ Plan analytics integration

Development:
□ Create CDN tables
□ Implement CdnProvider class
□ Implement CacheManager class
□ Implement Purger class
□ Add zone creation
□ Add zone deletion
□ Add cache purging
□ Add cache rules configuration
□ Build admin interface
□ Create zone management
□ Add analytics display
□ Implement service hooks
□ Create client area display

Testing:
□ Test zone creation
□ Test cache purging
□ Verify CDN integration
□ Test analytics display
□ Test service hooks
□ Test SSL configuration
□ Verify performance improvement
```