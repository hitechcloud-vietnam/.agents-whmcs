# WHMCS Cache Server Setup Workflow

## Description
Configure dedicated cache server for WHMCS to improve performance.

## Prerequisites
- Dedicated server or VM for cache
- Redis or Memcached
- Network connectivity to WHMCS server
- SSH access

## Supported Cache Backends
- Redis (recommended)
- Memcached
- APCu (local only)

## Steps

### Step 1: Choose Cache Backend

| Feature | Redis | Memcached |
|---------|-------|-----------|
| Persistence | Yes | No |
| Data Types | Multiple | Strings only |
| Clustering | Native | External |
| Memory Efficiency | Good | Better |
| Recommended | Yes | Alternative |

### Step 2: Install Redis Server
```bash
# On cache server
apt update && apt upgrade -y
apt install -y redis-server

# Configure Redis
cat > /etc/redis/redis.conf << 'EOF'
bind 0.0.0.0
protected-mode no
port 6379
tcp-backlog 65535
timeout 0
tcp-keepalive 300
daemonize yes
supervised systemd
pidfile /var/run/redis/redis.pid
loglevel notice
logfile /var/log/redis/redis-server.log
databases 16
save 900 1
save 300 10
save 60 10000
stop-writes-on-bgsave-error yes
rdbcompression yes
rdbchecksum yes
dbfilename dump.rdb
dir /var/lib/redis
maxmemory 2gb
maxmemory-policy allkeys-lru
appendonly yes
appendfilename "appendonly.aof"
appendfsync everysec
no-appendfsync-on-rewrite no
auto-aof-rewrite-percentage 100
auto-aof-rewrite-min-size 64mb
EOF

# Set password
redis-cli CONFIG SET requirepass "your_redis_password"

# Restart and enable
systemctl restart redis-server
systemctl enable redis-server
```

### Step 3: Configure Firewall
```bash
# On cache server
ufw allow from whmcs_server_ip to any port 6379
ufw enable
```

### Step 4: Install Redis on WHMCS Server
```bash
# On WHMCS server
apt install -y php-redis redis-tools

# Restart PHP-FPM
systemctl restart php*-fpm
```

### Step 5: Configure WHMCS for Redis
```php
// Add to configuration.php

// Redis Configuration
$redis_host = 'cache_server_ip';
$redis_port = 6379;
$redis_password = 'your_redis_password';
$redis_database = 0;

// Use Redis for caching
$whmcs_cache_driver = 'redis';
```

### Step 6: Configure Laravel Cache (WHMCS 8+)
```php
// Create bootstrap/cache.php or add to configuration.php

return [
    'default' => 'redis',
    
    'stores' => [
        'redis' => [
            'driver' => 'redis',
            'connection' => 'cache',
            'lock_connection' => 'default',
        ],
    ],
    
    'prefix' => 'whmcs_cache_',
];
```

### Step 7: Test Redis Connection
```bash
# Test from WHMCS server
redis-cli -h cache_server_ip -p 6379 -a your_redis_password ping

# Expected response: PONG
```

### Step 8: Configure PHP OPcache
```bash
# Edit PHP configuration
cat >> /etc/php/8.2/fpm/php.ini << 'EOF'
; OPcache settings
opcache.enable=1
opcache.memory_consumption=256
opcache.interned_strings_buffer=16
opcache.max_accelerated_files=10000
opcache.revalidate_freq=0
opcache.validate_timestamps=0
opcache.save_comments=1
EOF

systemctl restart php8.2-fpm
```

### Step 9: Enable WHMCS Caching
```php
// Add to configuration.php - Additional cache settings

// File cache configuration
$whmcsupeu_config['php_cache'] = [
    'enabled' => true,
    'driver' => 'redis',
    'prefix' => 'whmcs_',
    'ttl' => 3600,
];
```

### Step 10: Verify Cache Operation
```php
<?php
// Create test script: test-cache.php
require_once __DIR__ . '/init.php';

use WHMCS\Database\Capsule;

// Test Redis connection
try {
    $redis = new Redis();
    $redis->connect(REDIS_HOST, REDIS_PORT);
    $redis->auth(REDIS_PASSWORD);
    $redis->set('test_key', 'test_value');
    $value = $redis->get('test_key');
    
    if ($value === 'test_value') {
        echo "Redis cache: WORKING\n";
    } else {
        echo "Redis cache: FAILED\n";
    }
} catch (Exception $e) {
    echo "Redis error: " . $e->getMessage() . "\n";
}

// Test WHMCS cache
try {
    $cacheKey = 'test_' . time();
    Capsule::Capsule::getContainer()->make('cache')->put($cacheKey, 'test_data', 60);
    $cached = Capsule::getContainer()->make('cache')->get($cacheKey);
    
    if ($cached === 'test_data') {
        echo "WHMCS cache: WORKING\n";
    }
} catch (Exception $e) {
    echo "WHMCS cache error: " . $e->getMessage() . "\n";
}
```

### Step 11: Monitor Cache Performance
```bash
# Create monitoring script
cat > /usr/local/bin/redis-monitor.sh << 'EOF'
#!/bin/bash

REDIS_HOST="cache_server_ip"
REDIS_PORT="6379"
REDIS_PASS="your_redis_password"
ALERT_EMAIL="admin@example.com"

# Get Redis info
INFO=$(redis-cli -h $REDIS_HOST -p $REDIS_PORT -a $REDIS_PASS INFO)

# Extract metrics
USED_MEM=$(echo "$INFO" | grep "used_memory_human" | cut -d: -f2 | tr -d '\r')
CONNECTED_CLIENTS=$(echo "$INFO" | grep "connected_clients" | cut -d: -f2 | tr -d '\r')
KEYS=$(echo "$INFO" | grep "db0" | grep "keys" | cut -d, -f1 | cut -d= -f2)
HIT_RATE=$(echo "$INFO" | grep "keyspace_hits" | awk -F'[/:]' '{print $2}')

echo "=== Redis Cache Status ==="
echo "Memory Used: $USED_MEM"
echo "Connected Clients: $CONNECTED_CLIENTS"
echo "Keys: $KEYS"
echo "Hit Rate: $HIT_RATE"

# Alert if memory > 90%
MEM_PERCENT=$(echo "$INFO" | grep "used_memory:" | cut -d: -f2)
MAX_MEM=$(echo "$INFO" | grep "maxmemory:" | cut -d: -f2)
if [ ! -z "$MAX_MEM" ] && [ "$MAX_MEM" -gt 0 ]; then
    PERCENT=$((MEM_PERCENT * 100 / MAX_MEM))
    if [ $PERCENT -gt 90 ]; then
        echo "WARNING: Redis memory at ${PERCENT}%" | mail -s "Redis Memory Alert" $ALERT_EMAIL
    fi
fi
EOF

chmod +x /usr/local/bin/redis-monitor.sh

# Add to cron
echo "*/5 * * * * /usr/local/bin/redis-monitor.sh" | crontab -
```

## Troubleshooting

### Connection Issues
```bash
# Check Redis is running
systemctl status redis-server

# Check port is open
netstat -tlnp | grep 6379

# Test network connectivity
telnet cache_server_ip 6379
```

### Performance Issues
```bash
# Monitor Redis slow log
redis-cli -h $REDIS_HOST slowlog get 10

# Check memory fragmentation
redis-cli -h $REDIS_HOST info memory
```

## Tags
- cache
- redis
- performance
- optimization