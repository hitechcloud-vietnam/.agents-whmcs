# WHMCS Memory Cache Setup Workflow

## Description
Configure memory caching systems (Redis/Memcached) for WHMCS to improve performance.

## Prerequisites
- Redis or Memcached server
- PHP extension (php-redis or php-memcached)
- SSH access to WHMCS server

## Caching Options
| Backend | Pros | Cons |
|---------|------|------|
| Redis | Rich data types, persistence, clustering | Slightly more RAM |
| Memcached | Simpler, distributed cache | No persistence, string-only |
| APCu | No extra server needed | Local only, not shared |

## Steps

### Step 1: Install Redis Server
```bash
# Install Redis
apt update && apt install -y redis-server

# Configure Redis
cat > /etc/redis/redis.conf << 'EOF'
bind 127.0.0.1
port 6379
protected-mode yes
daemonize yes
pidfile /var/run/redis/redis.pid
loglevel notice
logfile /var/log/redis/redis-server.log

# Memory settings
maxmemory 512mb
maxmemory-policy allkeys-lru

# Persistence
save 900 1
save 300 10
save 60 10000
stop-writes-on-bgsave-error yes
rdbcompression yes

# AOF
appendonly yes
appendfilename "appendonly.aof"
appendfsync everysec
EOF

# Start Redis
systemctl restart redis-server
systemctl enable redis-server

# Test connection
redis-cli ping
# Should return: PONG
```

### Step 2: Install PHP Redis Extension
```bash
# Install PHP Redis extension
apt install -y php-redis

# Or for specific PHP version
apt install -y php8.2-redis

# Verify installation
php -m | grep redis
```

### Step 3: Configure WHMCS for Redis Cache
```php
// Add to configuration.php

// Redis cache configuration
$redis_host = '127.0.0.1';
$redis_port = 6379;
$redis_password = '';  // Set password if configured
$redis_database = 0;
$redis_prefix = 'whmcs_';

// WHMCS 8+ - Use Laravel cache config
// Create or modify bootstrap/cache.php
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

### Step 4: Alternative: Configure Memcached
```bash
# Install Memcached
apt install -y memcached libmemcached-tools

# Configure
cat > /etc/memcached.conf << 'EOF'
# Memory
-d mlockall
# 512MB cache
-m 512
-p /var/run/memcached/memcached.pid
-u memcache
-l 127.0.0.1
-p 11211
-c 1024

# Disable large requests (security)
# -t 4
EOF

systemctl restart memcached

# Install PHP extension
apt install -y php-memcached
systemctl restart php8.2-fpm

# Test
php -m | grep memcached
```

### Step 5: Configure WHMCS for Memcached
```php
// Add to configuration.php

// Memcached configuration
$memcached_servers = [
    [
        '127.0.0.1',  // Memcached server
        11211          // Port
    ]
];
```

### Step 6: Configure Laravel Cache (WHMCS 8+)
```php
// bootstrap/cache.php

return [
    'default' => env('CACHE_DRIVER', 'redis'),
    
    'stores' => [
        'array' => [
            'driver' => 'array',
            'serialize' => false,
        ],
        
        'file' => [
            'driver' => 'file',
            'path' => storage_path('cache/data'),
            'lock_path' => storage_path('cache/data'),
        ],
        
        'redis' => [
            'driver' => 'redis',
            'connection' => 'cache',
            'lock_connection' => 'default',
        ],
        
        'memcached' => [
            'driver' => 'memcached',
            'persistent_id' => 'whmcs',
            'servers' => [
                [
                    'host' => '127.0.0.1',
                    'port' => 11211,
                    'weight' => 100,
                ],
            ],
        ],
    ],
    
    'prefix' => env('CACHE_PREFIX', 'whmcs_cache_'),
];
```

### Step 7: Create Custom Cache Service
```php
<?php
// includes/clicodes/CacheService.php

namespace WHMCS\CLICodes;

class CacheService
{
    private $redis;
    private $prefix = 'whmcs_custom_';
    
    public function __construct()
    {
        if (!class_exists('Redis')) {
            throw new Exception('Redis extension not loaded');
        }
        
        $this->redis = new \Redis();
        $this->redis->connect(
            \DI::make('config')->get('redis_host', '127.0.0.1'),
            \DI::make('config')->get('redis_port', 6379)
        );
    }
    
    public function get($key, $default = null)
    {
        $value = $this->redis->get($this->prefix . $key);
        return $value !== false ? unserialize($value) : $default;
    }
    
    public function set($key, $value, $ttl = 3600)
    {
        $key = $this->prefix . $key;
        $value = serialize($value);
        
        if ($ttl > 0) {
            $this->redis->setex($key, $ttl, $value);
        } else {
            $this->redis->set($key, $value);
        }
        
        return $this;
    }
    
    public function delete($key)
    {
        $this->redis->del($this->prefix . $key);
        return $this;
    }
    
    public function clear()
    {
        $keys = $this->redis->keys($this->prefix . '*');
        if ($keys) {
            $this->redis->del($keys);
        }
        return $this;
    }
    
    public function remember($key, $ttl, callable $callback)
    {
        $value = $this->get($key);
        
        if ($value === null) {
            $value = $callback();
            $this->set($key, $value, $ttl);
        }
        
        return $value;
    }
}
```

### Step 8: Create Cache Hook
```php
<?php
// includes/hooks/cache_hooks.php

// Cache warmup on cron
add_hook('AfterCronJob', 1, function() {
    $cache = \App::cache();
    
    // Warmup frequently accessed data
    $cache->remember('product_categories', 3600, function() {
        return \WHMCS\Product\Category::all()->toArray();
    });
    
    $cache->remember('active_addons', 3600, function() {
        return \WHMCS\Addons\Addon::active()->get()->toArray();
    });
});

// Cache invalidation on data change
add_hook('AfterProductCategoryUpdate', 1, function($vars) {
    \App::cache()->forget('product_categories');
});

add_hook('AfterProductUpdate', 1, function($vars) {
    \App::cache()->forget('product_categories');
    \App::cache()->forget('products_list');
});
```

### Step 9: Configure Application Cache
```php
// Add to configuration.php

// Custom cache configuration
$whmcs_cache_config = [
    'driver' => 'redis',
    'prefix' => 'whmcs_app_',
    'ttl' => [
        'default' => 3600,
        'products' => 7200,
        'categories' => 7200,
        'config' => 86400,
    ],
];

// Environment-based caching
if (getenv('WHMCS_ENV') === 'production') {
    ini_set('opcache.enable', '1');
    ini_set('opcache.validate_timestamps', '0');
}
```

### Step 10: Monitor Cache Performance
```bash
# Create monitoring script
cat > /usr/local/bin/cache-monitor.sh << 'EOF'
#!/bin/bash

ALERT_EMAIL="admin@example.com"
REDIS_HOST="127.0.0.1"
REDIS_PORT="6379"

# Check Redis
if redis-cli -h $REDIS_HOST -p $REDIS_PORT ping > /dev/null 2>&1; then
    INFO=$(redis-cli -h $REDIS_HOST -p $REDIS_PORT INFO)
    
    MEMORY=$(echo "$INFO" | grep "used_memory_human" | cut -d: -f2 | tr -d '\r')
    KEYS=$(redis-cli -h $REDIS_HOST -p $REDIS_PORT DBSIZE)
    HIT_RATE=$(echo "$INFO" | grep "keyspace_hits" | awk -F'[/:]' '{hits=$2; miss=$3} END {print (hits/(hits+miss))*100}')
    
    echo "Redis Memory: $MEMORY"
    echo "Keys: $KEYS"
    echo "Hit Rate: ${HIT_RATE}%"
    
    # Alert on low hit rate
    if (( $(echo "$HIT_RATE < 80" | bc -l) )); then
        echo "Low cache hit rate: $HIT_RATE%" | \
            mail -s "Cache Alert" $ALERT_EMAIL
    fi
else
    echo "Redis connection failed" | mail -s "Cache Alert: DOWN" $ALERT_EMAIL
fi
EOF

chmod +x /usr/local/bin/cache-monitor.sh
echo "*/5 * * * * /usr/local/bin/cache-monitor.sh" >> /etc/crontab
```

### Step 11: Test Cache Configuration
```php
<?php
// test-cache.php
require_once __DIR__ . '/init.php';

use WHMCS\Cache\CacheManager;

echo "=== Cache Configuration Test ===\n\n";

// Test Redis
try {
    $redis = new Redis();
    $redis->connect('127.0.0.1', 6379);
    $redis->set('test_key', 'test_value');
    $value = $redis->get('test_key');
    
    if ($value === 'test_value') {
        echo "Redis: WORKING\n";
    }
} catch (Exception $e) {
    echo "Redis: FAILED - " . $e->getMessage() . "\n";
}

// Test WHMCS cache
try {
    $cache = App::cache();
    $cache->put('test_item', ['data' => 'test'], 60);
    $item = $cache->get('test_item');
    
    if ($item && $item['data'] === 'test') {
        echo "WHMCS Cache: WORKING\n";
    }
} catch (Exception $e) {
    echo "WHMCS Cache: FAILED - " . $e->getMessage() . "\n";
}

echo "\nCache drivers available:\n";
print_r(CacheManager::stores());
```

## APCu (Alternative - No Extra Server)
```bash
# Install APCu
apt install -y php-apcu

# Restart PHP
systemctl restart php8.2-fpm

# Configure
cat >> /etc/php/8.2/fpm/conf.d/99-apcu.ini << 'EOF'
apc.enabled=1
apc.shm_size=128M
apc.ttl=3600
apc.gc_ttl=600
apc.enable_cli=0
apc.include_once_override=0
EOF

systemctl restart php8.2-fpm

# Test
php -r "var_dump(apc_enabled());"
```

## Performance Comparison
| Backend | Read Speed | Write Speed | Memory Use | Persistence |
|---------|------------|-------------|------------|--------------|
| Redis | Fast | Fast | Medium | Yes |
| Memcached | Fast | Fast | Low | No |
| APCu | Very Fast | Very Fast | Low | No |
| File | Slow | Slow | N/A | Yes |

## Tags
- cache
- redis
- memcached
- apcu
- performance
- memory