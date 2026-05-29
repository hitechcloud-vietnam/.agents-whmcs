# WHMCS CDN Setup Workflow

## Description
Configure Content Delivery Network (CDN) for WHMCS to improve performance and reduce server load.

## Prerequisites
- CDN provider account (CloudFlare, StackPath, KeyCDN, etc.)
- SSL certificate
- DNS access
- SSH access (for configuration)

## Benefits of CDN
- Faster content delivery globally
- Reduced server bandwidth
- DDoS protection
- Static asset caching
- Reduced TTFB (Time To First Byte)

## Steps

### Step 1: Choose CDN Provider

| Provider | Features | Pricing |
|----------|----------|---------|
| CloudFlare | Free tier, DDoS protection, CDN | Free - $200/mo |
| StackPath | CDN, WAF, DDoS protection | $20/mo+ |
| KeyCDN | Pay-as-you-go, EU data centers | $4/100GB |
| AWS CloudFront | Deep AWS integration | $0.0085/GB+ |
| BunnyCDN | Simple pricing, fast global network | $1/100GB+ |

### Step 2: Configure CloudFlare (Recommended)
```bash
# Step 1: Add domain to CloudFlare
# Visit: https://dash.cloudflare.com/sign-up
# Add your domain and follow setup wizard

# Step 2: Update nameservers (CloudFlare will provide them)
# Update at your domain registrar

# Step 3: Wait for DNS propagation (usually 24-48 hours)
# Verify in CloudFlare dashboard

# Step 4: Configure DNS records
# Set A record for WHMCS with cloud (proxied) status
# Ensure SSL/TLS mode is "Full" or "Full (strict)"
```

### Step 3: Configure CloudFlare SSL
```bash
# In CloudFlare Dashboard:
# SSL/TLS > Overview
# Set to "Full (strict)"

# For origin certificates:
# SSL/TLS > Origin Server
# Create Origin Certificate
# Download and install on your server
```

### Step 4: Configure Page Rules
```bash
# In CloudFlare Dashboard > Rules > Page Rules

# Rule 1: Always Use HTTPS
# URL pattern: *whmcs.example.com/*
# Settings:
# - Always Use HTTPS: ON

# Rule 2: Cache static assets
# URL pattern: *whmcs.example.com/assets/*
# Settings:
# - Cache Level: Cache Everything
# - Edge Cache TTL: 1 week
# - Browser Cache TTL: 1 day

# Rule 3: Bypass cache for admin
# URL pattern: *whmcs.example.com/admin/*
# Settings:
# - Cache Level: Bypass
```

### Step 5: Configure Browser Cache Headers
```bash
# In CloudFlare Dashboard > Rules > Speed

# Add Cache Rules for static content
# Caching > Configuration > Browser Cache TTL
# Set to "Respect Existing Headers" or customize

# Or use Page Rules for specific paths
```

### Step 6: Configure Nginx for CDN
```bash
# Modify Nginx configuration
cat > /etc/nginx/sites-available/whmcs-cdn << 'EOF'
server {
    listen 443 ssl http2;
    server_name whmcs.example.com;
    root /var/www/whmcs;
    index index.php;

    # CloudFlare real IP configuration
    set_real_ip_from 103.21.244.0/22;
    set_real_ip_from 103.22.200.0/22;
    set_real_ip_from 103.31.4.0/22;
    set_real_ip_from 104.16.0.0/13;
    set_real_ip_from 104.24.0.0/14;
    set_real_ip_from 172.64.0.0/13;
    set_real_ip_from 192.168.127.0/21;
    set_real_ip_from 198.41.128.0/17;
    real_ip_header CF-Connecting-IP;

    # Static asset caching
    location ~* \.(jpg|jpeg|png|gif|ico|css|js|svg|woff|woff2|ttf|eot)$ {
        expires 7d;
        add_header Cache-Control "public, immutable";
        access_log off;
    }

    # HTML no cache
    location ~* \.html$ {
        expires -1;
        add_header Cache-Control "no-store, no-cache, must-revalidate";
    }

    # PHP files - no CDN
    location ~ \.php$ {
        # ... PHP configuration
    }
}
EOF

nginx -t && systemctl reload nginx
```

### Step 7: Configure Cache-Control Headers
```php
// Add to configuration.php or hooks file

// Hook to set cache headers for static assets
add_hook('ClientAreaPage', 1, function($vars) {
    // This runs on every page load
});

// Better: Set headers in Nginx/Apache instead
```

### Step 8: Configure CloudFlare Cache Rules
```bash
# In CloudFlare Dashboard > Caching > Cache Rules

# Rule 1: Edge TTL
# Pattern: *.whmcs.example.com/*
# TTL: 1 hour

# Rule 2: Query String Sort
# Sort query strings for better cache hit rate
# Enable "Origin Cache Control"
```

### Step 9: Setup Asset Minification
```bash
# CloudFlare Dashboard > Speed > Optimization

# Auto Minify
# Enable: HTML, CSS, JavaScript

# Brotli compression - already enabled on Pro+ plans
```

### Step 10: Configure CDN for Downloads
```bash
# If WHMCS serves large download files

# Option 1: Point downloads to CDN
# Configuration > System Settings > General
# Downloads Directory URL: https://cdn.whmcs.example.com/downloads/

# Option 2: Redirect downloads to CDN
// In configuration.php
$whmcs_config['downloads_url'] = 'https://cdn.whmcs.example.com/downloads';
```

### Step 11: Test CDN Configuration
```bash
# Test from multiple locations
curl -I https://whmcs.example.com/assets/img/logo.png

# Check for CDN headers
# Expected headers:
# cf-ray: [ray_id]
# CF-Cache-Status: HIT or MISS
# server: cloudflare

# Cache hit test
curl -I https://whmcs.example.com/assets/img/logo.png
# First request: CF-Cache-Status: MISS
# Second request: CF-Cache-Status: HIT

# Check with webpagetest.org
# https://www.webpagetest.org
```

### Step 12: CDN Monitoring
```bash
# CloudFlare Analytics
# View in Dashboard > Analytics

# For custom monitoring script
cat > /usr/local/bin/cdn-stats.sh << 'EOF'
#!/bin/bash

# CloudFlare API
API_KEY="your_cloudflare_api_key"
EMAIL="admin@example.com"
ZONE_ID="your_zone_id"

# Get cache statistics
curl -s -X GET "https://api.cloudflare.com/client/v4/zones/$ZONE_ID/stats?period=1440" \
    -H "X-Auth-Email: $EMAIL" \
    -H "X-Auth-Key: $API_KEY" \
    -H "Content-Type: application/json" | jq '.result'

# Get top traffic sources
curl -s -X GET "https://api.cloudflare.com/client/v4/zones/$ZONE_ID/top-traffic" \
    -H "X-Auth-Email: $EMAIL" \
    -H "X-Auth-Key: $API_KEY"
EOF
```

## Alternative CDN: BunnyCDN
```bash
# BunnyCDN Setup

# Step 1: Create Pull Zone
# https://bunny.net > CDN > New Pull Zone
# Origin: whmcs.example.com

# Step 2: Configure DNS
# Create CNAME record:
# cdn.whmcs.example.com -> cdn.bunny.net

# Step 3: Update WHMCS asset URLs
// In configuration.php
$asset_url = 'https://cdn.whmcs.example.com';
```

## CDN for Specific Assets

### Fonts
```bash
# Self-host fonts or use Google Fonts with CDN
# Avoid loading fonts from external CDNs for privacy

# Better: Self-host and serve via your CDN
```

### JavaScript Libraries
```bash
# Option 1: Use jsDelivr (public CDN)
# Option 2: Self-host critical JS
# Option 3: Load from your own CDN
```

## Performance Optimization
```bash
# Image optimization
# Enable Polish in CloudFlare
# Dashboard > Speed > Polish > Polish Images: Basic

# For Next-Gen formats:
# Enable Mirage, Polish with WebP
```

## Troubleshooting CDN Issues

### CSS/JS Not Loading
```bash
# Check for cache issues
# Purged CloudFlare cache?
curl -X POST "https://api.cloudflare.com/client/v4/zones/$ZONE_ID/purge_cache" \
    -H "X-Auth-Email: $EMAIL" \
    -H "X-Auth-Key: $API_KEY" \
    -H "Content-Type: application/json" \
    --data '{"purge_everything":true}'

# Check for mixed content
# View browser console for errors
```

### SSL Errors
```bash
# CloudFlare SSL modes:
# Off: No SSL (not recommended)
# Flexible: Client to CloudFlare HTTP (not recommended)
# Full: CloudFlare to origin HTTPS (cert required)
# Full (strict): Origin must have valid SSL

# Ensure origin has valid SSL certificate
```

## Performance Benchmarks
| Metric | Without CDN | With CDN |
|--------|-------------|----------|
| TTFB | 200-500ms | 20-50ms |
| Global Load | 3-5s | 1-2s |
| Bandwidth | High | Reduced 70% |
| Origin Requests | All | Static cached |

## Tags
- cdn
- performance
- cloudflare
- caching
- optimization