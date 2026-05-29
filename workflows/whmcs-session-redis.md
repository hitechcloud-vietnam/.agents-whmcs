# WHMCS Redis Sessions Workflow

## Description
Configure Redis for WHMCS session management for better performance and scalability.

## Prerequisites
- Redis server (local or remote)
- PHP Redis extension
- SSH access to WHMCS server

## Benefits
- Faster session handling
- Shared sessions across servers
- Survives PHP-FPM restarts
- Better memory management
- Supports session clustering

## Steps

### Step 1: Install Redis Extension
```bash
# Install PHP Redis extension
apt install -y php-redis

# Or for specific PHP version
apt install -y php8.2-redis

# Verify installation
php -m | grep redis
```

### Step 2: Configure Redis Server
```bash
# On Redis server
cat > /etc/redis/redis.conf << 'EOF'
bind 0.0.0.0
port 6379
protected-mode no
requirepass your_redis_password
daemonize yes
loglevel notice
logfile /var/log/redis/redis-server.log
databases 16
save 900 1
save 300 10
save 60 10000
stop-writes-on-bgsave-error yes
rdbcompression yes
dir /var/lib/redis
maxmemory 512mb
maxmemory-policy allkeys-lru
appendonly yes
appendfsync everysec
EOF

systemctl restart redis-server
```

### Step 3: Update Firewall
```bash
# Allow WHMCS server to connect
ufw allow from whmcs_server_ip to any port 6379
```

### Step 4: Configure WHMCS for Redis Sessions
```php
// Add to configuration.php

// Session configuration using Redis
ini_set('session.save_handler', 'redis');
ini_set('session.save_path', 'tcp://redis_host:6379?auth=your_redis_password&database=1&prefix=whmcs_sess_');

// Alternative: Use Unix socket
// ini_set('session.save_path', 'unix:///var/run/redis/redis.sock?auth=your_redis_password&database=1');
```

### Step 5: Configure Redis Database for Sessions
```bash
# Create dedicated database for sessions
redis-cli -h redis_host -p 6379 -a your_redis_password << 'EOF'
SELECT 1
CONFIG SET databases 16
EOF
```

### Step 6: Verify Session Storage
```php
<?php
// Create test script
session_start();

echo "Session ID: " . session_id() . "<br>";
echo "Session Save Handler: " . ini_get('session.save_handler') . "<br>";
echo "Session Save Path: " . ini_get('session.save_path') . "<br>";

// Store test data
$_SESSION['test'] = 'Hello from Redis!';
$_SESSION['timestamp'] = time();

echo "Session Data Stored<br>";

// Check Redis
echo "<pre>";
echo shell_exec("redis-cli -h redis_host -p 6379 -a your_redis_password KEYS 'whmcs_sess_*'");
echo "</pre>";

// Reload to verify persistence
session_write_close();
session_start();
echo "Retrieved: " . $_SESSION['test'] . "<br>";
```

### Step 7: Set Session Lifetime
```php
// Add to configuration.php

// Session timeout (in seconds)
$session_timeout = 86400; // 24 hours

ini_set('session.gc_maxlifetime', $session_timeout);
ini_set('session.cookie_lifetime', 0); // Session cookie
ini_set('session.cookie_httponly', 1);
ini_set('session.cookie_secure', 1);
ini_set('session.use_strict_mode', 1);
ini_set('session.use_only_cookies', 1);
```

### Step 8: Configure Cookie Domain
```php
// Add to configuration.php if using subdomains

$whmcs_config['cookie_domain'] = '.example.com';
$whmcs_config['cookie_path'] = '/';
$whmcs_config['cookie_secure'] = true;
$whmcs_config['cookie_httponly'] = true;
```

### Step 9: Performance Monitoring
```bash
# Create session monitoring script
cat > /usr/local/bin/whmcs-session-monitor.sh << 'EOF'
#!/bin/bash

REDIS_HOST="redis_host"
REDIS_PORT="6379"
REDIS_PASS="your_redis_password"

echo "=== WHMCS Session Statistics ==="
echo ""

# Count active sessions
SESSIONS=$(redis-cli -h $REDIS_HOST -p $REDIS_PORT -a $REDIS_PASS --no-raw KEYS 'whmcs_sess_*' | wc -l)
echo "Active Sessions: $SESSIONS"

# Memory usage
echo "Memory Usage:"
redis-cli -h $REDIS_HOST -p $REDIS_PORT -a $REDIS_PASS INFO memory | grep -E "used_memory_human|used_memory_peak_human"

# Connected clients
echo ""
echo "Connected Clients:"
redis-cli -h $REDIS_HOST -p $REDIS_PORT -a $REDIS_PASS CLIENT LIST | grep -c "cmd=auth"

# Session statistics
echo ""
echo "Session Details:"
redis-cli -h $REDIS_HOST -p $REDIS_PORT -a $REDIS_PASS DBSIZE
EOF

chmod +x /usr/local/bin/whmcs-session-monitor.sh
```

### Step 10: Handle Session Expiration
```php
// Optional: Hook to log session events
// Add to includes/hooks/session_hooks.php

<?php
use WHMCS\Session;

// Log session creation
add_hook('SessionStarted', 1, function() {
    $sessionId = session_id();
    logActivity("Session started: $sessionId");
});

// Log session destruction
add_hook('SessionEnded', 1, function() {
    logActivity("Session ended");
});
```

### Step 11: Test High Availability
```bash
# If using Redis Sentinel or Cluster
# Configure multiple Redis endpoints

php
// In configuration.php
$redis_seeds = [
    'redis1:6379',
    'redis2:6379', 
    'redis3:6379'
];

ini_set('session.save_path', 
    'tcp://' . implode(',tcp://', $redis_seeds) . 
    '?auth=your_redis_password&database=1&prefix=whmcs_sess_&timeout=2&read_timeout=2'
);
```

## Troubleshooting

### Sessions Not Saving
```bash
# Check Redis connection
redis-cli -h redis_host -p 6379 -a your_redis_password ping

# Check PHP error logs
tail -f /var/log/php*-fpm.log
tail -f /var/log/nginx/error.log

# Verify Redis extension loaded
php -m | grep redis
```

### Performance Issues
```bash
# Check for blocked keys
redis-cli -h redis_host -p 6379 -a your_redis_password --latency

# Monitor command stats
redis-cli -h redis_host -p 6379 -a your_redis_password INFO commandstats
```

## Security Considerations
- Use strong Redis password
- Restrict Redis access via firewall
- Enable TLS for remote connections
- Use Unix sockets for local connections
- Regular security audits

## Tags
- redis
- sessions
- performance
- scalability