# WHMCS HTTP/2 Setup Workflow

## Description
Enable HTTP/2 protocol for WHMCS to improve page load performance and user experience.

## Prerequisites
- SSL certificate (HTTP/2 requires TLS)
- Web server with HTTP/2 support (Nginx 1.9.5+, Apache 2.4.17+)
- OpenSSL 1.0.2+ (for ALPN support)

## Benefits of HTTP/2
- Multiplexing (multiple requests over single connection)
- Header compression
- Server push
- Parallel asset loading
- Reduced latency

## Steps

### Step 1: Check Current HTTP Version
```bash
# Check what HTTP version is being used
curl -I https://whmcs.example.com 2>/dev/null | grep -i "http/"

# Or use browser dev tools
# Network tab > Protocol column

# Test with h2c (HTTP/2 cleartext)
curl -I --http2 https://whmcs.example.com
```

### Step 2: Verify HTTP/2 Support

**Nginx:**
```bash
# Check Nginx version
nginx -v
# Should be 1.9.5 or higher

# Check module
nginx -V 2>&1 | grep -o http_v2_module
# Should output: http_v2_module
```

**Apache:**
```bash
# Check Apache version
apache2 -v
# Should be 2.4.17 or higher

# Check module
apache2ctl -M | grep http2
```

### Step 3: Enable HTTP/2 in Nginx
```bash
# Edit Nginx site configuration
cat > /etc/nginx/sites-available/whmcs << 'EOF'
server {
    listen 443 ssl http2;
    server_name whmcs.example.com;
    root /var/www/whmcs;
    index index.php;

    # SSL Configuration
    ssl_certificate /etc/ssl/certs/whmcs-combined.crt;
    ssl_certificate_key /etc/ssl/private/whmcs.key;
    
    # Modern SSL settings
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256;
    ssl_prefer_server_ciphers off;
    
    # OCSP Stapling
    ssl_stapling on;
    ssl_stapling_verify on;
    resolver 8.8.8.8 8.8.4.4 valid=300s;
    resolver_timeout 5s;

    # HTTP/2 Server Push (optional)
    http2_push_preload on;

    # PHP Configuration
    location ~ \.php$ {
        include snippets/fastcgi-php.conf;
        fastcgi_pass unix:/var/run/php/php-fpm.sock;
    }

    # Static assets with caching
    location ~* \.(jpg|jpeg|png|gif|ico|css|js|svg|woff|woff2|ttf|eot)$ {
        expires 7d;
        add_header Cache-Control "public, immutable";
        access_log off;
    }
}
EOF

# Test and reload
nginx -t
systemctl reload nginx
```

### Step 4: Enable HTTP/2 in Apache
```bash
# Enable HTTP/2 module
a2enmod http2

# Configure in site
cat > /etc/apache2/sites-available/whmcs.conf << 'EOF'
<VirtualHost *:443>
    ServerName whmcs.example.com
    DocumentRoot /var/www/whmcs
    
    Protocols h2 h2c http/1.1
    
    SSLEngine on
    SSLCertificateFile /etc/ssl/certs/whmcs-combined.crt
    SSLCertificateKeyFile /etc/ssl/private/whmcs.key
    
    <Directory /var/www/whmcs>
        AllowOverride All
        Require all granted
    </Directory>
    
    # Enable HTTP/2 settings
    H2EarlyHints on
    H2PushResource /assets/css/main.css
    H2PushResource /assets/js/main.js
</VirtualHost>
EOF

# Test and reload
apachectl configtest
systemctl reload apache2
```

### Step 5: Configure HTTP/2 Server Push
```php
// Add server push headers in PHP or Nginx config

// Nginx - Manual push
location ~* \.php$ {
    http2_push /assets/css/main.css;
    http2_push /assets/js/main.js;
    http2_push_preload on;
}

// PHP - Programmatic push
header("Link: </assets/css/main.css>; rel=preload; as=style", false);
header("Link: </assets/js/main.js>; rel=preload; as=script", false);
```

### Step 6: Verify HTTP/2 is Active
```bash
# Check with curl
curl -I -v --http2 https://whmcs.example.com 2>&1 | grep -E "HTTP/|h2"

# Expected output should show h2 or HTTP/2

# Online testing
# https://tools.keycdn.com/http2-test
# https://http2.pro/

# Browser testing
# Chrome DevTools > Network > Protocol column
# Should show "h2" for HTTP/2 connections
```

### Step 7: Optimize for HTTP/2

**Enable Connection Keep-Alive:**
```bash
# In Nginx
keepalive_timeout 65;
keepalive_requests 100;

# In Apache
KeepAlive On
MaxKeepAliveRequests 100
KeepAliveTimeout 5
```

**Optimize SSL Session Cache:**
```bash
# In Nginx
ssl_session_cache shared:SSL:50m;
ssl_session_timeout 1d;
ssl_session_tickets off;
```

**Resource Hints:**
```html
<!-- Add to template header -->
<link rel="preconnect" href="https://whmcs.example.com">
<link rel="dns-prefetch" href="https://whmcs.example.com">

<!-- Preload critical resources -->
<link rel="preload" href="/assets/css/main.css" as="style">
<link rel="preload" href="/assets/js/main.js" as="script">
```

### Step 8: Configure Brotli Compression
```bash
# Install Nginx with brotli module
apt install -y nginx-module-brotli

# Configure brotli
cat >> /etc/nginx/nginx.conf << 'EOF'
brotli on;
brotli_comp_level 6;
brotli_types text/plain text/css application/json application/javascript text/xml application/xml application/xml+rss text/javascript;
EOF

# Add to site config
cat >> /etc/nginx/sites-available/whmcs << 'EOF'
    # Brotli compression
    brotli on;
    brotli_types text/html text/plain text/css application/json application/javascript text/xml application/xml application/xml+rss text/javascript;
EOF

systemctl reload nginx
```

### Step 9: Monitor HTTP/2 Performance
```bash
# Use webpagetest.org
# Test from: https://www.webpagetest.org
# Check "Enable HTTP/2 by default" in Advanced Settings

# Check with nghttp
apt install -y nghttp2
nghttp -ns https://whmcs.example.com | head -20

# View HTTP/2 frames
nghttp -ya https://whmcs.example.com
```

### Step 10: HTTP/3 (QUIC) - Experimental
```bash
# HTTP/3 is the next generation
# Requires Nginx with quic support or CloudFlare

# For CloudFlare users, HTTP/3 is automatic
# Check if enabled: CloudFlare Dashboard > Network > HTTP/3 (QUIC)

# For Nginx with QUIC (experimental)
# Use Cloudflare's patch or nginx-quic package
```

## Troubleshooting

### HTTP/2 Not Working
```bash
# Check SSL certificate validity
openssl s_client -connect whmcs.example.com:443 -servername whmcs.example.com

# Check OpenSSL version
openssl version
# Must be 1.0.2+ for ALPN

# Check for conflicting protocols
# Some proxies don't support HTTP/2 on older TLS

# Clear browser cache
# Chrome: Shift+Ctrl+R (hard reload)
```

### Performance Issues
```bash
# HTTP/2 multiplexing can cause head-of-line blocking
# Consider TLS False Start

# Monitor with:
# Chrome DevTools > Performance tab
# Check "Network" filter for stalled requests
```

## Performance Comparison
| Metric | HTTP/1.1 | HTTP/2 |
|--------|----------|--------|
| Initial Page Load | 2.5s | 1.2s |
| Concurrent Connections | 6 max | 1 (multiplexed) |
| Header Size | Full | Compressed (HPACK) |
| Server Resources | Higher | Lower |

## Tags
- http2
- http3
- quic
- performance
- optimization
- protocol