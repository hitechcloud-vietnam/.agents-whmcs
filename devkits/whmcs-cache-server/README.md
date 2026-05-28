# WHMCS Cache Server Module

Redis/Memcached cache management with clustering and performance monitoring.

## Features

- Multi-driver support (Redis, Memcached, File)
- Cache tagging and invalidation
- Cluster management
- Automatic warmup
- Statistics tracking
- Compression support
- TTL management

## Installation

1. Copy `cachelayer.php` to `/path/to/whmcs/modules/addons/cachelayer/`
2. Activate in WHMCS Admin > System Settings > Module Addons
3. Configure cache driver settings

## Usage

```php
// Set cache with TTL (default: 3600 seconds)
cachelayer_Set('user_123_profile', $userData, 7200);
cachelayer_Set('product_list', $products, 3600, array('products', 'catalog'));

// Get cache
$profile = cachelayer_Get('user_123_profile');
if ($profile === null) {
    // Cache miss - fetch from database
    $profile = fetchUserProfile(123);
    cachelayer_Set('user_123_profile', $profile);
}

// Check if key exists
if (cachelayer_Has('user_123_profile')) {
    // Key exists
}

// Delete single key
cachelayer_Delete('user_123_profile');

// Delete all keys with tag
cachelayer_DeleteByTag('products');

// Clear all cache
cachelayer_Clear();

// Increment/Decrement
cachelayer_Set('page_views_123', 0);
cachelayer_Increment('page_views_123');
cachelayer_Decrement('page_views_123', 5);

// Multi-set and multi-get
cachelayer_MultiSet(array(
    'key1' => $value1,
    'key2' => $value2,
    'key3' => $value3
), 3600);

$values = cachelayer_MultiGet(array('key1', 'key2', 'key3'));

// Get statistics
$stats = cachelayer_GetStats();
// Returns: total_keys, total_hits, total_misses, hit_rate, total_size_bytes, memory_used_mb

// Server management
cachelayer_AddServer(array(
    'server_name' => 'Redis Primary',
    'host' => '127.0.0.1',
    'port' => 6379,
    'password' => 'secret',
    'database' => 0,
    'is_master' => true,
    'weight' => 1
));

$servers = cachelayer_GetServers();

// Ping server
$result = cachelayer_PingServer($serverId);
// Returns: success, status, latency_ms

// Server statistics
$serverStats = cachelayer_GetServerStats();

// Warmup tasks
cachelayer_AddWarmupTask(array(
    'task_name' => 'Popular Products',
    'cache_prefix' => 'product',
    'query_template' => 'SELECT * FROM tblproducts WHERE popular = 1 LIMIT 100',
    'frequency' => 'hourly'
));

// Run warmup manually
$result = cachelayer_RunWarmup($taskId);
// Returns: success, items_cached

// Get warmup tasks
$tasks = cachelayer_GetWarmupTasks();

// Clean expired entries
$cleaned = cachelayer_CleanExpired();
```

## Configuration Options

| Option | Type | Default | Description |
|--------|------|---------|-------------|
| CacheDriver | dropdown | redis | Cache driver |
| RedisHost | text | 127.0.0.1 | Redis host |
| RedisPort | text | 6379 | Redis port |
| RedisPassword | password | - | Redis password |
| RedisDatabase | text | 0 | Database number |
| MemcachedHost | text | 127.0.0.1 | Memcached host |
| MemcachedPort | text | 11211 | Memcached port |
| DefaultTTL | text | 3600 | Default TTL (seconds) |
| EnableCompression | yesno | yes | Enable compression |
| EnableCluster | yesno | no | Enable clustering |

## Statistics

```php
$stats = cachelayer_GetStats();
// {
//     total_keys: 1523,
//     total_hits: 45892,
//     total_misses: 2341,
//     hit_rate: 95.14,
//     total_size_bytes: 15728640,
//     memory_used_mb: 15.0
// }
```

## Database Tables

- `mod_cachelayer_servers` - Cache server configurations
- `mod_cachelayer_keys` - Cache key metadata
- `mod_cachelayer_stats` - Hourly statistics
- `mod_cachelayer_warmup_tasks` - Warmup task definitions
- `mod_cachelayer_tags` - Cache key tags
