# WHMCS Cache Management Workflow

## Overview
This workflow covers cache management strategies for WHMCS.

## Step 1: Cache Management Service

```php
<?php
// src/Service/CacheManagementService.php

namespace WHMCS\Module\Addon\YourModule\Service;

use WHMCS\Database\Capsule;

class CacheManagementService
{
    private $cacheDir;
    private $redis = null;

    public function __construct()
    {
        $this->cacheDir = dirname(__DIR__, 3) . '/storage/cache';

        if (class_exists('Redis')) {
            $this->redis = new \Redis();
            try {
                $this->redis->connect('127.0.0.1', 6379);
            } catch (\Exception $e) {
                $this->redis = null;
            }
        }
    }

    public function clearAllCache(): array
    {
        $cleared = [];

        // Clear file cache
        $cleared['file_cache'] = $this->clearFileCache();

        // Clear opcode cache
        $cleared['opcode_cache'] = $this->clearOpCache();

        // Clear database query cache
        $cleared['query_cache'] = $this->clearQueryCache();

        // Clear Redis cache if available
        if ($this->redis) {
            $cleared['redis_cache'] = $this->clearRedisCache();
        }

        return $cleared;
    }

    private function clearFileCache(): bool
    {
        if (!is_dir($this->cacheDir)) {
            return true;
        }

        $files = glob($this->cacheDir . '/*');
        foreach ($files as $file) {
            if (is_file($file)) {
                unlink($file);
            } elseif (is_dir($file)) {
                $this->deleteDirectory($file);
            }
        }

        return true;
    }

    private function clearOpCache(): bool
    {
        if (function_exists('opcache_reset')) {
            return opcache_reset();
        }
        return false;
    }

    private function clearQueryCache(): bool
    {
        try {
            Capsule::connection()->statement('RESET QUERY CACHE');
            return true;
        } catch (\Exception $e) {
            return false;
        }
    }

    private function clearRedisCache(): bool
    {
        if ($this->redis) {
            return $this->redis->flushDB();
        }
        return false;
    }

    public function getCacheStats(): array
    {
        $stats = [
            'file_cache' => $this->getFileCacheStats(),
            'opcache' => $this->getOpCacheStats(),
            'database_cache' => $this->getDatabaseCacheStats()
        ];

        if ($this->redis) {
            $stats['redis'] = $this->getRedisStats();
        }

        return $stats;
    }

    private function getFileCacheStats(): array
    {
        $files = glob($this->cacheDir . '/*');
        $totalSize = 0;

        foreach ($files as $file) {
            if (is_file($file)) {
                $totalSize += filesize($file);
            }
        }

        return [
            'file_count' => count($files),
            'total_size_bytes' => $totalSize,
            'total_size_mb' => round($totalSize / 1024 / 1024, 2)
        ];
    }

    private function getOpCacheStats(): array
    {
        if (!function_exists('opcache_get_status')) {
            return ['enabled' => false];
        }

        $status = opcache_get_status(false);

        return [
            'enabled' => true,
            'memory_usage' => $status['memory_usage'],
            'opcache_hit_rate' => $status['opcache_statistics']['opcache_hit_rate'] ?? 0
        ];
    }

    private function getDatabaseCacheStats(): array
    {
        try {
            $result = Capsule::connection()->select('SHOW STATUS LIKE "Qcache%"');
            $qcache = [];
            foreach ($result as $row) {
                $qcache[$row->Variable_name] = $row->Value;
            }

            return [
                'qcache_enabled' => isset($qcache['Qcache_enabled']) && $qcache['Qcache_enabled'] === 'ON',
                'qcache_hits' => $qcache['Qcache_hits'] ?? 0,
                'qcache_misses' => $qcache['Qcache_misses'] ?? 0,
                'qcache_size_mb' => round(($qcache['Qcache_size'] ?? 0) / 1024 / 1024, 2)
            ];
        } catch (\Exception $e) {
            return ['error' => $e->getMessage()];
        }
    }

    private function getRedisStats(): array
    {
        try {
            $info = $this->redis->info();

            return [
                'connected' => true,
                'used_memory_mb' => round($info['used_memory'] / 1024 / 1024, 2),
                'keys' => $this->redis->dbSize(),
                'hits' => $info['keyspace_hits'] ?? 0,
                'misses' => $info['keyspace_misses'] ?? 0
            ];
        } catch (\Exception $e) {
            return ['connected' => false, 'error' => $e->getMessage()];
        }
    }

    public function warmCache(): array
    {
        $warmed = [];

        // Warm client cache
        $warmed['clients'] = $this->warmClientCache();

        // Warm product cache
        $warmed['products'] = $this->warmProductCache();

        // Warm configuration cache
        $warmed['config'] = $this->warmConfigCache();

        return $warmed;
    }

    private function warmClientCache(): int
    {
        $count = 0;
        $clients = Capsule::table('tblclients')
            ->where('status', 'Active')
            ->limit(100)
            ->get();

        foreach ($clients as $client) {
            // Cache client data
            $key = "client_{$client->id}";
            if ($this->redis) {
                $this->redis->setex($key, 3600, json_encode($client));
            }
            $count++;
        }

        return $count;
    }

    private function warmProductCache(): int
    {
        $count = 0;
        $products = Capsule::table('tblproducts')
            ->where('status', 'Active')
            ->get();

        foreach ($products as $product) {
            $key = "product_{$product->id}";
            if ($this->redis) {
                $this->redis->setex($key, 3600, json_encode($product));
            }
            $count++;
        }

        return $count;
    }

    private function warmConfigCache(): bool
    {
        $config = Capsule::table('tblconfiguration')->get();

        if ($this->redis) {
            $this->redis->setex('config_all', 3600, json_encode($config));
        }

        return true;
    }

    private function deleteDirectory(string $dir): void
    {
        if (!is_dir($dir)) return;

        $files = array_diff(scandir($dir), ['.', '..']);
        foreach ($files as $file) {
            $path = "$dir/$file";
            is_dir($path) ? $this->deleteDirectory($path) : unlink($path);
        }
        rmdir($dir);
    }
}
```

## Verification Checklist

- [ ] Cache service implemented
- [ ] File cache clearing working
- [ ] Opcode cache clearing working
- [ ] Redis cache configured
- [ ] Cache warming working
- [ ] Cache statistics collecting
