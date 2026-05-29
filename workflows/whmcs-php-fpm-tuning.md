# WHMCS PHP-FPM Tuning Workflow

## Description
Optimize PHP-FPM configuration for better WHMCS performance and resource utilization.

## Prerequisites
- PHP-FPM installed
- SSH access
- Understanding of your server resources

## Performance Metrics
- Requests per second
- Response time
- Memory usage
- CPU utilization
- Connection handling

## Steps

### Step 1: Check Current Configuration
```bash
# Check PHP-FPM version
php-fpm8.2 -v

# Check current configuration
php-fpm8.2 -tt

# View pool configuration
cat /etc/php/8.2/fpm/pool.d/www.conf

# Check current status
systemctl status php8.2-fpm
pm status php8.2-fpm
```

### Step 2: Configure PHP-FPM Pool
```bash
# Create optimized pool configuration
cat > /etc/php/8.2/fpm/pool.d/whmcs.conf << 'EOF'
[whmcs]
user = www-data
group = www-data

listen = /var/run/php/whmcs.sock
listen.owner = www-data
listen.group = www-data
listen.mode = 0660

# Process manager: dynamic, static, ondemand
pm = dynamic

# For low traffic: ondemand
# For high traffic: dynamic or static
# pm = dynamic

# Maximum children
pm.max_children = 50

# Number of children at startup
pm.start_servers = 10

# Minimum spare servers
pm.min_spare_servers = 5

# Maximum spare servers
pm.max_spare_servers = 20

# Number of requests before recycling
pm.max_requests = 500

# Request termination timeout
request_terminate_timeout = 60s

# FastCGI finish request
request_terminate_timeout = 60

# Slow log for debugging
slowlog = /var/log/php8.2-fpm-slow.log
request_slowlog_timeout = 10s

# PHP settings per pool
php_admin_value[error_log] = /var/log/php8.2-fpm-whmcs-error.log
php_admin_flag[log_errors] = on
php_admin_value[memory_limit] = 256M
php_admin_value[max_execution_time] = 60
php_admin_value[post_max_size] = 64M
php_admin_value[upload_max_filesize] = 64M
php_admin_value[max_input_time] = 60

# Security
security.limit_extensions = .php
EOF

# Disable default pool
mv /etc/php/8.2/fpm/pool.d/www.conf /etc/php/8.2/fpm/pool.d/www.conf.disabled
```

### Step 3: Update PHP Settings
```bash
# Edit PHP configuration
cat > /etc/php/8.2/fpm/php.ini << 'EOF'
; Basic settings
memory_limit = 256M
max_execution_time = 60
max_input_time = 60
post_max_size = 64M
upload_max_filesize = 64M

; Error handling
error_reporting = E_ALL & ~E_DEPRECATED & ~E_STRICT
display_errors = Off
display_startup_errors = Off
log_errors = On
error_log = /var/log/php_errors.log

; OpCache
opcache.enable = 1
opcache.memory_consumption = 256
opcache.interned_strings_buffer = 16
opcache.max_accelerated_files = 10000
opcache.revalidate_freq = 0
opcache.validate_timestamps = 0
opcache.save_comments = 1
opcache.fast_shutdown = 1

; Session
session.save_handler = redis
session.save_path = "tcp://127.0.0.1:6379?auth=your_redis_password"

; Performance
realpath_cache_size = 4096K
realpath_cache_ttl = 600

; Security
expose_php = Off
allow_url_fopen = On
allow_url_include = Off
EOF

# Restart PHP-FPM
systemctl restart php8.2-fpm
```

### Step 4: Configure Nginx for PHP-FPM
```bash
cat > /etc/nginx/sites-available/whmcs << 'EOF'
server {
    listen 443 ssl http2;
    server_name whmcs.example.com;
    root /var/www/whmcs;
    index index.php;

    ssl_certificate /etc/ssl/certs/whmcs-combined.crt;
    ssl_certificate_key /etc/ssl/private/whmcs.key;

    # Logging
    access_log /var/log/nginx/whmcs_access.log;
    error_log /var/log/nginx/whmcs_error.log;

    location ~ \.php$ {
        include snippets/fastcgi-php.conf;
        
        # Use WHMCS pool
        fastcgi_pass unix:/var/run/php/whmcs.sock;
        
        # Timeouts
        fastcgi_connect_timeout 60s;
        fastcgi_send_timeout 300s;
        fastcgi_read_timeout 300s;
        
        # Buffers
        fastcgi_buffer_size 128k;
        fastcgi_buffers 4 256k;
        fastcgi_busy_buffers_size 256k;
        fastcgi_temp_file_write_size 256k;
        
        # Pass headers
        fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
        include fastcgi_params;
    }

    # Static files
    location ~* \.(jpg|jpeg|png|gif|ico|css|js|svg|woff|woff2)$ {
        expires 7d;
        add_header Cache-Control "public, immutable";
        access_log off;
    }
}
EOF

nginx -t && systemctl reload nginx
```

### Step 5: Process Manager Tuning

**Dynamic PM (Recommended for variable traffic):**
```bash
# Calculate based on server resources
# Total PHP processes = (Total RAM - OS RAM - MySQL RAM) / PHP memory limit

# Example: 8GB server
# OS: 2GB
# MySQL: 2GB
# Available for PHP: 4GB
# PHP memory_limit: 256MB
# Max children: 16

pm.max_children = 16
pm.start_servers = 4
pm.min_spare_servers = 2
pm.max_spare_servers = 8
pm.max_requests = 500
```

**Static PM (For consistent high traffic):**
```bash
# Fixed number of processes
pm = static
pm.max_children = 25
pm.max_requests = 1000
```

**OnDemand PM (For low traffic, saves resources):**
```bash
pm = ondemand
pm.max_children = 10
pm.process_idle_timeout = 10s
pm.max_requests = 500
```

### Step 6: Enable Status Page
```bash
# Enable PHP-FPM status page
cat >> /etc/php/8.2/fpm/pool.d/whmcs.conf << 'EOF'

# Status page
pm.status_path = /php-fpm-status
EOF

# Configure Nginx for status page
cat >> /etc/nginx/sites-available/whmcs << 'EOF'

location ~ ^/(php-fpm-status|ping)$ {
    fastcgi_pass unix:/var/run/php/whmcs.sock;
    include fastcgi_params;
    fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
    
    # Allow only from localhost
    allow 127.0.0.1;
    allow 10.0.0.0/8;
    deny all;
}
EOF

systemctl reload php8.2-fpm nginx

# Test status page
curl http://127.0.0.1/php-fpm-status
curl http://127.0.0.1/ping
```

### Step 7: Monitor and Optimize
```bash
# Create monitoring script
cat > /usr/local/bin/php-fpm-monitor.sh << 'EOF'
#!/bin/bash

STATUS=$(curl -s http://127.0.0.1/php-fpm-status?json)
ALERT_EMAIL="admin@example.com"

# Extract metrics
echo "$STATUS" | jq -r '.pool'
echo "$STATUS" | jq -r '.active processes'
echo "$STATUS" | jq -r '.max children reached'

# Alert if max children reached
MAX_REACHED=$(echo "$STATUS" | jq -r '.max children reached')
if [ "$MAX_REACHED" -gt 10 ]; then
    echo "Max children reached $MAX_REACHED times" | \
        mail -s "PHP-FPM Alert" $ALERT_EMAIL
fi

# Check memory usage
ps aux | grep php-fpm | grep -v grep | awk '{sum+=$6} END {print sum/1024 " MB"}'
EOF

chmod +x /usr/local/bin/php-fpm-monitor.sh
echo "*/5 * * * * /usr/local/bin/php-fpm-monitor.sh" >> /etc/crontab
```

### Step 8: Performance Testing
```bash
# Install Apache Bench
apt install -y apache2-utils

# Test current performance
ab -n 1000 -c 10 https://whmcs.example.com/index.php

# Monitor during test
watch -n 1 "curl -s http://127.0.0.1/php-fpm-status"
```

## Recommended Settings by Traffic

### Low Traffic (< 1000 req/day)
```bash
pm = ondemand
pm.max_children = 5
pm.process_idle_timeout = 10s
memory_limit = 128M
```

### Medium Traffic (< 10000 req/day)
```bash
pm = dynamic
pm.max_children = 20
pm.start_servers = 3
pm.min_spare_servers = 2
pm.max_spare_servers = 10
memory_limit = 256M
```

### High Traffic (> 10000 req/day)
```bash
pm = dynamic
pm.max_children = 50
pm.start_servers = 10
pm.min_spare_servers = 5
pm.max_spare_servers = 20
pm.max_requests = 500
memory_limit = 512M
```

## Troubleshooting
```bash
# Check error logs
tail -f /var/log/php8.2-fpm-whmcs-error.log
tail -f /var/log/nginx/whmcs_error.log

# Check slow log
tail -f /var/log/php8.2-fpm-slow.log

# Restart on issues
systemctl restart php8.2-fpm
```

## Tags
- php-fpm
- performance
- tuning
- optimization
- server