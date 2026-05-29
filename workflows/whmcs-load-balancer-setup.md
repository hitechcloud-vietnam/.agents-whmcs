# WHMCS Load Balancer Setup Workflow

## Description
Guide for configuring WHMCS behind a load balancer with proper session handling and health checks.

## Prerequisites
- Multiple WHMCS servers
- Load balancer (HAProxy, Nginx, or cloud LB)
- Shared storage or session management
- SSL certificate

## Architecture
```
                    [Load Balancer]
                    (443/80)
                         |
    +--------------------+--------------------+
    |                    |                    |
  [Node 1]            [Node 2]            [Node 3]
  /var/www/whmcs     /var/www/whmcs     /var/www/whmcs
      |                   |                   |
      +-------------------+-------------------+
                    [Shared Storage]
                    [Database Cluster]
```

## Steps

### Step 1: Prepare WHMCS Nodes
```bash
# Install WHMCS on all nodes (same installation)
for node in node1 node2 node3; do
    ssh root@$node "apt update && apt install -y nginx php-fpm php-mysql php-gd php-curl"
done

# Setup shared storage mount on all nodes
for node in node1 node2 node3; do
    ssh root@$node "mount -t nfs nfs_server:/shared /var/www/whmcs"
done
```

### Step 2: Configure Session Management
```php
// In configuration.php - Use database or Redis sessions
// Option 1: Database sessions
ini_set('session.save_handler', 'database');

// Option 2: Redis sessions (recommended)
$redis_host = 'redis.internal';
$redis_port = 6379;
$redis_password = 'secret';

ini_set('session.save_handler', 'redis');
ini_set('session.save_path', "tcp://$redis_host:$redis_port?auth=$redis_password");
```

### Step 3: Create Health Check Script
```php
<?php
// /var/www/whmcs/healthcheck.php
// Place this in WHMCS root directory

$checks = [];
$healthy = true;

// Check database
try {
    $pdo = new PDO(
        'mysql:host=' . DB_HOST . ';dbname=' . DB_NAME . ';charset=utf8mb4',
        DB_USERNAME,
        DB_PASSWORD,
        [PDO::ATTR_TIMEOUT => 5]
    );
    $pdo->query('SELECT 1');
    $checks['database'] = 'ok';
} catch (Exception $e) {
    $checks['database'] = 'error';
    $healthy = false;
}

// Check disk space
$free = disk_free_space('/var/www/whmcs');
if ($free < 1000000000) { // Less than 1GB
    $checks['disk'] = 'low';
    $healthy = false;
} else {
    $checks['disk'] = 'ok';
}

// Check PHP-FPM
if (function_exists('opcache_get_status')) {
    $checks['opcache'] = 'ok';
}

// Return response
http_response_code($healthy ? 200 : 503);
header('Content-Type: application/json');
echo json_encode([
    'status' => $healthy ? 'healthy' : 'unhealthy',
    'timestamp' => time(),
    'checks' => $checks
]);
exit;
```

### Step 4: Configure HAProxy
```bash
cat > /etc/haproxy/haproxy.cfg << 'EOF'
global
    log /dev/log local0
    log /dev/log local1 notice
    maxconn 4096
    user haproxy
    group haproxy
    daemon
    tune.ssl.default-dh-param 2048

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

# Frontend - HTTPS
frontend https_front
    bind *:443 ssl crt /etc/ssl/private/whmcs-combo.pem
    mode http
    
    # Add X-Forwarded headers
    http-request set-header X-Forwarded-Proto https
    http-request set-header X-Forwarded-For %[src]
    http-request set-header X-Real-IP %[src]
    http-request set-header Host %[hdr(host)]
    
    default_backend whmcs_backend

# Frontend - HTTP (redirect to HTTPS)
frontend http_front
    bind *:80
    mode http
    http-request redirect scheme https code 301 if !{ ssl_fc }

# Backend
backend whmcs_backend
    mode http
    balance roundrobin
    
    # Health check
    option httpchk
    http-check expect status 200
    http-check get /healthcheck.php
    
    # Cookie-based persistence (optional)
    cookie WHMCS_NODE insert indirect nocache
    
    # Backend servers
    server node1 10.0.0.11:443 check inter 5s fall 2 rise 2 ssl verify none cookie node1
    server node2 10.0.0.12:443 check inter 5s fall 2 rise 2 ssl verify none cookie node2
    server node3 10.0.0.13:443 check inter 5s fall 2 rise 2 ssl verify none cookie node3

# Stats page
listen stats
    bind *:8404
    stats enable
    stats uri /stats
    stats refresh 30s
    stats admin if LOCALHOST
EOF
```

### Step 5: Nginx as Load Balancer (Alternative)
```nginx
upstream whmcs_backend {
    least_conn;
    
    server 10.0.0.11:443 ssl;
    server 10.0.0.12:443 ssl;
    server 10.0.0.13:443 ssl;
    
    keepalive 32;
}

server {
    listen 443 ssl;
    server_name whmcs.example.com;
    
    ssl_certificate /etc/ssl/certs/whmcs.pem;
    ssl_certificate_key /etc/ssl/private/whmcs.key;
    
    location / {
        proxy_pass https://whmcs_backend;
        
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_set_header X-Forwarded-Host $host;
        
        proxy_http_version 1.1;
        proxy_set_header Connection "";
        
        # Timeouts
        proxy_connect_timeout 60s;
        proxy_send_timeout 300s;
        proxy_read_timeout 300s;
        
        # Buffers
        proxy_buffer_size 128k;
        proxy_buffers 4 256k;
        proxy_busy_buffers_size 256k;
    }
    
    location /healthcheck.php {
        proxy_pass https://whmcs_backend/healthcheck.php;
        proxy_connect_timeout 5s;
        proxy_next_upstream error timeout http_503;
    }
}
```

### Step 6: Whitelist Load Balancer IP
```sql
-- In WHMCS database
-- Add load balancer IP to trusted proxies

INSERT INTO tblconfiguration (setting, value) VALUES ('TrustedProxyIPs', '10.0.0.1,10.0.0.2,10.0.0.3');
```

### Step 7: Configure File Permissions
```bash
# On each node
chown -R www-data:www-data /var/www/whmcs
chmod -R 755 /var/www/whmcs
chmod 644 /var/www/whmcs/configuration.php
chmod 755 /var/www/whmcs/{templates_c,attachments,downloads,language}
find /var/www/whmcs/templates_c -type f -exec chmod 644 {} \;
```

### Step 8: Test Load Balancing
```bash
# Test health check endpoint
curl https://10.0.0.11/healthcheck.php
curl https://10.0.0.12/healthcheck.php
curl https://10.0.0.13/healthcheck.php

# Test through load balancer
curl -I https://whmcs.example.com/healthcheck.php

# Test session persistence
curl -c cookies.txt -b cookies.txt https://whmcs.example.com/
curl -b cookies.txt https://whmcs.example.com/clientarea.php
```

### Step 9: Monitor Load Balancer
```bash
# View HAProxy stats
# Access http://lb-server:8404/stats

# Monitor backend status
echo "show stat" | socat stdio /var/run/haproxy/admin.sock

# View current connections
echo "show connections" | socat stdio /var/run/haproxy/admin.sock
```

## Troubleshooting
- Verify all nodes have identical WHMCS configuration
- Check session storage is shared across nodes
- Ensure health check script returns 200 for healthy nodes
- Check firewall allows load balancer to reach backend nodes
- Verify SSL certificates are valid on all nodes

## Tags
- load-balancer
- haproxy
- nginx
- clustering
- scalability