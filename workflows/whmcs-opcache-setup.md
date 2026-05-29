# WHMCS OPcache Configuration Workflow

## Description
Configure and optimize OPcache for WHMCS to improve PHP performance through bytecode caching.

## Prerequisites
- PHP 5.5+ with OPcache extension
- SSH access
- Understanding of server memory

## What is OPcache?
OPcache improves PHP performance by storing precompiled script bytecode in shared memory, eliminating the need for PHP to load and parse scripts on each request.

## Steps

### Step 1: Verify OPcache Installation
```bash
# Check if OPcache is available
php -m | grep -i opcache

# Check OPcache version
php -r "echo phpversion('Zend OPcache');"

# Check OPcache configuration
php -i | grep -A 50 "Zend OPcache"
```

### Step 2: Configure OPcache
```bash
# Create OPcache configuration
cat > /etc/php/8.2/mods-available/opcache.ini << 'EOF'
; Enable OPcache
zend_extension=opcache

; Memory configuration
opcache.memory_consumption=256
opcache.interned_strings_buffer=32

; File cache configuration
; opcache.file_cache=/var/www/whmcs/cache/opcache

; Performance settings
opcache.max_accelerated_files=10000
opcache.revalidate_freq=0
opcache.validate_timestamps=0

; Optimization
opcache.fast_shutdown=1
opcache.enable_cli=0

; Preloading (PHP 7.4+)
; opcache.preload=/var/www/whmcs/preload.php
; opcache.preload_user=www-data

; Error handling
opcache.log_verbosity_level=1
opcache.protect_memory=0

; Restrict memory to single pool
opcache.restrict_api=

; Crash protection
opcache.enable_file_override=0
EOF

# Enable the module
phpenmod opcache

# Restart PHP-FPM
systemctl restart php8.2-fpm
```

### Step 3: Advanced OPcache Configuration
```bash
# For high-traffic WHMCS installations
cat > /etc/php/8.2/fpm/conf.d/99-opcache.ini << 'EOF'
; OPcache for WHMCS - Optimized for high traffic

zend_extension=opcache

; Memory - allocate based on total PHP memory needs
; WHMCS typical usage: 128-256MB per process
; Allocate 256-512MB total for OPcache
opcache.memory_consumption=512

; Interned strings - 16-32MB is usually sufficient
opcache.interned_strings_buffer=32

; Number of files to cache
; Should be greater than number of PHP files
opcache.max_accelerated_files=20000

; Cache validation
; Set to 0 in production (requires opcache_reset for updates)
opcache.validate_timestamps=0
opcache.revalidate_freq=0

; Optimization
opcache.fast_shutdown=1
opcache.enable_cli=0

; Compilation cache
opcache.opt_debug_level=0

; File-based caching (optional - for CLI caching)
; opcache.file_cache=/var/www/whmcs/cache/opcache
; opcache.file_cache_only=0
; opcache.file_cache_consistency_checks=1

; Memory protection (for debugging)
opcache.protect_memory=0

; Crash protection
opcache.enable_file_override=0

; Logging
opcache.log_verbosity_level=1

; Huge pages (advanced - requires system configuration)
; opcache.huge_code_pages=1
EOF

systemctl restart php8.2-fpm
```

### Step 4: Create OPcache Management Script
```php
<?php
// /var/www/whmcs/opcache-management.php
// Access via admin panel or CLI

// Prevent direct web access
if (php_sapi_name() !== 'cli') {
    die('CLI only');
}

// Check if OPcache is enabled
if (!function_exists('opcache_get_status')) {
    die("OPcache not available\n");
}

// Get current status
$status = opcache_get_status(false);

echo "=== OPcache Status ===\n";
echo "Enabled: " . ($status['opcache_enabled'] ? 'Yes' : 'No') . "\n";
echo "Memory Usage: " . round($status['memory_usage']['used_memory'] / 1024 / 1024, 2) . " MB\n";
echo "Memory Free: " . round($status['memory_usage']['free_memory'] / 1024 / 1024, 2) . " MB\n";
echo "Cached Files: " . $status['opcache_statistics']['num_cached_scripts'] . "\n";
echo "Hits: " . $status['opcache_statistics']['hits'] . "\n";
echo "Misses: " . $status['opcache_statistics']['misses'] . "\n";
echo "Hit Rate: " . round($status['opcache_statistics']['opcache_hit_rate'], 2) . "%\n";
echo "OOM Restarts: " . $status['opcache_statistics']['oom_restarts'] . "\n";
echo "\n";

// Commands
if (isset($argv[1])) {
    switch ($argv[1]) {
        case 'reset':
            opcache_reset();
            echo "OPcache cleared!\n";
            break;
        case 'status':
            print_r($status);
            break;
        case 'config':
            print_r(opcache_get_configuration());
            break;
        default:
            echo "Usage: php opcache-management.php [reset|status|config]\n";
    }
}
```

### Step 5: Create WHMCS Hook for Cache Management
```php
<?php
// includes/hooks/opcache_management.php

// Clear OPcache after WHMCS updates
add_hook('AfterCronJob', 1, function() {
    if (function_exists('opcache_get_status')) {
        $status = opcache_get_status(false);
        if ($status['memory_usage']['used_memory'] > ($status['memory_usage']['free_memory'] * 0.9)) {
            // Clear cache if over 90% usage
            opcache_reset();
            logActivity('OPcache cleared due to memory pressure');
        }
    }
});

// Clear OPcache after module updates
add_hook('AfterModuleUpdate', 1, function($vars) {
    opcache_reset();
    logActivity("OPcache cleared after updating module: {$vars['module']}");
});

// Clear OPcache after template changes
add_hook('AfterTemplateChange', 1, function($vars) {
    opcache_reset();
    logActivity("OPcache cleared after template change: {$vars['template']}");
});
```

### Step 6: Monitor OPcache Health
```bash
# Create monitoring script
cat > /usr/local/bin/opcache-monitor.sh << 'EOF'
#!/bin/bash

STATUS=$(php /var/www/whmcs/opcache-management.php status 2>/dev/null)
ALERT_EMAIL="admin@example.com"

# Extract metrics
MEMORY_USED=$(echo "$STATUS" | grep "Memory Usage" | awk '{print $3}' | tr -d ' MB')
HIT_RATE=$(echo "$STATUS" | grep "Hit Rate" | awk '{print $3}' | tr -d '%')

# Alert on low hit rate
if (( $(echo "$HIT_RATE < 80" | bc -l) )); then
    echo "OPcache hit rate is low: $HIT_RATE%" | \
        mail -s "OPcache Alert: Low Hit Rate" $ALERT_EMAIL
fi

# Alert on high memory usage
MEMORY_THRESHOLD=90
if (( $(echo "$MEMORY_USED > $MEMORY_THRESHOLD" | bc -l) )); then
    echo "OPcache memory at ${MEMORY_USED}MB" | \
        mail -s "OPcache Alert: High Memory" $ALERT_EMAIL
fi

# Log status
echo "$(date): Hit Rate=$HIT_RATE%, Memory=${MEMORY_USED}MB" >> /var/log/opcache.log
EOF

chmod +x /usr/local/bin/opcache-monitor.sh
echo "*/15 * * * * /usr/local/bin/opcache-monitor.sh" >> /etc/crontab
```

### Step 7: OPcache Performance Tuning
```bash
# For very large WHMCS installations
cat >> /etc/php/8.2/fpm/conf.d/99-opcache.ini << 'EOF'

; Preloading for maximum performance (PHP 7.4+)
; Preload all WHMCS files at startup
opcache.preload=/var/www/whmcs/includes/preload.php
opcache.preload_user=www-data

; JIT settings (PHP 8.0+)
opcache.jit_buffer_size=100M
opcache.jit=function
EOF

# Create preload script
cat > /var/www/whmcs/includes/preload.php << 'EOF'
<?php
/**
 * OPcache Preload Script
 * Loads all WHMCS files into OPcache at startup for maximum performance
 */

$whmcsRoot = __DIR__ . '/../';

function preloadDirectory($dir) {
    $iterator = new RecursiveIteratorIterator(
        new RecursiveDirectoryIterator($dir, RecursiveDirectoryIterator::SKIP_DOTS),
        RecursiveIteratorIterator::SELF_FIRST
    );
    
    $loaded = 0;
    foreach ($iterator as $file) {
        if ($file->isDir()) {
            continue;
        }
        
        $ext = $file->getExtension();
        if (in_array($ext, ['php'])) {
            $path = $file->getPathname();
            // Skip templates and cache
            if (strpos($path, '/templates_c/') === false &&
                strpos($path, '/cache/') === false &&
                strpos($path, '/vendor/') === false) {
                opcache_compile_file($path);
                $loaded++;
            }
        }
    }
    return $loaded;
}

// Preload core includes
if (is_dir($whmcsRoot . 'includes')) {
    $count = preloadDirectory($whmcsRoot . 'includes');
    echo "Preloaded $count files from includes/\n";
}
EOF
```

### Step 8: Verify OPcache is Working
```bash
# Check OPcache status
php /var/www/whmcs/opcache-management.php status

# Check hit rate via CLI
php -r "var_dump(opcache_get_status()['opcache_statistics']['opcache_hit_rate']);"

# Create test page
cat > /var/www/whmcs/test-opcache.php << 'EOF'
<?php
header('Content-Type: text/plain');
echo "OPcache Status:\n";
var_dump(opcache_get_status(true));
EOF

# Access: https://whmcs.example.com/test-opcache.php

# Clean up test file
rm /var/www/whmcs/test-opcache.php
```

### Step 9: Clear OPcache When Needed
```bash
# After updating WHMCS
php /var/www/whmcs/opcache-management.php reset

# Or via command line
php -r "opcache_reset();"

# After updating modules
# Clear via WHMCS admin: Utilities > System > Clear Cache

# Force clear via API
# Add to admin area or cron
```

## Troubleshooting

### OPcache Not Working
```bash
# Check if loaded
php -m | grep opcache

# Check for errors
php -r "var_dump(opcache_get_status());"

# Check log files
tail -f /var/log/php_errors.log
```

### High Memory Usage
```bash
# Increase memory or reduce cached files
# Check what's cached
php -r 'print_r(array_keys(opcache_get_status(true)["scripts"]));' | head -50
```

### Cache Invalidation Issues
```bash
# Always reset after code changes
opcache_reset();

// Or for specific file
opcache_invalidate('/path/to/file.php', true);
```

## Recommended Settings by Server Size

### Small Server (2GB RAM)
```
opcache.memory_consumption=128
opcache.interned_strings_buffer=16
opcache.max_accelerated_files=5000
```

### Medium Server (4GB RAM)
```
opcache.memory_consumption=256
opcache.interned_strings_buffer=32
opcache.max_accelerated_files=10000
```

### Large Server (8GB+ RAM)
```
opcache.memory_consumption=512
opcache.interned_strings_buffer=64
opcache.max_accelerated_files=20000
```

## Tags
- opcache
- performance
- php
- caching
- optimization