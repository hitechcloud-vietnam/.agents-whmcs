# WHMCS Reverse Proxy Setup Workflow

## Description
Configure reverse proxy in front of WHMCS for load balancing, SSL termination, or security.

## Prerequisites
- Reverse proxy server (Nginx/HAProxy/Varnish)
- Backend WHMCS server
- SSL certificates

## Use Cases
- SSL termination
- Load balancing multiple WHMCS instances
- Caching static content
- DDoS protection
- Geographic routing

## Steps

### Step 1: Configure Nginx as Reverse Proxy
```bash
# Install Nginx
apt install -y nginx

# Create reverse proxy configuration
cat > /etc/nginx/sites-available/whmcs-proxy << 'EOF'
# Upstream backend servers
upstream whmcs_backend {
    server 10.0.0.11:443;
    server 10.0.0.12:443 backup;
    keepalive 32;
}

# HTTP to HTTPS redirect
server {
    listen 80;
    server_name whmcs.example.com;
    return 301 https://$server_name$request_uri;
}

# HTTPS Reverse Proxy
server {
    listen 443 ssl http2;
    server_name whmcs.example.com;

    # SSL termination
    ssl_certificate /etc/ssl/certs/whmcs.pem;
    ssl_certificate_key /etc/ssl/private/whmcs.key;
    
    # SSL settings
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256;
    ssl_prefer_server_ciphers off;
    ssl_session_cache shared:SSL:50m;
    ssl_session_tickets off;

    # Headers
    add_header X-Real-IP $remote_addr;
    add_header X-Forwarded-For $proxy_add_x_forwarded_for;
    add_header X-Forwarded-Proto $scheme;
    add_header X-Forwarded-Host $host;
    add_header Host $host;

    # Proxy settings
    proxy_pass https://whmcs_backend;
    proxy_http_version 1.1;
    
    # Connection settings
    proxy_set_header Connection "";
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto $scheme;
    
    # Timeouts
    proxy_connect_timeout 60s;
    proxy_send_timeout 300s;
    proxy_read_timeout 300s;
    
    # Buffers
    proxy_buffer_size 128k;
    proxy_buffers 4 256k;
    proxy_busy_buffers_size 256k;
    
    # SSL to backend
    proxy_ssl_verify off;
    proxy_ssl_server_name on;

    # Caching (optional)
    # proxy_cache_valid 200 60m;
    # proxy_cache_bypass $cookie_nocache;
}

# Stats page (optional)
server {
    listen 8888;
    server_name localhost;
    location / {
        stub_status on;
        access_log off;
    }
}
EOF

# Enable and test
ln -s /etc/nginx/sites-available/whmcs-proxy /etc/nginx/sites-enabled/
nginx -t
systemctl reload nginx
```

### Step 2: Configure HAProxy as Reverse Proxy
```bash
# Install HAProxy
apt install -y haproxy

# Configure HAProxy
cat > /etc/haproxy/haproxy.cfg << 'EOF'
global
    log /dev/log local0
    log /dev/log local1 notice
    maxconn 4096
    user haproxy
    group haproxy
    daemon
    stats socket /run/haproxy/admin.sock mode 660 level admin

defaults
    log     global
    mode    http
    option  httplog
    option  dontlognull
    option  http-server-close
    option  forwardfor except 127.0.0.0/8
    option  redispatch
    retries 3
    timeout connect 5000ms
    timeout client  50000ms
    timeout server  50000ms

# Frontend
frontend https_front
    bind *:443 ssl crt /etc/ssl/private/whmcs.pem
    mode http
    
    # Add headers
    http-request add-header X-Forwarded-Proto https
    http-request add-header X-Real-IP %[src]
    
    # Default backend
    default_backend whmcs_servers

# Backend
backend whmcs_servers
    mode http
    balance roundrobin
    
    # Health check
    option httpchk GET /healthcheck.php
    http-check expect status 200
    
    # Backend servers
    server whmcs1 10.0.0.11:443 ssl verify none check inter 5s fall 2 rise 2
    server whmcs2 10.0.0.12:443 ssl verify none check inter 5s fall 2 rise 2 backup

# Stats
listen stats
    bind *:8404
    stats enable
    stats uri /stats
    stats refresh 30s
EOF

systemctl restart haproxy
```

### Step 3: Configure Backend WHMCS
```bash
# On WHMCS server, trust proxy headers
# Add to Nginx config on backend

cat > /etc/nginx/sites-available/whmcs-backend << 'EOF'
server {
    listen 443 ssl http2;
    server_name whmcs.example.com;
    root /var/www/whmcs;
    index index.php;

    ssl_certificate /etc/ssl/certs/whmcs.pem;
    ssl_certificate_key /etc/ssl/private/whmcs.key;

    # Trust proxy headers
    set_real_ip_from 10.0.0.1/32;  # Proxy IP
    real_ip_header X-Forwarded-For;
    real_ip_recursive on;

    # PHP configuration
    location ~ \.php$ {
        include snippets/fastcgi-php.conf;
        fastcgi_pass unix:/var/run/php/php-fpm.sock;
    }
}
EOF
```

### Step 4: Configure WHMCS for Proxy
```php
// In configuration.php

// Whitelist proxy server IP
$whmcs_config = [
    'TrustedProxyIPs' => ['10.0.0.1'],
];

// Or add multiple proxies
// $whmcs_config = [
//     'TrustedProxyIPs' => ['10.0.0.1', '10.0.0.2', '10.0.0.3'],
// ];
```

### Step 5: Handle WebSocket (if needed)
```bash
# Add WebSocket support for real-time features
# In Nginx proxy config:

location /ws {
    proxy_pass https://whmcs_backend;
    proxy_http_version 1.1;
    proxy_set_header Upgrade $http_upgrade;
    proxy_set_header Connection "upgrade";
    proxy_set_header Host $host;
    proxy_read_timeout 86400;
}
```

### Step 6: Configure Caching
```bash
# Varnish cache (advanced)
// Create /etc/varnish/default.vcl

vcl 4.0;

backend default {
    .host = "10.0.0.11";
    .port = "443";
    .ssl = true;
    .probe = {
        .url = "/healthcheck.php";
        .timeout = 5s;
        .interval = 10s;
        .window = 5;
        .threshold = 3;
    }
}

sub vcl_recv {
    # Don't cache admin pages
    if (req.url ~ "^/admin/") {
        return (pass);
    }
    
    # Don't cache logged in users
    if (req.http.Cookie ~ "whmcs_auth_") {
        return (pass);
    }
    
    # Cache static assets
    if (req.url ~ "\.(jpg|jpeg|png|gif|ico|css|js|svg|woff|woff2|ttf)$") {
        unset req.http.Cookie;
        return (hash);
    }
    
    return (pass);
}

sub vcl_backend_response {
    # Cache static assets
    if (bereq.url ~ "\.(jpg|jpeg|png|gif|ico|css|js|svg|woff|woff2|ttf)$") {
        set beresp.ttl = 7d;
        unset beresp.http.Set-Cookie;
    }
}
```

### Step 7: Test Reverse Proxy
```bash
# Test connectivity through proxy
curl -I https://whmcs.example.com

# Check proxy headers
curl -I https://whmcs.example.com | grep -E "X-Forwarded|X-Real-IP"

# Expected output:
# X-Forwarded-For: client_ip
# X-Real-IP: client_ip
# X-Forwarded-Proto: https

# Test from backend perspective
# On backend, check if real IP is received
tail -f /var/log/nginx/access.log
# Should show proxy IP, not client IP (unless configured correctly)

# Load test
apt install -y apache2-utils
ab -n 1000 -c 10 https://whmcs.example.com/
```

### Step 8: Monitoring Proxy
```bash
# HAProxy stats
# Access: http://your-proxy:8404/stats

# Check backend health
curl -I https://whmcs.example.com/healthcheck.php

# Monitor logs
tail -f /var/log/haproxy/haproxy.log

# Create monitoring script
cat > /usr/local/bin/proxy-monitor.sh << 'EOF'
#!/bin/bash

BACKEND1="10.0.0.11"
BACKEND2="10.0.0.12"
ALERT_EMAIL="admin@example.com"

# Check backend health
for backend in $BACKEND1 $BACKEND2; do
    if ! curl -sf https://$backend/healthcheck.php > /dev/null; then
        echo "Backend $backend is DOWN" | \
            mail -s "Proxy Alert: Backend Down" $ALERT_EMAIL
    fi
done

# Check response time
TIME=$(curl -o /dev/null -s -w '%{time_total}' https://whmcs.example.com)
if (( $(echo "$TIME > 2.0" | bc -l) )); then
    echo "Response time slow: $TIME seconds" | \
        mail -s "Proxy Alert: Slow Response" $ALERT_EMAIL
fi
EOF

chmod +x /usr/local/bin/proxy-monitor.sh
echo "*/5 * * * * /usr/local/bin/proxy-monitor.sh" >> /etc/crontab
```

## Troubleshooting
```bash
# Check proxy logs
tail -f /var/log/nginx/error.log

# Check backend logs
ssh backend_server "tail -f /var/log/nginx/access.log"

# Common issues:
# - SSL certificate mismatch
# - Backend not accepting proxy headers
# - Firewall blocking proxy connections
```

## Security Considerations
- Restrict backend server access to proxy only
- Use internal network for proxy-to-backend
- Enable SSL on both proxy and backend
- Configure rate limiting
- Set up intrusion detection

## Tags
- reverse-proxy
- load-balancer
- nginx
- haproxy
- infrastructure