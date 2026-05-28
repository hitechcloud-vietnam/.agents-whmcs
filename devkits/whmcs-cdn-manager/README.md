# WHMCS CDN Manager Module

Comprehensive CDN management and optimization with multi-provider support.

## Features

- Multi-CDN provider support (CloudFlare, AWS CloudFront, Akamai, KeyCDN)
- Cache management and purging
- Performance analytics
- Region-based routing
- SSL management
- Bandwidth tracking

## Installation

1. Copy `cdnmanager.php` to `/path/to/whmcs/modules/addons/cdnmanager/`
2. Activate in WHMCS Admin > System Settings > Module Addons
3. Configure your CDN provider credentials

## Usage

```php
// Create CDN domain
$result = cdnmanager_CreateDomain(array(
    'domain' => 'cdn.example.com',
    'origin_url' => 'https://origin.example.com',
    'provider' => 'cloudflare',
    'zone_id' => 'zone123',
    'api_key' => 'your_api_key',
    'cdn_url' => 'https://cdn.example.com',
    'cache_ttl' => 86400
));

// Get domain
$domain = cdnmanager_GetDomain($domainId);
$domain = cdnmanager_GetDomainByName('cdn.example.com');

// Get all domains
$domains = cdnmanager_GetAllDomains();

// Update domain
cdnmanager_UpdateDomain($domainId, array(
    'cache_ttl' => 172800
));

// Purge single file
$result = cdnmanager_PurgeCache($domainId, '/images/logo.png');

// Purge all
$result = cdnmanager_PurgeCache($domainId);

// Get CDN URL
$url = cdnmanager_GetCDNUrl($domainId, 'images/logo.png');

// Cache file metadata
cdnmanager_CacheFile($domainId, '/images/logo.png', 'image/png', 1024);

// Get cache statistics
$stats = cdnmanager_GetCacheStats($domainId);
// Returns: cached_files, total_hits, total_size_bytes

// Get analytics
$analytics = cdnmanager_GetAnalytics($domainId, 30);

// Get purge logs
$logs = cdnmanager_GetPurgeLogs($domainId, 50);

// Add region routing
cdnmanager_AddRegion($domainId, 'eu-west', 'Europe West', 'https://eu-origin.example.com');

// Get regions
$regions = cdnmanager_GetRegions($domainId);

// Get region-specific origin
$origin = cdnmanager_GetRegionOrigin($domainId, 'eu-west');
```

## Supported Providers

| Provider | API Support |
|----------|-------------|
| CloudFlare | Full API |
| AWS CloudFront | Basic |
| Akamai | Basic |
| KeyCDN | Basic |
| Custom | Configuration |

## Configuration Options

| Option | Type | Default | Description |
|--------|------|---------|-------------|
| DefaultProvider | dropdown | cloudflare | Default CDN provider |
| EnableAutoPurge | yesno | yes | Auto purge on updates |
| EnableAnalytics | yesno | yes | Enable analytics |
| CacheTTL | text | 86400 | Default cache TTL |
| EnableSSL | yesno | yes | Enable HTTPS |

## Database Tables

- `mod_cdnmanager_domains` - CDN domain configurations
- `mod_cdnmanager_cache` - File cache metadata
- `mod_cdnmanager_analytics` - Daily analytics data
- `mod_cdnmanager_purge_logs` - Purge operation logs
- `mod_cdnmanager_regions` - Region routing rules
