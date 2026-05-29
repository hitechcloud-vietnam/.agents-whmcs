# WHMCS Custom PHP Settings

## Overview

WHMCS allows extensive customization of PHP settings to optimize performance, enhance security, and meet specific hosting requirements. This guide covers the configuration options available for customizing PHP behavior within the WHMCS environment.

Custom PHP settings can be applied at multiple levels: system-wide (php.ini), per-directory (.htaccess), or within WHMCS hooks using `ini_set()` for runtime modifications.

## Technical Details

### Configuration File Locations

| File | Purpose | Notes |
|------|---------|-------|
| `php.ini` | System PHP configuration | Affects entire server |
| `.htaccess` | Per-directory PHP settings | Apache only |
| `.user.ini` | Per-directory PHP settings | Alternative to .htaccess |
| WHMCS hooks | Runtime PHP modifications | Application-level |

### Key PHP Directives for WHMCS

```php
; Memory and Performance
memory_limit = 256M
max_execution_time = 300
max_input_time = 300
upload_max_filesize = 64M
post_max_size = 64M

; Session Configuration
session.gc_maxlifetime = 86400
session.cookie_httponly = 1
session.cookie_secure = 1

; Error Handling
display_errors = Off
log_errors = On
error_log = /var/log/whmcs_php_errors.log
```

## Configuration Options

### 1. Memory and Execution Limits

```php
// In php.ini or .htaccess
php_value memory_limit 256M
php_value max_execution_time 300
php_value max_input_time 300

// In .user.ini
memory_limit=256M
max_execution_time=300
```

### 2. File Upload Settings

```php
// For invoice attachments, support tickets, etc.
php_value upload_max_filesize 64M
php_value post_max_size 64M
php_value max_file_uploads 20
```

### 3. Session Configuration

```php
// Optimal for WHMCS session handling
php_value session.gc_maxlifetime 86400
php_value session.cookie_lifetime 0
php_value session.cookie_httponly 1
```

### 4. Timezone Settings

```php
// Prevent timezone warnings
php_value date.timezone "America/New_York"
```

## Code Examples

### Runtime PHP Configuration via Hook

```php
<?php
// hooks/custom_php_settings.php

add_hook('AdminAreaPage', 1, function($vars) {
    // Set custom memory limit for admin operations
    if (ini_get('memory_limit') < 256 * 1024 * 1024) {
        ini_set('memory_limit', '256M');
    }

    // Increase execution time for bulk operations
    if (defined('BULK_OPERATION')) {
        ini_set('max_execution_time', 600);
    }
});

add_hook('ClientAreaPage', 1, function($vars) {
    // Optimize session handling for client area
    ini_set('session.gc_maxlifetime', 86400);
});
```

### Custom Error Handler for WHMCS

```php
<?php
// hooks/error_handler.php

add_hook('AdminAreaHeaderOutput', 1, function($vars) {
    // Display errors only in development
    if (getenv('WHMCS_ENV') === 'development') {
        ini_set('display_errors', 1);
        error_reporting(E_ALL);
    } else {
        ini_set('display_errors', 0);
        error_reporting(E_ALL & ~E_NOTICE & ~E_DEPRECATED);
    }
});
```

### Performance Optimization Hook

```php
<?php
// hooks/performance_settings.php

add_hook('PreCronJob', 1, function($vars) {
    // Optimize for background processing
    ini_set('memory_limit', '512M');
    ini_set('max_execution_time', 3600);
    ini_set('session.gc_maxlifetime', 7200);
});
```

## Troubleshooting Tips

### Common Issues and Solutions

| Issue | Cause | Solution |
|-------|-------|----------|
| Memory Exhausted | Large module operations | Increase `memory_limit` in php.ini |
| Timeout Errors | Long-running queries | Increase `max_execution_time` |
| Session Expires Quickly | Low GC lifetime | Set `session.gc_maxlifetime` to 86400+ |
| File Upload Fails | Size limit exceeded | Increase `upload_max_filesize` and `post_max_size` |
| Timezone Warnings | Missing timezone config | Set `date.timezone` in php.ini |

### Debugging PHP Configuration

```php
<?php
// Debug endpoint - access via secure URL
add_hook('AdminAreaHeaderOutput', 1, function() {
    if (isset($_GET['debug_php'])) {
        echo '<pre>';
        echo 'memory_limit: ' . ini_get('memory_limit') . "\n";
        echo 'max_execution_time: ' . ini_get('max_execution_time') . "\n";
        echo 'session.gc_maxlifetime: ' . ini_get('session.gc_maxlifetime') . "\n";
        echo 'upload_max_filesize: ' . ini_get('upload_max_filesize') . "\n";
        echo 'date.timezone: ' . date_default_timezone_get() . "\n";
        echo '</pre>';
    }
});
```

### Performance Monitoring

```php
<?php
// Log PHP performance metrics
add_hook('AdminAreaFooterOutput', 1, function($vars) {
    $memoryUsage = memory_get_usage(true) / 1024 / 1024;
    $peakMemory = memory_get_peak_usage(true) / 1024 / 1024;

    logActivity("Memory: {$memoryUsage}MB | Peak: {$peakMemory}MB");
});
```

### Recommended Production Settings

```ini
; WHMCS Production php.ini settings
memory_limit = 512M
max_execution_time = 300
max_input_time = 300
upload_max_filesize = 64M
post_max_size = 64M

session.gc_maxlifetime = 86400
session.cookie_lifetime = 0
session.cookie_httponly = 1
session.cookie_secure = 1

display_errors = Off
log_errors = On
error_log = /var/log/whmcs_php_errors.log

date.timezone = "UTC"
```