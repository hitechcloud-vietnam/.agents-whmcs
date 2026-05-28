# WHMCS Load Balancing Workflow

## Purpose

Configure and manage load balancing for WHMCS to ensure high availability, improve performance, and handle traffic spikes. This workflow covers setup, configuration, health monitoring, and failover handling.

## Prerequisites

- Multiple WHMCS server instances
- Load balancer (HAProxy, Nginx, AWS ALB, etc.)
- Shared storage (NFS, GlusterFS, EFS)
- Database replication configured
- SSL certificates prepared

## Workflow Steps

### Step 1: Architecture Design

Design a load-balanced WHMCS architecture:

```
                    ┌─────────────────┐
                    │   Load Balancer  │
                    │   (HAProxy/Nginx)│
                    └────────┬────────┘
                             │
         ┌───────────────────┼───────────────────┐
         │                   │                   │
    ┌────▼────┐         ┌────▼────┐         ┌────▼────┐
    │ WHMCS 1 │         │ WHMCS 2 │         │ WHMCS 3 │
    │ App     │         │ App     │         │ App     │
    └────┬────┘         └────┬────┘         └────┬────┘
         │                   │                   │
         └───────────────────┼───────────────────┘
                             │
                    ┌────────▼────────┐
                    │  Shared Storage  │
                    │  (NFS/GlusterFS) │
                    └──────────────────┘
```

### Step 2: Configure Shared Storage

Set up shared storage for WHMCS files:

```bash
#!/bin/bash
# /opt/scripts/setup_shared_storage.sh

# On NFS server
echo "/var/www/whmcs *(rw,sync,no_subtree_check,no_root_squash)" >> /etc/exports
exportfs -a
systemctl enable nfs-server
systemctl start nfs-server

# On each WHMCS app server
apt-get install -y nfs-common
mkdir -p /var/www/whmcs
mount nfs-server:/var/www/whmcs /var/www/whmcs
echo "nfs-server:/var/www/whmcs /var/www/whmcs nfs defaults 0 0" >> /etc/fstab

# Set permissions
chown -R www-data:www-data /var/www/whmcs
find /var/www/whmcs -type d -exec chmod 755 {} \;
find /var/www/whmcs -type f -exec chmod 644 {} \;
```

### Step 3: Configure Database Replication

Set up MySQL master-slave replication:

```bash
# Master server (my.cnf)
[mysqld]
server-id=1
log-bin=mysql-bin
binlog-do-db=whmcs_main
sync-binlog=1
innodb_flush_log_at_trx_commit=1

# Slave server (my.cnf)
[mysqld]
server-id=2
relay-log=relay-bin
read-only=1

# On master: Create replication user
CREATE USER 'repl'@'%' IDENTIFIED BY 'repl_password';
GRANT REPLICATION SLAVE ON *.* TO 'repl'@'%';
FLUSH PRIVILEGES;

# On master: Get binary log position
SHOW MASTER STATUS;
# Note: File=mysql-bin.000001, Position=123

# On slave: Configure and start replication
CHANGE MASTER TO
    MASTER_HOST='master_ip',
    MASTER_USER='repl',
    MASTER_PASSWORD='repl_password',
    MASTER_LOG_FILE='mysql-bin.000001',
    MASTER_LOG_POS=123;

START SLAVE;
SHOW SLAVE STATUS\G
```

### Step 4: Configure HAProxy Load Balancer

Setup HAProxy for WHMCS:

```bash
# /etc/haproxy/haproxy.cfg

global
    log /dev/log local0
    log /dev/log local1 notice
    chroot /var/lib/haproxy
    stats socket /run/haproxy/admin.sock mode 660 level admin
    stats timeout 30s
    user haproxy
    group haproxy
    daemon
    maxconn 4000
    ssl-default-bind-ciphers ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256
    ssl-default-bind-options no-sslv3

defaults
    log global
    mode http
    option httplog
    option dontlognull
    option http-server-close
    option forwardfor except 127.0.0.0/8
    option redispatch
    retries 3
    timeout connect 5000
    timeout client 50000
    timeout server 50000

# WHMCS Frontend
frontend whmcs_frontend
    bind *:80
    bind *:443 ssl crt /etc/ssl/certs/whmcs.pem
    mode http
    
    # Redirect to HTTPS
    http-request redirect scheme https if !{ ssl_fc }
    
    # Sticky sessions for admin area
    acl is_admin path_beg /admin /whmcs/admin
    use_backend whmcs_admin if is_admin
    
    default_backend whmcs_servers

# Admin backend (sticky sessions)
backend whmcs_admin
    balance source
    option httpchk GET /whmcs/index.php
    option forwardfor
    http-check expect status 200
    server whmcs1 10.0.1.10:443 check ssl inter 2000 rise 2 fall 3
    server whmcs2 10.0.1.11:443 check ssl inter 2000 rise 2 fall 3
    server whmcs3 10.0.1.12:443 check ssl inter 2000 rise 2 fall 3

# Public backend (least connections)
backend whmcs_servers
    balance leastconn
    option httpchk GET /whmcs/index.php
    option forwardfor
    http-check expect status 200
    server whmcs1 10.0.1.10:443 check ssl inter 2000 rise 2 fall 3
    server whmcs2 10.0.1.11:443 check ssl inter 2000 rise 2 fall 3
    server whmcs3 10.0.1.12:443 check ssl inter 2000 rise 2 fall 3

# Stats page
listen stats
    bind *:8404
    stats enable
    stats uri /stats
    stats refresh 30s
    stats auth admin:password
```

### Step 5: Configure PHP-FPM for Multiple Servers

```bash
# /etc/php/8.1/fpm/pool.d/whmcs.conf

[whmcs]
user = www-data
group = www-data
listen = /run/php/php-fpm-whmcs.sock
listen.owner = www-data
listen.group = www-data

pm = dynamic
pm.max_children = 100
pm.start_servers = 20
pm.min_spare_servers = 10
pm.max_spare_servers = 30
pm.max_requests = 500

; Health check
ping.path = /ping
ping.response = pong

; Logging
php_admin_value[error_log] = /var/log/php-fpm/whmcs-error.log
php_admin_flag[log_errors] = on

; Performance
php_value[session.save_handler] = redis
php_value[session.save_path] = "tcp://redis-server:6379"
php_value[opcache.enable] = 1
php_value[opcache.memory_consumption] = 256
php_value[opcache.max_accelerated_files] = 10000
```

### Step 6: Configure Redis Session Storage

```php
# configuration.php additions for Redis sessions

// Session handling via Redis
ini_set('session.save_handler', 'redis');
ini_set('session.save_path', 'tcp://redis-server:6379?database=0');

// For addon modules
$redis = new Redis();
$redis->connect('redis-server', 6379);
$redis->select(1); // Use database 1 for cache

// WHMCS configuration.php
$redis_config = [
    'host' => 'redis-server',
    'port' => 6379,
    'database' => 1,
    'prefix' => 'whmcs_'
];
```

### Step 7: Configure Health Checks

```bash
#!/bin/bash
# /opt/scripts/health_check.sh

WHXCS_SERVERS=("10.0.1.10" "10.0.1.11" "10.0.1.12")
HEALTH_ENDPOINT="/whmcs/index.php"
TIMEOUT=5

check_server() {
    local ip=$1
    local status=$(curl -s -o /dev/null -w "%{http_code}" \
        --max-time $TIMEOUT \
        "https://$ip$HEALTH_ENDPOINT")
    
    if [ "$status" = "200" ]; then
        echo "OK"
        return 0
    else
        echo "FAILED (HTTP $status)"
        return 1
    fi
}

# Monitor each server
for server in "${WHXCS_SERVERS[@]}"; do
    result=$(check_server "$server")
    timestamp=$(date '+%Y-%m-%d %H:%M:%S')
    
    if [ $? -eq 0 ]; then
        echo "[$timestamp] $server: $result"
    else
        echo "[$timestamp] $server: $result" >&2
        # Alert via monitoring system
        /opt/scripts/alert.sh "WHMCS server $server is down"
    fi
done
```

### Step 8: Setup Failover Configuration

```bash
#!/bin/bash
# /opt/scripts/failover.sh
# Automatic failover script for HAProxy

VIP="10.0.0.100"
KEEPALIVED_STATE="/var/run/keepalived.state"

monitor_vip() {
    ip addr show | grep -q "$VIP"
}

become_master() {
    echo "Master: $1" > $KEEPALIVED_STATE
    ip addr add $VIP/24 dev eth0
}

become_backup() {
    echo "Backup: $1" > $KEEPALIVED_STATE
    ip addr del $VIP/24 dev eth0 2>/dev/null || true
}

# KeepAlived config /etc/keepalived/keepalived.conf
# vrrp_instance VI_1 {
#     state BACKUP
#     interface eth0
#     virtual_router_id 51
#     priority 100
#     advert_int 1
#     virtual_ipaddress {
#         10.0.0.100
#     }
# }
```

## Best Practices

1. **Session Management**: Use Redis for centralized session storage
2. **Static Assets**: Offload to CDN or separate domain
3. **Health Checks**: Implement comprehensive endpoint checks
4. **SSL Termination**: Handle SSL at load balancer level
5. **Sticky Sessions**: Use for admin area, stateless for public
6. **Rate Limiting**: Apply at load balancer for DDoS protection
7. **Monitoring**: Track latency, error rates, and server health

## Common Pitfalls

- **Session loss**: Not using shared session storage
- **File sync issues**: Incomplete shared storage configuration
- **Database bottleneck**: Single master can't handle writes
- **SSL issues**: Certificate mismatch or outdated ciphers
- **Cache stampede**: No cache warming on failure
- **Health check false positives**: Checks too aggressive

## Verification Checklist

- [ ] All servers respond to health checks
- [ ] Sessions persist across server restarts
- [ ] Database replication lag under 1 second
- [ ] Failover works automatically
- [ ] SSL certificates valid and match
- [ ] Performance under load tested
- [ ] Logs centralized and monitored
- [ ] Rollback procedure documented

## Related Documentation

- [WHMCS Multi-Server Workflow](whmcs-multi-server-workflow.md)
- [WHMCS Disaster Recovery](whmcs-disaster-recovery-workflow.md)
- [WHMCS Performance Audit](whmcs-performance-audit.md)