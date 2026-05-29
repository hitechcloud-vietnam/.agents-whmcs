# WHMCS High Availability Setup Workflow

## Description
Guide for configuring WHMCS in a high-availability architecture.

## Prerequisites
- Multiple servers
- Load balancer
- Shared storage (NFS/GlusterFS)
- Database clustering (MySQL Galera/Replication)
- Redis cluster for sessions

## Architecture Overview
```
                    [Load Balancer]
                         |
         +---------------+---------------+
         |               |               |
      [Server1]      [Server2]      [Server3]
         |               |               |
         +---------------+---------------+
                         |
                   [Shared Storage]
                         |
                   [Database Cluster]
                         |
                   [Redis Cluster]
```

## Steps

### Step 1: Prepare Database Cluster

**MySQL Galera Setup:**
```bash
# Install MariaDB Galera on all DB nodes
apt install -y mariadb-server mariadb-client galera-4

# Node 1 - Initialize
cat > /etc/mysql/maria.conf.d/galera.cnf << 'EOF'
[galera]
wsrep_on = ON
wsrep_provider = /usr/lib/galera/libgalera_smm.so
wsrep_cluster_name = "whmcs_cluster"
wsrep_cluster_address = "gcomm://node1_ip,node2_ip,node3_ip"
wsrep_node_name = node1
wsrep_node_address = node1_ip
binlog_format = row
default_storage_engine = InnoDB
innodb_autoinc_lock_mode = 2
EOF

# Start cluster on node 1
galera_new_cluster

# Join nodes 2 and 3
systemctl start mariadb
```

### Step 2: Setup Redis Cluster
```bash
# Install Redis on all nodes
apt install -y redis-server

# Configure Redis cluster
# Node 1
cat > /etc/redis/redis.conf << 'EOF'
bind 0.0.0.0
port 6379
cluster-enabled yes
cluster-config-file nodes.conf
cluster-node-timeout 5000
appendonly yes
EOF

# Create Redis cluster
redis-cli --cluster create node1_ip:6379 node2_ip:6379 node3_ip:6379 \
    --cluster-replicas 1
```

### Step 3: Setup Shared Storage
```bash
# Install NFS server on storage node
apt install -y nfs-kernel-server

# Export shared storage
cat >> /etc/exports << 'EOF'
/var/www/whmcs_shared *(rw,sync,no_subtree_check,no_root_squash)
EOF

# Export and restart
exportfs -a
systemctl restart nfs-server

# On WHMCS app servers, mount NFS
apt install -y nfs-common
mount -t nfs nfs_server:/var/www/whmcs_shared /var/www/whmcs
```

### Step 4: Configure WHMCS for HA
```php
// In configuration.php
$db_host = "cluster_vip";  // Use virtual IP from load balancer
$db_username = "whmcs_ha";
$db_password = "secure_password";
$db_name = "whmcs";

// Redis for sessions
$redis_host = "redis_cluster_vip";
$redis_port = 6379;
$redis_password = "redis_password";
$redis_database = 0;

// Shared storage paths
$attachments_dir = '/var/www/whmcs_shared/attachments';
$downloads_dir = '/var/www/whmcs_shared/downloads';
$templates_c_dir = '/var/www/whmcs_shared/templates_c';
```

### Step 5: Setup Load Balancer
```nginx
# HAProxy configuration
global
    log /dev/log local0
    log /dev/log local1 notice
    chroot /var/lib/haproxy
    stats socket /run/haproxy/admin.sock mode 660 level admin
    maxconn 4096
    user haproxy
    group haproxy
    daemon

defaults
    log global
    mode http
    option httplog
    option dontlognull
    timeout connect 5000ms
    timeout client 50000ms
    timeout server 50000ms

frontend whmcs_frontend
    bind *:443 ssl crt /etc/ssl/private/whmcs.pem
    default_backend whmcs_backend

backend whmcs_backend
    balance roundrobin
    option httpchk GET /healthcheck.php
    http-check expect status 200
    server whmcs1 10.0.0.11:443 check inter 5s fall 2 rise 2
    server whmcs2 10.0.0.12:443 check inter 5s fall 2 rise 2
    server whmcs3 10.0.0.13:443 check inter 5s fall 2 rise 2
```

### Step 6: Create Health Check Script
```php
<?php
// /var/www/whmcs/healthcheck.php
header('Content-Type: application/json');
header('Cache-Control: no-cache');

// Check database connection
try {
    $pdo = new PDO(
        'mysql:host='.DB_HOST.';dbname='.DB_NAME,
        DB_USERNAME,
        DB_PASSWORD,
        [PDO::ATTR_TIMEOUT => 5]
    );
    $pdo->query('SELECT 1');
    $db_status = 'ok';
} catch (Exception $e) {
    $db_status = 'error: ' . $e->getMessage();
}

// Check Redis
try {
    $redis = new Redis();
    $redis->connect(REDIS_HOST, REDIS_PORT);
    if (defined('REDIS_PASSWORD')) {
        $redis->auth(REDIS_PASSWORD);
    }
    $redis->ping();
    $redis_status = 'ok';
} catch (Exception $e) {
    $redis_status = 'error: ' . $e->getMessage();
}

// Return status
$status = [
    'status' => ($db_status === 'ok' && $redis_status === 'ok') ? 'healthy' : 'unhealthy',
    'timestamp' => date('c'),
    'checks' => [
        'database' => $db_status,
        'redis' => $redis_status
    ]
];

http_response_code($status['status'] === 'healthy' ? 200 : 503);
echo json_encode($status);
```

### Step 7: Sync Cron Jobs
```bash
# Only run cron on one node (leader election)
# Using cron with flock
cat >> /etc/cron.d/whmcs_ha << 'EOF'
# WHMCS Cron - HA synchronized
*/5 * * * * root /usr/bin/flock -n /var/lock/whmcs_cron.lock -c "/usr/bin/php -q /var/www/whmcs/crons/cron.php"
EOF
```

### Step 8: Session Handling
```php
// In configuration.php - Use Redis for sessions
ini_set('session.save_handler', 'redis');
ini_set('session.save_path', 'tcp://redis_cluster_vip:6379?auth=redis_password&database=0');

// Or use WHMCS session configuration
$redis_config = [
    'host' => 'redis_cluster_vip',
    'port' => 6379,
    'password' => 'redis_password',
    'database' => 1,
    'prefix' => 'whmcs_session_'
];
```

## Monitoring & Alerts
```bash
# Create monitoring script
cat > /usr/local/bin/whmcs-ha-monitor.sh << 'EOF'
#!/bin/bash

ALERT_EMAIL="admin@example.com"
VIP="load_balancer_ip"

# Check VIP accessibility
if ! ping -c 1 $VIP &>/dev/null; then
    echo "VIP is down!" | mail -s "HA Alert: VIP Down" $ALERT_EMAIL
fi

# Check each backend
for server in 10.0.0.11 10.0.0.12 10.0.0.13; do
    if ! curl -s -f http://$server/healthcheck.php > /dev/null; then
        echo "Server $server is down" | mail -s "HA Alert: Server Down" $ALERT_EMAIL
    fi
done

# Check database cluster
mysql -h cluster_vip -u root -p -e "SHOW STATUS LIKE 'wsrep%';"

# Check Redis cluster
redis-cli -h redis_cluster_vip cluster info
EOF
```

## Failover Testing
```bash
# Test procedure
# 1. Bring down one WHMCS server
systemctl stop nginx

# 2. Verify traffic redirects
curl -I https://vip.whmcs.example.com

# 3. Bring server back
systemctl start nginx

# 4. Verify it rejoins cluster
php /var/www/whmcs/crons/healthcheck.php
```

## Tags
- high-availability
- clustering
- infrastructure
- redundancy
- failover