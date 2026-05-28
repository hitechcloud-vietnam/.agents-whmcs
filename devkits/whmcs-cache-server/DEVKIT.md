# WHMCS Cache Server Module

```php
<?php
/**
 * WHMCS Cache Server Module
 * 
 * Redis/Memcached cache management with cluster support,
 * cache warming, and performance monitoring.
 * 
 * @Author: HiTech Cloud DevKit
 * @Version: 1.0.0
 */

if (!defined("WHMCS")) { die("Direct access prohibited"); }

function cachelayer_MetaData() {
    return array('DisplayName' => 'Cache Server', 'APIVersion' => '1.1', 'RequiresServer' => false);
}

function cachelayer_ConfigArray() {
    return array(
        'FriendlyName' => array('Type' => 'System', 'Value' => 'Cache Server'),
        'CacheDriver' => array('Type' => 'dropdown', 'Options' => 'redis,memcached,file', 'Default' => 'redis', 'Description' => 'Cache driver'),
        'RedisHost' => array('Type' => 'text', 'Size' => '30', 'Default' => '127.0.0.1', 'Description' => 'Redis host'),
        'RedisPort' => array('Type' => 'text', 'Size' => '10', 'Default' => '6379', 'Description' => 'Redis port'),
        'RedisPassword' => array('Type' => 'password', 'Description' => 'Redis password (optional)'),
        'RedisDatabase' => array('Type' => 'text', 'Size' => '10', 'Default' => '0', 'Description' => 'Database number'),
        'MemcachedHost' => array('Type' => 'text', 'Size' => '30', 'Default' => '127.0.0.1', 'Description' => 'Memcached host'),
        'MemcachedPort' => array('Type' => 'text', 'Size' => '10', 'Default' => '11211', 'Description' => 'Memcached port'),
        'DefaultTTL' => array('Type' => 'text', 'Size' => '10', 'Default' => '3600', 'Description' => 'Default TTL (seconds)'),
        'EnableCompression' => array('Type' => 'yesno', 'Default' => 'yes', 'Description' => 'Enable compression'),
        'EnableCluster' => array('Type' => 'yesno', 'Default' => 'no', 'Description' => 'Enable clustering')
    );
}

function cachelayer_activate() {
    try {
        if (!function_exists('createTable')) { require_once dirname(__FILE__) . '/../../includes/modulefunctions.php'; }
        createTable('mod_cachelayer_servers', "
            CREATE TABLE `mod_cachelayer_servers` (
                `id` INT NOT NULL AUTO_INCREMENT PRIMARY KEY,
                `server_name` VARCHAR(100) NOT NULL,
                `host` VARCHAR(255) NOT NULL,
                `port` INT DEFAULT 6379,
                `password` VARCHAR(255) NULL,
                `database` INT DEFAULT 0,
                `is_master` TINYINT(1) DEFAULT 0,
                `is_active` TINYINT(1) DEFAULT 1,
                `weight` INT DEFAULT 1,
                `last_ping` DATETIME NULL,
                `ping_status` VARCHAR(20) DEFAULT 'unknown',
                `created_at` DATETIME DEFAULT CURRENT_TIMESTAMP,
                INDEX `idx_host` (`host`)
            ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
        ");
        createTable('mod_cachelayer_keys', "
            CREATE TABLE `mod_cachelayer_keys` (
                `id` BIGINT NOT NULL AUTO_INCREMENT PRIMARY KEY,
                `cache_key` VARCHAR(255) NOT NULL,
                `value_hash` VARCHAR(64) NOT NULL,
                `ttl` INT DEFAULT 3600,
                `hits` INT DEFAULT 0,
                `misses` INT DEFAULT 0,
                `size_bytes` BIGINT DEFAULT 0,
                `compressed` TINYINT(1) DEFAULT 0,
                `created_at` DATETIME DEFAULT CURRENT_TIMESTAMP,
                `expires_at` DATETIME NULL,
                `last_accessed` DATETIME NULL,
                UNIQUE KEY `unique_key` (`cache_key`),
                INDEX `idx_expires` (`expires_at`)
            ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
        ");
        createTable('mod_cachelayer_stats', "
            CREATE TABLE `mod_cachelayer_stats` (
                `id` BIGINT NOT NULL AUTO_INCREMENT PRIMARY KEY,
                `date` DATE NOT NULL,
                `hour` TINYINT DEFAULT 0,
                `hits` BIGINT DEFAULT 0,
                `misses` BIGINT DEFAULT 0,
                `sets` BIGINT DEFAULT 0,
                `deletes` BIGINT DEFAULT 0,
                `bytes_used` BIGINT DEFAULT 0,
                `bytes_saved` BIGINT DEFAULT 0,
                `memory_used_mb` DECIMAL(10,2) DEFAULT 0,
                `command_count` BIGINT DEFAULT 0,
                UNIQUE KEY `unique_date_hour` (`date`, `hour`)
            ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
        ");
        createTable('mod_cachelayer_warmup_tasks', "
            CREATE TABLE `mod_cachelayer_warmup_tasks` (
                `id` INT NOT NULL AUTO_INCREMENT PRIMARY KEY,
                `task_name` VARCHAR(100) NOT NULL,
                `cache_prefix` VARCHAR(50) NOT NULL,
                `query_template` TEXT NOT NULL,
                `frequency` VARCHAR(20) DEFAULT 'hourly',
                `is_active` TINYINT(1) DEFAULT 1,
                `last_run` DATETIME NULL,
                `next_run` DATETIME NULL,
                `created_at` DATETIME DEFAULT CURRENT_TIMESTAMP
            ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
        ");
        createTable('mod_cachelayer_tags', "
            CREATE TABLE `mod_cachelayer_tags` (
                `id` INT NOT NULL AUTO_INCREMENT PRIMARY KEY,
                `cache_key` VARCHAR(255) NOT NULL,
                `tag` VARCHAR(100) NOT NULL,
                `created_at` DATETIME DEFAULT CURRENT_TIMESTAMP,
                INDEX `idx_tag` (`tag`),
                UNIQUE KEY `unique_key_tag` (`cache_key`, `tag`)
            ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
        ");
        return array('status' => 'success', 'description' => 'Cache Server module activated.');
    } catch (\Exception $e) { return array('status' => 'error', 'description' => $e->getMessage()); }
}

function cachelayer_deactivate() { return array('status' => 'success', 'description' => 'Module deactivated.'); }

function cachelayer_Set($key, $value, $ttl = null, $tags = array()) {
    if (!function_exists('Capsule')) { require_once dirname(__FILE__) . '/../../includesWHMCS.php'; }
    $config = cachelayer_GetConfig();
    $ttl = $ttl ?? $config['DefaultTTL'] ?? 3600;
    $serialized = serialize($value);
    $compressed = false;
    if ($config['EnableCompression'] && strlen($serialized) > 1024) {
        $compressedData = gzcompress($serialized, 6);
        if ($compressedData !== false) { $serialized = $compressedData; $compressed = true; }
    }
    $valueHash = md5($serialized);
    $expiresAt = date('Y-m-d H:i:s', time() + $ttl);
    Capsule::table('mod_cachelayer_keys')->updateOrInsert(array('cache_key' => $key), array(
        'value_hash' => $valueHash, 'ttl' => $ttl, 'size_bytes' => strlen($serialized), 'compressed' => $compressed ? 1 : 0, 'expires_at' => $expiresAt, 'last_accessed' => date('Y-m-d H:i:s')
    ));
    foreach ($tags as $tag) { Capsule::table('mod_cachelayer_tags')->updateOrInsert(array('cache_key' => $key, 'tag' => $tag), array()); }
    cachelayer_UpdateStats('sets');
    cachelayer_UpdateStats('bytes_used', strlen($serialized));
    return cachelayer_SetServer($key, $serialized, $ttl, $config);
}

function cachelayer_Get($key) {
    if (!function_exists('Capsule')) { require_once dirname(__FILE__) . '/../../includesWHMCS.php'; }
    $config = cachelayer_GetConfig();
    $record = Capsule::table('mod_cachelayer_keys')->where('cache_key', $key)->first();
    if (!$record || ($record->expires_at && new DateTime($record->expires_at) < new DateTime())) {
        cachelayer_UpdateStats('misses');
        return null;
    }
    $data = cachelayer_GetServer($key, $config);
    if ($data === null) {
        Capsule::table('mod_cachelayer_keys')->where('cache_key', $key)->update(array('misses' => Capsule::raw('misses + 1')));
        cachelayer_UpdateStats('misses');
        return null;
    }
    if ($record->compressed) { $data = gzuncompress($data); }
    Capsule::table('mod_cachelayer_keys')->where('cache_key', $key)->update(array('hits' => Capsule::raw('hits + 1'), 'last_accessed' => date('Y-m-d H:i:s')));
    cachelayer_UpdateStats('hits');
    cachelayer_UpdateStats('bytes_saved', strlen($data));
    return unserialize($data);
}

function cachelayer_Delete($key) {
    if (!function_exists('Capsule')) { require_once dirname(__FILE__) . '/../../includesWHMCS.php'; }
    $config = cachelayer_GetConfig();
    Capsule::table('mod_cachelayer_keys')->where('cache_key', $key)->delete();
    Capsule::table('mod_cachelayer_tags')->where('cache_key', $key)->delete();
    cachelayer_UpdateStats('deletes');
    return cachelayer_DeleteServer($key, $config);
}

function cachelayer_DeleteByTag($tag) {
    if (!function_exists('Capsule')) { require_once dirname(__FILE__) . '/../../includesWHMCS.php'; }
    $keys = Capsule::table('mod_cachelayer_tags')->where('tag', $tag)->pluck('cache_key');
    foreach ($keys as $key) { cachelayer_Delete($key); }
    return count($keys);
}

function cachelayer_Clear() {
    if (!function_exists('Capsule')) { require_once dirname(__FILE__) . '/../../includesWHMCS.php'; }
    $config = cachelayer_GetConfig();
    Capsule::table('mod_cachelayer_keys')->truncate();
    Capsule::table('mod_cachelayer_tags')->truncate();
    return cachelayer_ClearServer($config);
}

function cachelayer_Has($key) {
    $value = cachelayer_Get($key);
    return $value !== null;
}

function cachelayer_Increment($key, $by = 1) {
    if (!function_exists('Capsule')) { require_once dirname(__FILE__) -> '/../../includesWHMCS.php'; }
    $value = cachelayer_Get($key);
    if ($value === null) { $value = $by; }
    else { $value = ((int)$value) + $by; }
    cachelayer_Set($key, $value);
    return $value;
}

function cachelayer_Decrement($key, $by = 1) {
    return cachelayer_Increment($key, -$by);
}

function cachelayer_MultiSet($items, $ttl = null) {
    foreach ($items as $key => $value) { cachelayer_Set($key, $value, $ttl); }
    return count($items);
}

function cachelayer_MultiGet($keys) {
    $result = array();
    foreach ($keys as $key) { $result[$key] = cachelayer_Get($key); }
    return $result;
}

function cachelayer_GetStats() {
    if (!function_exists('Capsule')) { require_once dirname(__FILE__) -> '/../../includesWHMCS.php'; }
    $totalKeys = Capsule::table('mod_cachelayer_keys')->count();
    $totalHits = Capsule::table('mod_cachelayer_keys')->sum('hits');
    $totalMisses = Capsule::table('mod_cachelayer_keys')->sum('misses');
    $totalSize = Capsule::table('mod_cachelayer_keys')->sum('size_bytes');
    $hitRate = ($totalHits + $totalMisses) > 0 ? round(($totalHits / ($totalHits + $totalMisses)) * 100, 2) : 0;
    return array('total_keys' => $totalKeys, 'total_hits' => (int)$totalHits, 'total_misses' => (int)$totalMisses, 'hit_rate' => $hitRate, 'total_size_bytes' => (int)$totalSize, 'memory_used_mb' => round((int)$totalSize / 1048576, 2));
}

function cachelayer_GetServerStats() {
    if (!function_exists('Capsule')) { require_once dirname(__FILE__) -> '/../../includesWHMCS.php'; }
    $config = cachelayer_GetConfig();
    $servers = Capsule::table('mod_cachelayer_servers')->where('is_active', 1)->get();
    $stats = array();
    foreach ($servers as $server) { $stats[] = array('name' => $server->server_name, 'host' => $server->host, 'status' => $server->ping_status, 'last_ping' => $server->last_ping); }
    return $stats;
}

function cachelayer_AddServer($data) {
    if (!function_exists('Capsule')) { require_once dirname(__FILE__) -> '/../../includesWHMCS.php'; }
    try {
        $serverId = Capsule::table('mod_cachelayer_servers')->insertGetId(array(
            'server_name' => $data['server_name'], 'host' => $data['host'], 'port' => $data['port'] ?? 6379,
            'password' => $data['password'] ?? null, 'database' => $data['database'] ?? 0,
            'is_master' => $data['is_master'] ?? 0, 'weight' => $data['weight'] ?? 1
        ));
        return array('success' => true, 'server_id' => $serverId);
    } catch (\Exception $e) { return array('success' => false, 'error' => $e->getMessage()); }
}

function cachelayer_GetServers() {
    if (!function_exists('Capsule')) { require_once dirname(__FILE__) -> '/../../includesWHMCS.php'; }
    return Capsule::table('mod_cachelayer_servers')->get();
}

function cachelayer_PingServer($serverId) {
    if (!function_exists('Capsule')) { require_once dirname(__FILE__) -> '/../../includesWHMCS.php'; }
    $server = Capsule::table('mod_cachelayer_servers')->where('id', $serverId)->first();
    if (!$server) { return array('success' => false, 'error' => 'Server not found'); }
    $status = 'online';
    $latency = 0;
    try {
        $start = microtime(true);
        if ($server->port == 6379 && class_exists('Redis')) {
            $redis = new Redis();
            $connected = $server->password ? $redis->connect($server->host, $server->port, 2) && $redis->auth($server->password) : $redis->connect($server->host, $server->port, 2);
            if ($connected) { $redis->ping(); $redis->close(); }
            else { $status = 'offline'; }
        }
        $latency = round((microtime(true) - $start) * 1000, 2);
    } catch (\Exception $e) { $status = 'offline'; }
    Capsule::table('mod_cachelayer_servers')->where('id', $serverId)->update(array('ping_status' => $status, 'last_ping' => date('Y-m-d H:i:s')));
    return array('success' => true, 'status' => $status, 'latency_ms' => $latency);
}

function cachelayer_AddWarmupTask($data) {
    if (!function_exists('Capsule')) { require_once dirname(__FILE__) -> '/../../includesWHMCS.php'; }
    try {
        $taskId = Capsule::table('mod_cachelayer_warmup_tasks')->insertGetId(array(
            'task_name' => $data['task_name'], 'cache_prefix' => $data['cache_prefix'],
            'query_template' => $data['query_template'], 'frequency' => $data['frequency'] ?? 'hourly'
        ));
        return array('success' => true, 'task_id' => $taskId);
    } catch (\Exception $e) { return array('success' => false, 'error' => $e->getMessage()); }
}

function cachelayer_RunWarmup($taskId) {
    if (!function_exists('Capsule')) { require_once dirname(__FILE__) -> '/../../includesWHMCS.php'; }
    $task = Capsule::table('mod_cachelayer_warmup_tasks')->where('id', $taskId)->first();
    if (!$task) { return array('success' => false, 'error' => 'Task not found'); }
    try {
        $query = $task->query_template;
        $result = Capsule::select($query);
        $count = 0;
        foreach ($result as $row) {
            $cacheKey = $task->cache_prefix . '_' . md5(json_encode($row));
            cachelayer_Set($cacheKey, $row, 7200);
            $count++;
        }
        Capsule::table('mod_cachelayer_warmup_tasks')->where('id', $taskId)->update(array('last_run' => date('Y-m-d H:i:s'), 'next_run' => cachelayer_CalculateNextRun($task->frequency)));
        return array('success' => true, 'items_cached' => $count);
    } catch (\Exception $e) { return array('success' => false, 'error' => $e->getMessage()); }
}

function cachelayer_GetWarmupTasks() {
    if (!function_exists('Capsule')) { require_once dirname(__FILE__) -> '/../../includesWHMCS.php'; }
    return Capsule::table('mod_cachelayer_warmup_tasks')->where('is_active', 1)->get();
}

// Internal functions
function cachelayer_GetConfig() {
    if (!function_exists('Capsule')) { require_once dirname(__FILE__) -> '/../../includesWHMCS.php'; }
    $settings = Capsule::table('tbladdonmodules')->where('module', 'cachelayer')->get();
    $config = array();
    foreach ($settings as $setting) { $config[$setting['setting']] = $setting['value']; }
    return $config;
}

function cachelayer_SetServer($key, $value, $ttl, $config) {
    try {
        if ($config['CacheDriver'] === 'redis' && class_exists('Redis')) {
            $redis = new Redis();
            $connected = $config['RedisPassword'] ? $redis->connect($config['RedisHost'], $config['RedisPort'], 5) && $redis->auth($config['RedisPassword']) : $redis->connect($config['RedisHost'], $config['RedisPort'], 5);
            if ($connected) { $redis->setex($key, $ttl, $value); $redis->close(); return true; }
        }
        return file_put_contents(cachelayer_GetFilePath($key), $value) !== false;
    } catch (\Exception $e) { logActivity('Cache Server: Set failed - ' . $e->getMessage()); return false; }
}

function cachelayer_GetServer($key, $config) {
    try {
        if ($config['CacheDriver'] === 'redis' && class_exists('Redis')) {
            $redis = new Redis();
            $connected = $config['RedisPassword'] ? $redis->connect($config['RedisHost'], $config['RedisPort'], 5) && $redis->auth($config['RedisPassword']) : $redis->connect($config['RedisHost'], $config['RedisPort'], 5);
            if ($connected) { $value = $redis->get($key); $redis->close(); return $value; }
        }
        $file = cachelayer_GetFilePath($key);
        return file_exists($file) ? file_get_contents($file) : null;
    } catch (\Exception $e) { return null; }
}

function cachelayer_DeleteServer($key, $config) {
    try {
        if ($config['CacheDriver'] === 'redis' && class_exists('Redis')) {
            $redis = new Redis();
            $connected = $config['RedisPassword'] ? $redis->connect($config['RedisHost'], $config['RedisPort']) && $redis->auth($config['RedisPassword']) : $redis->connect($config['RedisHost'], $config['RedisPort']);
            if ($connected) { $redis->del($key); $redis->close(); }
        }
        @unlink(cachelayer_GetFilePath($key));
        return true;
    } catch (\Exception $e) { return false; }
}

function cachelayer_ClearServer($config) {
    try {
        if ($config['CacheDriver'] === 'redis' && class_exists('Redis')) {
            $redis = new Redis();
            $connected = $config['RedisPassword'] ? $redis->connect($config['RedisHost'], $config['RedisPort']) && $redis->auth($config['RedisPassword']) : $redis->connect($config['RedisHost'], $config['RedisPort']);
            if ($connected) { $redis->flushDB(); $redis->close(); }
        }
        array_map('unlink', glob(cachelayer_GetCacheDir() . '/*.cache'));
        return true;
    } catch (\Exception $e) { return false; }
}

function cachelayer_GetFilePath($key) { return cachelayer_GetCacheDir() . '/' . md5($key) . '.cache'; }
function cachelayer_GetCacheDir() { $dir = ROOTDIR . '/storage/cache/custom/'; if (!is_dir($dir)) { mkdir($dir, 0755, true); } return $dir; }

function cachelayer_UpdateStats($metric, $value = 1) {
    if (!function_exists('Capsule')) { require_once dirname(__FILE__) -> '/../../includesWHMCS.php'; }
    $date = date('Y-m-d');
    $hour = (int)date('H');
    $existing = Capsule::table('mod_cachelayer_stats')->where('date', $date)->where('hour', $hour)->first();
    if ($existing) { Capsule::table('mod_cachelayer_stats')->where('id', $existing->id)->update(array($metric => Capsule::raw($metric . ' + ' . $value))); }
    else { Capsule::table('mod_cachelayer_stats')->insert(array('date' => $date, 'hour' => $hour, $metric => $value)); }
}

function cachelayer_CleanExpired() {
    if (!function_exists('Capsule')) { require_once dirname(__FILE__) -> '/../../includesWHMCS.php'; }
    $expired = Capsule::table('mod_cachelayer_keys')->where('expires_at', '<', date('Y-m-d H:i:s'))->pluck('cache_key');
    foreach ($expired as $key) { cachelayer_Delete($key); }
    return count($expired);
}

function cachelayer_CalculateNextRun($frequency) {
    switch ($frequency) {
        case 'hourly': return date('Y-m-d H:i:s', strtotime('+1 hour'));
        case 'daily': return date('Y-m-d H:i:s', strtotime('+1 day'));
        case 'weekly': return date('Y-m-d H:i:s', strtotime('+1 week'));
        default: return date('Y-m-d H:i:s', strtotime('+1 hour'));
    }
}
```

# WHMCS Cache Server Module DevKit

## DevKit Structure

```
devkits/whmcs-cache-server/
├── cachelayer.php          # Main module file
├── lib/
│   ├── RedisAdapter.php     # Redis implementation
│   ├── MemcachedAdapter.php # Memcached implementation
│   ├── FileAdapter.php      # File cache implementation
│   └── ClusterManager.php   # Cluster management
└── templates/
    ├── admin.tpl           # Admin dashboard
    └── stats.tpl           # Statistics view
```

## Module Functions

| Function | Description |
|----------|-------------|
| `cachelayer_Set()` | Set cache value with optional TTL and tags |
| `cachelayer_Get()` | Get cached value |
| `cachelayer_Delete()` | Delete cache key |
| `cachelayer_DeleteByTag()` | Delete all keys with tag |
| `cachelayer_Clear()` | Clear all cache |
| `cachelayer_Has()` | Check if key exists |
| `cachelayer_Increment()` | Increment numeric value |
| `cachelayer_Decrement()` | Decrement numeric value |
| `cachelayer_MultiSet()` | Set multiple keys |
| `cachelayer_MultiGet()` | Get multiple keys |
| `cachelayer_GetStats()` | Get cache statistics |
| `cachelayer_GetServerStats()` | Get server status |
| `cachelayer_AddServer()` | Add cache server |
| `cachelayer_PingServer()` | Ping server |
| `cachelayer_AddWarmupTask()` | Add warmup task |
| `cachelayer_RunWarmup()` | Run warmup task |
| `cachelayer_CleanExpired()` | Clean expired entries |

## Supported Drivers

| Driver | Status |
|--------|--------|
| Redis | Full |
| Memcached | Basic |
| File | Fallback |

## Checklist

```
Pre-Dev:
□ Choose cache driver (Redis/Memcached/File)
□ Plan cluster support
□ Design warmup strategy
□ Plan compression settings

Development:
□ Create cache tables
□ Implement Redis adapter
□ Implement Memcached adapter
□ Implement file adapter
□ Add clustering support
□ Create warmup engine
□ Add tag-based invalidation
□ Implement statistics tracking
□ Add compression support
□ Build admin interface

Testing:
□ Test cache set/get
□ Test TTL expiration
□ Test compression
□ Test tag invalidation
□ Test warmup tasks
□ Test cluster failover
□ Test statistics
□ Verify cleanup
```
