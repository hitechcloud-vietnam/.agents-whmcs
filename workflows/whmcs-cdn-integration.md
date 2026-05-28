# WHMCS CDN Integration Workflow

## Purpose

Integrate Content Delivery Network (CDN) with WHMCS to improve page load times, reduce server load, and enhance user experience. This workflow covers CDN setup, configuration, cache management, and performance optimization.

## Prerequisites

- WHMCS installation (version 8.x recommended)
- CDN provider account (Cloudflare, AWS CloudFront, KeyCDN, etc.)
- SSL certificate configured
- Access to DNS management
- Understanding of static vs dynamic content

## Workflow Steps

### Step 1: CDN Provider Selection

Evaluate providers based on:

| Provider | Performance | Features | Cost | Best For |
|----------|-------------|----------|------|----------|
| Cloudflare | Excellent | Security + CDN | Free tier | Most use cases |
| AWS CloudFront | Excellent | Deep AWS integration | Pay-as-you-go | AWS infrastructure |
| KeyCDN | Good | Simple pricing | Flat rate | Budget-conscious |
| BunnyCDN | Good | Fast, affordable | Flat rate | European traffic |
| Fastly | Excellent | Real-time purging | Premium | Enterprise |

### Step 2: Cloudflare Integration

```bash
#!/bin/bash
# /opt/scripts/setup_cloudflare.sh

# Install Cloudflare origin SSL certificate
mkdir -p /etc/ssl/cloudflare
cd /etc/ssl/cloudflare

# Generate origin certificate (via Cloudflare dashboard)
# Dashboard > SSL/TLS > Origin Server > Create Certificate

# Save certificates
mv origin.pem /etc/ssl/cloudflare/origin.pem
mv origin-key.pem /etc/ssl/cloudflare/origin-key.pem

chown root:root *.pem
chmod 600 *.pem

# Configure Nginx for Cloudflare
# /etc/nginx/sites-available/whmcs

server {
    listen 443 ssl http2;
    server_name whmcs.example.com;
    
    ssl_certificate /etc/ssl/cloudflare/origin.pem;
    ssl_certificate_key /etc/ssl/cloudflare/origin-key.pem;
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256;
    
    # Cloudflare IP ranges (optional verification)
    set $cloudflare false;
    if ($http CF-Connecting-IP) {
        set $cloudflare true;
    }
    
    root /var/www/whmcs;
    index index.php index.html;
    
    location / {
        try_files $uri $uri/ /index.php?$query_string;
    }
    
    # Static assets - long cache
    location ~* \.(css|js|jpg|jpeg|png|gif|ico|woff|woff2|ttf|svg)$ {
        expires 30d;
        add_header Cache-Control "public, immutable";
    }
    
    # PHP - no cache
    location ~ \.php$ {
        fastcgi_pass unix:/run/php/php-fpm-whmcs.sock;
        fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
        include fastcgi_params;
    }
}
```

### Step 3: Configure Cloudflare Page Rules

```
Page Rule 1: Cache Everything (for static content)
URL: example.com/whmcs/templates/*
TTL: 30 days
Edge Cache TTL: 30 days
Cache Level: Cache Everything

Page Rule 2: No Cache (for dynamic content)
URL: example.com/whmcs/clientarea.php
Cache Level: Bypass

Page Rule 3: Origin Cache Control
URL: example.com/whmcs/*
Edge Cache TTL: 7 days
```

### Step 4: AWS CloudFront Integration

```bash
#!/bin/bash
# /opt/scripts/setup_cloudfront.sh

# Install AWS CLI
apt-get install -y python3-pip
pip3 install awscli

# Configure AWS credentials
aws configure

# Create CloudFront distribution
DISTRIBUTION=$(aws cloudfront create-distribution \
    --origin-domain-name whmcs.example.com \
    --default-root-object index.php \
    --viewer-certificate acm-certificate-arn \
    --default-cache-behavior '{
        "TargetOriginId": "whmcs-origin",
        "ViewerProtocolPolicy": "redirect-to-https",
        "AllowedMethods": ["GET", "HEAD", "OPTIONS", "POST"],
        "CachedMethods": ["GET", "HEAD"],
        "ForwardedValues": {
            "QueryString": true,
            "Cookies": { "Forward": "whitelist", "WhitelistedNames": "WHMCS" }
        },
        "MinTTL": 0,
        "DefaultTTL": 86400,
        "MaxTTL": 604800
    }' \
    --price-class "PriceClass_All" \
    --query-output-format json)

echo "$DISTRIBUTION" | jq -r '.Distribution.Id'
echo "$DISTRIBUTION" | jq -r '.Distribution.DomainName'
```

### Step 5: WHMCS CDN Configuration Hook

```php
<?php
// /var/www/whmcs/includes/hooks/cdn_hook.php
// CDN optimization hook for WHMCS

use WHMCS\View\Markup\Smarty\MediaUri;

add_hook('ClientAreaHeadOutput', 1, function($vars) {
    $cdnBase = 'https://cdn.your-cdn.com/whmcs';
    
    return <<<HTML
<script>
    // Preload critical assets
    document.addEventListener('DOMContentLoaded', function() {
        // Preconnect to CDN
        var preconnect = document.createElement('link');
        preconnect.rel = 'preconnect';
        preconnect.href = 'https://cdn.your-cdn.com';
        preconnect.crossOrigin = 'anonymous';
        document.head.appendChild(preconnect);
    });
</script>
<style>
    /* Critical CSS inline */
</style>
HTML;
});

// Filter asset URLs for CDN
add_hook('ClientAreaPage', 1, function($vars) {
    $cdnBase = 'https://cdn.your-cdn.com/whmcs';
    
    // Add CDN hints to asset URLs
});
```

### Step 6: Image Optimization for CDN

```php
<?php
// /var/www/whmcs/includes/hooks/image_optimization_hook.php

add_hook('ClientAreaHeadOutput', 1, function($vars) {
    return <<<HTML
<script>
// Lazy load images with CDN optimization
document.addEventListener('DOMContentLoaded', function() {
    const cdnBase = 'https://cdn.your-cdn.com/whmcs';
    
    // Lazy load images
    const images = document.querySelectorAll('img[data-src]');
    const observer = new IntersectionObserver((entries) => {
        entries.forEach(entry => {
            if (entry.isIntersecting) {
                const img = entry.target;
                const src = img.getAttribute('data-src');
                
                // Add CDN transformation
                if (src.includes('/assets/img/')) {
                    img.src = cdnBase + src + '?w=400&q=75&f=auto';
                    img.srcset = 
                        cdnBase + src + '?w=400&q=75&f=auto 400w,' +
                        cdnBase + src + '?w=800&q=75&f=auto 800w,' +
                        cdnBase + src + '?w=1200&q=75&f=auto 1200w';
                }
                
                observer.unobserve(img);
            }
        });
    });
    
    images.forEach(img => observer.observe(img));
});
</script>
HTML;
});
```

### Step 7: Cache Management Script

```bash
#!/bin/bash
# /opt/scripts/cdn_cache_purge.sh

CDN_API_KEY="your-cloudflare-api-key"
CDN_ZONE_ID="your-zone-id"
CDN_BASE_URL="https://api.cloudflare.com/client/v4"

purge_all() {
    echo "Purging all CDN cache..."
    curl -X POST "$CDN_BASE_URL/zones/$CDN_ZONE_ID/purge_cache" \
        -H "Authorization: Bearer $CDN_API_KEY" \
        -H "Content-Type: application/json" \
        -d '{"purge_everything": true}'
}

purge_url() {
    local url=$1
    echo "Purging URL: $url"
    curl -X POST "$CDN_BASE_URL/zones/$CDN_ZONE_ID/purge_cache" \
        -H "Authorization: Bearer $CDN_API_KEY" \
        -H "Content-Type: application/json" \
        -d "{\"files\": [\"$url\"]}"
}

purge_prefix() {
    local prefix=$1
    echo "Purging prefix: $prefix"
    curl -X POST "$CDN_BASE_URL/zones/$CDN_ZONE_ID/purge_cache" \
        -H "Authorization: Bearer $CDN_API_KEY" \
        -H "Content-Type: application/json" \
        -d "{\"prefixes\": [\"$prefix\"]}"
}

# Usage examples
case "${1:-}" in
    all)
        purge_all
        ;;
    url)
        purge_url "$2"
        ;;
    prefix)
        purge_prefix "$2"
        ;;
    *)
        echo "Usage: $0 {all|url|prefix} [value]"
        ;;
esac
```

### Step 8: Performance Monitoring

```php
<?php
// /opt/scripts/cdn_performance.php
// Monitor CDN performance and cache hit rates

class CDNPerformanceMonitor {
    private $pdo;
    private $metricsTable = 'mod_cdn_metrics';
    
    public function __construct() {
        $this->pdo = new PDO('mysql:host=localhost', 'whmcs', 'password');
    }
    
    public function recordMetrics(): void {
        $timestamp = date('Y-m-d H:i:s');
        
        // Cloudflare Analytics API
        $analytics = $this->getCloudflareAnalytics();
        
        // Record to database
        $stmt = $this->pdo->prepare("
            INSERT INTO {$this->metricsTable} 
            (timestamp, requests, cached_requests, bandwidth_saved, avg_cache_ttl)
            VALUES (?, ?, ?, ?, ?)
        ");
        
        $stmt->execute([
            $timestamp,
            $analytics['requests'],
            $analytics['cached_requests'],
            $analytics['bandwidth_saved'],
            $analytics['avg_cache_ttl']
        ]);
        
        // Check for issues
        $this->alertLowCacheRate($analytics);
        $this->alertHighBandwidth($analytics);
    }
    
    private function getCloudflareAnalytics(): array {
        $zoneId = 'your-zone-id';
        $apiKey = 'your-api-key';
        
        $start = date('c', strtotime('-1 hour'));
        $end = date('c');
        
        $ch = curl_init("https://api.cloudflare.com/client/v4/zones/{$zoneId}/analytics/dashboard?since={$start}&until={$end}");
        curl_setopt_array($ch, [
            CURLOPT_RETURNTRANSFER => true,
            CURLOPT_HTTPHEADER => ["Authorization: Bearer {$apiKey}"]
        ]);
        
        $response = json_decode(curl_exec($ch), true);
        curl_close($ch);
        
        $totals = $response['result']['totals'];
        
        return [
            'requests' => $totals['requests'],
            'cached_requests' => $totals['cached_requests'],
            'bandwidth_saved' => $totals['bandwidth_saved_gb'] ?? 0,
            'avg_cache_ttl' => $totals['cached_bytes'] / max(1, $totals['cached_requests'])
        ];
    }
    
    private function alertLowCacheRate(array $analytics): void {
        $cacheRate = ($analytics['cached_requests'] / max(1, $analytics['requests'])) * 100;
        
        if ($cacheRate < 70) {
            logActivity("CDN Alert: Low cache rate ({$cacheRate}%)");
            // Send alert
        }
    }
    
    public function generateReport(): array {
        $stmt = $this->pdo->query("
            SELECT 
                DATE(timestamp) as date,
                SUM(requests) as total_requests,
                SUM(cached_requests) as cached,
                SUM(bandwidth_saved) as bandwidth_saved
            FROM {$this->metricsTable}
            WHERE timestamp > DATE_SUB(NOW(), INTERVAL 30 DAY)
            GROUP BY DATE(timestamp)
            ORDER BY date DESC
        ");
        
        return $stmt->fetchAll(PDO::FETCH_ASSOC);
    }
}
```

## CDN Configuration for Different Assets

| Asset Type | Cache Duration | Strategy |
|------------|---------------|----------|
| CSS/JS | 7-30 days | Versioned filenames |
| Images | 30-90 days | Optimize on upload |
| Documents | 30 days | Original filenames |
| Dynamic pages | 0 (bypass) | No cache |
| API responses | 0 (bypass) | No cache |

## Best Practices

1. **Cache Static, Bypass Dynamic**: Only cache assets that don't change per user
2. **Use Cache Tags**: Tag content by type for selective purging
3. **Optimize Images**: Resize before CDN, use WebP
4. **Version Assets**: Append version to CSS/JS for cache busting
5. **Monitor Hit Rate**: Track and optimize cache efficiency
6. **Purge Strategy**: Plan for WHMCS updates and content changes

## Common Pitfalls

- **Caching Dynamic Content**: User-specific data cached for all users
- **Stale Content**: Old versions served after updates
- **Cookie Issues**: Cookies forwarded causing cache misses
- **SSL Issues**: Mixed content warnings
- **Origin Shielding**: Double caching without proper setup

## Verification Checklist

- [ ] CDN configured for static assets
- [ ] Dynamic pages bypass cache
- [ ] SSL working correctly
- [ ] Images optimized and cached
- [ ] Cache purge working
- [ ] Performance improved (test with Lighthouse)
- [ ] Cache hit rate above 80%
- [ ] No mixed content warnings

## Related Documentation

- [WHMCS Performance Audit](whmcs-performance-audit.md)
- [WHMCS Load Balancing](whmcs-load-balancing.md)
- [Cloudflare Documentation](https://developers.cloudflare.com)