# WHMCS Scaling Guide Workflow

## Purpose

Implement horizontal and vertical scaling strategies for WHMCS to handle growing workloads, improve performance, and ensure high availability. This workflow covers scaling approaches, implementation steps, and best practices for optimizing WHMCS infrastructure.

## Prerequisites

- Existing WHMCS installation
- Load balancing infrastructure
- Database replication capability
- Monitoring system
- CDN integration (recommended)

## Workflow Steps

### Step 1: Vertical Scaling Assessment

```bash
#!/bin/bash
# /opt/scripts/assess_vertical_scaling.sh

echo "=== WHMCS Vertical Scaling Assessment ==="
echo "Generated: $(date)"
echo ""

# CPU Analysis
echo "=== CPU Resources ==="
echo "CPU Model: $(cat /proc/cpuinfo | grep 'model name' | head -1)"
echo "CPU Cores: $(nproc)"
echo "Current Load: $(uptime | awk -F'load average:' '{print $2}')"

# Memory Analysis
echo ""
echo "=== Memory Resources ==="
free -h
echo ""
echo "Recommended for WHMCS:"
echo "- Minimum: 8GB RAM for <1000 users"
echo "- Recommended: 16GB RAM for 1000-5000 users"
echo "- High Usage: 32GB+ RAM for 5000+ users"

# PHP-FPM Analysis
echo ""
echo "=== PHP-FPM Configuration ==="
cat /etc/php/8.1/fpm/pool.d/www.conf | grep -E "pm\.|pm\.|max_children"
echo ""
echo "Current pm.max_children: $(grep 'pm.max_children' /etc/php/8.1/fpm/pool.d/www.conf)"

# Calculate recommended settings
php_memory_limit_mb=256
expected_concurrent=500
recommended_children=$((expected_concurrent / 2))
recommended_ondemand=$((expected_concurrent / 4))

echo ""
echo "=== Vertical Scaling Recommendations ==="
echo "PHP-FPM Settings:"
echo "- pm.max_children: $recommended_children (current: needs adjustment)"
echo "- pm.start_servers: $((recommended_children / 4))"
echo "- pm.min_spare_servers: $((recommended_children / 4))"
echo "- pm.max_spare_servers: $((recommended_children / 2))"
```

### Step 2: Horizontal Scaling Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                    WHMCS Horizontal Scaling Architecture         │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│                         ┌──────────────┐                        │
│                         │   CDN/WAF    │                        │
│                         │  (Cloudflare)│                        │
│                         └──────┬───────┘                        │
│                                │                                │
│                    ┌───────────┴───────────┐                    │
│                    │    Load Balancer       │                    │
│                    │  (Nginx/HAProxy)       │                    │
│                    └───────┬─────┬─────────┘                    │
│                            │     │                              │
│            ┌───────────────┼─────┼───────────────┐             │
│            │               │     │               │             │
│     ┌──────▼──────┐  ┌──────▼──────┐  ┌──────────▼────────┐    │
│     │   Web 1    │  │   Web 2     │  │     Web N          │    │
│     │ (PHP-FPM)  │  │ (PHP-FPM)   │  │    (PHP-FPM)       │    │
│     └──────┬─────┘  └──────┬─────┘  └──────────┬─────────┘    │
│            │               │                   │               │
│            └───────────────┼───────────────────┘               │
│                            │                                   │
│                    ┌───────▼───────┐                          │
│                    │  Shared Storage │                         │
│                    │  (NFS/GlusterFS)│                         │
│                    └───────┬───────┘                          │
│                            │                                   │
│     ┌──────────────────────┼──────────────────────────┐         │
│     │                      │                          │         │
│ ┌───▼───┐  ┌───────────────┴────────┐  ┌─────────────▼─────┐   │
│ │Primary│  │  Read Replica 1       │  │   Read Replica N  │   │
│ │ MySQL │  │  (SELECT queries)     │  │   (SELECT queries)│   │
│ └───────┘  └───────────────────────┘  └───────────────────┘   │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### Step 3: Load Balancer Configuration

```nginx
# /etc/nginx/conf.d/whmcs-load-balancer.conf

upstream whmcs_backend {
    # Least connections algorithm
    least_conn;
    
    # Web server 1
    server 10.0.1.10:8080 weight=5 max_fails=3 fail_timeout=30s;
    
    # Web server 2
    server 10.0.1.11:8080 weight=5 max_fails=3 fail_timeout=30s;
    
    # Web server 3
    server 10.0.1.12:8080 weight=5 max_fails=3 fail_timeout=30s;
    
    # Backup server
    server 10.0.1.99:8080 backup;
    
    # Keepalive connections
    keepalive 32;
}

server {
    listen 443 ssl http2;
    server_name whmcs.example.com;
    
    # SSL Configuration
    ssl_certificate /etc/nginx/ssl/whmcs.crt;
    ssl_certificate_key /etc/nginx/ssl/whmcs.key;
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers HIGH:!aNULL:!MD5;
    ssl_prefer_server_ciphers on;
    
    # Security Headers
    add_header X-Frame-Options "SAMEORIGIN" always;
    add_header X-Content-Type-Options "nosniff" always;
    add_header X-XSS-Protection "1; mode=block" always;
    
    # Compression
    gzip on;
    gzip_types text/plain text/css application/json application/javascript;
    gzip_min_length 1000;
    
    # Static files caching
    location ~* \.(jpg|jpeg|png|gif|ico|css|js|woff|woff2)$ {
        expires 30d;
        add_header Cache-Control "public, immutable";
        access_log off;
    }
    
    # PHP handling
    location ~ \.php$ {
        proxy_pass http://whmcs_backend;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        
        # Timeout settings
        proxy_connect_timeout 60s;
        proxy_send_timeout 60s;
        proxy_read_timeout 60s;
        
        # Buffering
        proxy_buffering on;
        proxy_buffer_size 128k;
        proxy_buffers 4 256k;
    }
    
    # Admin area - more strict
    location /whmcs/admin/ {
        proxy_pass http://whmcs_backend;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        
        # Rate limiting for admin
        limit_req zone=admin_limit burst=10 nodelay;
    }
    
    # Health check endpoint
    location /health {
        access_log off;
        return 200 "healthy\n";
        add_header Content-Type text/plain;
    }
}

# Rate limiting zones
limit_req_zone $binary_remote_addr zone=admin_limit:10m rate=5r/s;
limit_req_zone $binary_remote_addr zone=general_limit:10m rate=30r/s;
```

### Step 4: Database Scaling Configuration

```sql
-- MySQL Replication Configuration (Primary)
-- /etc/mysql/mysql.conf.d/whmcs-primary.cnf

[mysqld]
# Server ID (must be unique)
server-id=1

# Binary logging for replication
log-bin=mysql-bin
binlog-format=ROW
binlog-rows-query-log-events=ON
expire-logs-days=7
max-binlog-size=100M

# InnoDB settings for high performance
innodb-buffer-pool-size=8G  -- Adjust to 70% of available RAM
innodb-log-file-size=1G
innodb-flush-log-at-trx-commit=2
innodb-flush-method=O_DIRECT
innodb-file-per-table=ON

# Connection settings
max-connections=500
wait-timeout=600
interactive-timeout=600

# Query cache (MySQL 8.0+ removed, using ProxySQL instead)
# For MySQL 5.7: query_cache_type=0 (disabled)

# Slow query log
slow-query-log=1
slow-query-log-file=/var/log/mysql/slow.log
long-query-time=2

# Replication settings
relay-log=mysql-relay
relay-log-index=mysql-relay.index
replicate-do-db=whmcs_main

# Read/Write Split (disable on primary)
read-only=OFF
super-read-only=OFF

-- MySQL Read Replica Configuration
-- /etc/mysql/mysql.conf.d/whmcs-replica.cnf

[mysqld]
server-id=2

# Disable binary logging on replicas
log-bin=OFF
relay-log=mysql-relay

# Same InnoDB settings
innodb-buffer-pool-size=8G
innodb-log-file-size=1G

# Read-only mode
read-only=ON
super-read-only=ON

# Disable replication write
slave-skip-errors=1062
```

### Step 5: PHP-FPM Scaling Configuration

```php
; /etc/php/8.1/fpm/pool.d/whmcs.conf

[whmcs]
; Pool name
user = www-data
group = www-data

; Socket configuration
listen = /run/php/php8.1-fpm-whmcs.sock
listen.owner = www-data
listen.group = www-data
listen.mode = 0660

; Process manager configuration
pm = dynamic
pm.max_children = 50          ; Based on available memory (50 * 256MB = 12.8GB)
pm.start_servers = 10
pm.min_spare_servers = 10
pm.max_spare_servers = 25
pm.max_requests = 500        ; Prevents memory leaks

; Timeout settings
request_terminate_timeout = 60s
request_slowlog_timeout = 10s

; Logging
slowlog = /var/log/php8.1-fpm-slow.log
php_admin_flag[log_errors] = on
php_admin_value[error_log] = /var/log/php8.1-fpm-whmcs-error.log

; Chroot settings
chroot = /var/www/whmcs
chdir = /

; Environment variables
env[HOSTNAME] = $HOSTNAME
env[PATH] = /usr/local/bin:/usr/bin:/bin
env[TMP] = /tmp
env[TMPDIR] = /tmp
env[TEMP] = /tmp

; PHP settings
php_admin_value[memory_limit] = 256M
php_admin_value[post_max_size] = 64M
php_admin_value[upload_max_filesize] = 64M
php_admin_value[max_execution_time] = 60
php_admin_value[max_input_time] = 60
```

### Step 6: Session Management for Scaling

```php
<?php
// /var/www/whmcs/includes/session_scaling.php

/**
 * WHMCS Session Scaling Configuration
 * Use Redis/Memcached for distributed sessions
 */

// Database sessions (for multi-server setup)
ini_set('session.save_handler', 'database');

// Custom session handler for Redis
class WHMCSSessionHandler {
    private $redis;
    private $ttl = 7200; // 2 hours
    
    public function __construct() {
        $this->redis = new Redis();
        $this->redis->connect(REDIS_HOST, REDIS_PORT);
        $this->redis->auth(REDIS_PASSWORD);
        $this->redis->select(1); // Database 1 for sessions
    }
    
    public function open($path, $name): bool {
        return true;
    }
    
    public function close(): bool {
        return true;
    }
    
    public function read($sessionId): string {
        $key = 'whmcs_session:' . $sessionId;
        $data = $this->redis->get($key);
        return $data ?: '';
    }
    
    public function write($sessionId, $data): bool {
        $key = 'whmcs_session:' . $sessionId;
        $this->redis->setex($key, $this->ttl, $data);
        
        // Update activity for monitoring
        $this->redis->zadd('whmcs_active_sessions', time(), $sessionId);
        
        return true;
    }
    
    public function destroy($sessionId): bool {
        $key = 'whmcs_session:' . $sessionId;
        $this->redis->del($key);
        $this->redis->zrem('whmcs_active_sessions', $sessionId);
        return true;
    }
    
    public function gc($maxlifetime): bool {
        // Redis handles this automatically with TTL
        return true;
    }
}

// Register session handler
$handler = new WHMCSSessionHandler();
session_set_save_handler(
    [$handler, 'open'],
    [$handler, 'close'],
    [$handler, 'read'],
    [$handler, 'write'],
    [$handler, 'destroy'],
    [$handler, 'gc']
);

// Start session
session_start();

// Helper function to get active session count
function get_active_session_count(): int {
    $redis = new Redis();
    $redis->connect(REDIS_HOST, REDIS_PORT);
    $redis->auth(REDIS_PASSWORD);
    $redis->select(1);
    
    // Count sessions from last 10 minutes
    $ten_minutes_ago = time() - 600;
    return $redis->zCount('whmcs_active_sessions', $ten_minutes_ago, '+inf');
}
```

### Step 7: Auto-Scaling Implementation

```yaml
# /opt/kubernetes/whmcs-autoscaling.yaml

apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: whmcs-web-autoscaler
  namespace: whmcs
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: whmcs-web
  minReplicas: 3
  maxReplicas: 30
  metrics:
    # CPU-based scaling
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 65
    
    # Memory-based scaling
    - type: Resource
      resource:
        name: memory
        target:
          type: Utilization
          averageUtilization: 75
    
    # Request rate scaling
    - type: Pods
      pods:
        metric:
          name: http_requests_per_second
        target:
          type: AverageValue
          averageValue: "100"
  
  behavior:
    # Scale up quickly
    scaleUp:
      stabilizationWindowSeconds: 0
      policies:
        - type: Percent
          value: 100
          periodSeconds: 15
        - type: Pods
          value: 4
          periodSeconds: 15
    
    # Scale down slowly (avoid thrashing)
    scaleDown:
      stabilizationWindowSeconds: 300
      policies:
        - type: Percent
          value: 10
          periodSeconds: 60

---
# Vertical Pod Autoscaler for resource recommendations
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: whmcs-web-vpa
  namespace: whmcs
spec:
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: whmcs-web
  updatePolicy:
    updateMode: "Off"  # Recommend only, don't auto-apply
  resourcePolicy:
    containerPolicies:
      - containerName: whmcs-php
        minAllowed:
          cpu: 250m
          memory: 256Mi
        maxAllowed:
          cpu: 4
          memory: 2Gi
```

## Scaling Metrics

| Metric | Current | Scaled Level 1 | Scaled Level 2 | Scaled Level 3 |
|--------|---------|----------------|----------------|-----------------|
| Web Servers | 1 | 3 | 5 | 10+ |
| PHP Workers | 10 | 30 | 50 | 100+ |
| Database | Single | Primary + 1 Replica | Primary + 2 Replicas | Sharded |
| Memory per Server | 8GB | 16GB | 32GB | 64GB |
| Concurrent Users | 100 | 500 | 1,000 | 5,000+ |

## Best Practices

1. **Measure Before Scaling**: Use metrics to drive scaling decisions
2. **Scale Incrementally**: Add capacity in small increments
3. **Use Load Testing**: Verify scaling effectiveness
4. **Monitor Performance**: Track key metrics after scaling
5. **Plan for Peak**: Capacity should handle 2-3x normal load
6. **Document Configuration**: Keep scaling configurations in version control
7. **Automate Where Possible**: Use auto-scaling for dynamic workloads

## Common Pitfalls

- **Over-scaling**: Wasting resources and money
- **Under-scaling**: Poor user experience during peaks
- **Single Point of Failure**: Not distributing components
- **Ignoring Database**: Database often becomes bottleneck
- **Session Issues**: Not handling distributed sessions
- **Cache Invalidation**: Not properly invalidating caches

## Verification Checklist

- [ ] Load balancer configured and tested
- [ ] Multiple web servers deployed
- [ ] Database replication configured
- [ ] Session handling distributed
- [ ] Auto-scaling policies defined
- [ ] Load testing performed
- [ ] Failover tested
- [ ] Performance benchmarks established
- [ ] Monitoring dashboards created
- [ ] Scaling runbook documented

## Related Documentation

- [WHMCS Load Balancing](whmcs-load-balancing.md)
- [WHMCS Load Testing](whmcs-load-testing-workflow.md)
- [WHMCS Capacity Planning](whmcs-capacity-planning.md)