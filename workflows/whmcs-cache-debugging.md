# WHMCS Cache Debug Workflow

## Overview
This workflow guides you through debugging cache issues.

## Prerequisites
- Cache configuration
- Cache driver access

## Step-by-Step Guide

### Step 1: Check Cache Configuration
```php
// Check cache driver
$driver = \Config::get('cache.default');
echo "Cache driver: $driver";

// Check cache connection
$store = \Cache::getStore();
echo "Cache store: " . get_class($store);
```

### Step 2: Debug Cache Operations
```php
// Add to your cache code
public function getCachedData(string $key)
{
    $cached = \Cache::get($key);
    
    if ($cached !== null) {
        $this->log->debug("Cache hit", ['key' => $key]);
        return $cached;
    }
    
    $this->log->debug("Cache miss", ['key' => $key]);
    
    $data = $this->computeData();
    \Cache::put($key, $data, 3600);
    
    return $data;
}
```

### Step 3: Test Cache Directly
```bash
# Clear specific cache
php artisan cache:clear --tags=yourmodule

# View cache
php artisan tinker
>>> Cache::get('your_key');

# Check file cache
ls -la /var/www/whmcs/storage/framework/cache/data/
```

### Step 4: Common Cache Issues
```php
// Issue: Cache not updating
// Fix: Use Cache::forget() before Cache::put()

// Issue: Cache not persisting
// Fix: Check cache directory permissions

// Issue: Stale data
// Fix: Implement cache tags and proper expiration
```

## Cache Debug Checklist

### Investigation
- [ ] Cache driver checked
- [ ] Cache operations logged
- [ ] Cache storage examined
- [ ] TTL verified

### Resolution
- [ ] Cache cleared
- [ ] Permissions fixed
- [ ] TTL adjusted
- [ ] Tags implemented
