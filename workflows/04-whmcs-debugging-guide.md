# WHMCS Debugging Guide

## Overview
This workflow provides systematic approaches to debugging WHMCS modules, customizations, and integrations.

## Debug Mode Setup

### Enable WHMCS Debug Logging

```php
// In your config.php
$debug = true;
$display_errors = true;

// Enable query logging
$db_debug = true;
```

### Enable Module Debug Logging

```php
<?php
// src/Helper/Logger.php

namespace WHMCS\Module\Addon\YourModule\Helper;

class Logger
{
    private static $enabled = false;
    private static $logPath;

    public static function enable(): void
    {
        self::$enabled = true;
        self::$logPath = dirname(__DIR__, 3) . '/storage/logs/module_debug.log';
    }

    public static function debug(string $message, array $context = []): void
    {
        if (!self::$enabled) {
            return;
        }

        $timestamp = date('Y-m-d H:i:s');
        $contextStr = !empty($context) ? ' | ' . json_encode($context) : '';
        $logLine = "[{$timestamp}] DEBUG: {$message}{$contextStr}\n";

        file_put_contents(self::$logPath, $logLine, FILE_APPEND);
    }

    public static function error(string $message, array $context = []): void
    {
        $timestamp = date('Y-m-d H:i:s');
        $contextStr = !empty($context) ? ' | ' . json_encode($context) : '';
        $logLine = "[{$timestamp}] ERROR: {$message}{$contextStr}\n";

        $logPath = dirname(__DIR__, 3) . '/storage/logs/module_error.log';
        file_put_contents($logPath, $logLine, FILE_APPEND);
    }
}
```

## Step 1: PHP Error Logging

### Configure PHP Error Handler

```php
<?php
// Debug bootstrap
error_reporting(E_ALL);
ini_set('display_errors', 1);
ini_set('log_errors', 1);
ini_set('error_log', '/var/log/whmcs/php_errors.log');
```

### WHMCS Error Handler

```php
<?php
// Add to includes/init.php or a custom bootstrap

// Custom error handler
set_error_handler(function($severity, $message, $file, $line) {
    throw new ErrorException($message, 0, $severity, $file, $line);
});

// Custom exception handler
set_exception_handler(function($e) {
    $logPath = dirname(__DIR__) . '/storage/logs/exception.log';
    $log = sprintf(
        "[%s] %s in %s:%d\n%s\n\n",
        date('Y-m-d H:i:s'),
        $e->getMessage(),
        $e->getFile(),
        $e->getLine(),
        $e->getTraceAsString()
    );
    file_put_contents($logPath, $log, FILE_APPEND);
    echo "<pre>Error: " . htmlspecialchars($e->getMessage()) . "\n";
    echo "File: " . $e->getFile() . ":" . $e->getLine() . "\n";
    echo "<a href='?debug=1'>Show Trace</a></pre>";

    if (isset($_GET['debug'])) {
        echo "<pre>" . $e->getTraceAsString() . "</pre>";
    }
});
```

## Step 2: Database Debugging

### Enable Query Logging

```php
<?php
// Enable in config.php or dynamically
use WHMCS\Database\Capsule;

Capsule::connection()->enableQueryLog();
Capsule::connection()->getQueryLog();

// Later, retrieve logs
$queries = Capsule::connection()->getQueryLog();
foreach ($queries as $query) {
    echo $query['query'] . " (" . implode(', ', $query['bindings']) . ")\n";
}
```

### Query Performance Analysis

```php
<?php
// Debug slow queries
use WHMCS\Database\Capsule;

Capsule::connection()->enableQueryLog();

$start = microtime(true);

// Your query
$result = Capsule::table('tblclients')
    ->join('tblhosting', 'tblclients.id', '=', 'tblhosting.userid')
    ->where('tblclients.status', 'Active')
    ->get();

$elapsed = microtime(true) - $start;

if ($elapsed > 0.1) {
    $queries = Capsule::connection()->getQueryLog();
    error_log("Slow query detected: " . json_encode($queries));
}
```

## Step 3: Hook Debugging

### Debug Hook Execution

```php
<?php
// Add to configuration or bootstrap

// Log hook calls
add_hook('AfterModuleChangePassword', 1, function($params) {
    $logPath = dirname(__DIR__) . '/storage/logs/hook_debug.log';
    $log = sprintf(
        "[%s] Hook: AfterModuleChangePassword\nParams: %s\n",
        date('Y-m-d H:i:s'),
        json_encode($params)
    );
    file_put_contents($logPath, $log, FILE_APPEND);

    return $params;
});
```

### List All Registered Hooks

```php
<?php
// Debug script to list hooks
use WHMCS\Module\Addon\YourModule\Helper\Logger;

Logger::enable();

add_hook('ClientAreaPage', 1, function($vars) {
    Logger::debug('ClientAreaPage hook fired', [
        'page' => $vars['templatefile'] ?? 'unknown',
        'user_id' => $_SESSION['uid'] ?? null
    ]);
    return $vars;
});

// Access hooks registry
$hooks = Hook::all();
foreach ($hooks as $hook) {
    echo $hook->getHookPoint() . " - Priority: " . $hook->getPriority() . "\n";
}
```

## Step 4: API Debugging

### Log API Requests/Responses

```php
<?php
// src/Helpers/ApiDebugger.php

namespace WHMCS\Module\Addon\YourModule\Helper;

class ApiDebugger
{
    private static $logPath;

    public static function init(): void
    {
        self::$logPath = dirname(__DIR__, 3) . '/storage/logs/api_debug.log';
    }

    public static function logRequest(string $endpoint, array $params): void
    {
        $log = sprintf(
            "[%s] REQUEST: %s\nParams: %s\n",
            date('Y-m-d H:i:s'),
            $endpoint,
            json_encode($params, JSON_PRETTY_PRINT)
        );
        file_put_contents(self::$logPath, $log, FILE_APPEND);
    }

    public static function logResponse(string $endpoint, $response, ?string $error = null): void
    {
        $log = sprintf(
            "[%s] RESPONSE: %s\nStatus: %s\nBody: %s\nError: %s\n",
            date('Y-m-d H:i:s'),
            $endpoint,
            $error ? 'ERROR' : 'SUCCESS',
            is_string($response) ? $response : json_encode($response),
            $error ?? 'None'
        );
        file_put_contents(self::$logPath, $log, FILE_APPEND);
    }
}
```

### Use in API Client

```php
<?php
// src/Service/ApiClient.php

namespace WHMCS\Module\Addon\YourModule\Service;

use WHMCS\Module\Addon\YourModule\Helper\ApiDebugger;

class ApiClient
{
    private $baseUrl;
    private $apiKey;

    public function __construct(string $baseUrl, string $apiKey)
    {
        $this->baseUrl = $baseUrl;
        $this->apiKey = $apiKey;
        ApiDebugger::init();
    }

    public function request(string $endpoint, array $params = []): array
    {
        ApiDebugger::logRequest($endpoint, $params);

        try {
            $ch = curl_init($this->baseUrl . $endpoint);
            curl_setopt_array($ch, [
                CURLOPT_POST => true,
                CURLOPT_POSTFIELDS => json_encode($params),
                CURLOPT_HTTPHEADER => [
                    'Authorization: Bearer ' . $this->apiKey,
                    'Content-Type: application/json'
                ],
                CURLOPT_RETURNTRANSFER => true,
                CURLOPT_TIMEOUT => 30
            ]);

            $response = curl_exec($ch);
            $httpCode = curl_getinfo($ch, CURLINFO_HTTP_CODE);
            $error = curl_error($ch);
            curl_close($ch);

            if ($error) {
                ApiDebugger::logResponse($endpoint, null, $error);
                throw new \Exception("API Error: " . $error);
            }

            $data = json_decode($response, true);
            ApiDebugger::logResponse($endpoint, $data);

            return ['success' => $httpCode < 400, 'data' => $data];

        } catch (\Exception $e) {
            ApiDebugger::logResponse($endpoint, null, $e->getMessage());
            throw $e;
        }
    }
}
```

## Step 5: Xdebug Configuration

### Install Xdebug

```bash
# Ubuntu/Debian
sudo apt install php-xdebug

# Or via PECL
pecl install xdebug
```

### Configure Xdebug

```ini
; /etc/php/8.1/mods-available/xdebug.ini
zend_extension=xdebug.so

xdebug.mode=debug,develop
xdebug.client_host=127.0.0.1
xdebug.client_port=9003
xdebug.start_with_request=trigger
xdebug.idekey=PHPSTORM

; Or enable for all requests in development
xdebug.start_with_request=yes

; Better var output
xdebug.var_display_max_depth=5
xdebug.var_display_max_data=1024
xdebug.var_display_max_children=256
```

### VS Code Configuration

```json
{
    "version": "0.2.0",
    "configurations": [
        {
            "name": "Listen for Xdebug",
            "type": "php",
            "request": "launch",
            "port": 9003,
            "pathMappings": {
                "/var/www/whmcs": "${workspaceFolder}"
            },
            "xdebugSettings": {
                "max_children": 256,
                "max_data": 1024,
                "max_depth": 5
            }
        }
    ]
}
```

## Step 6: WHMCS-Specific Debugging

### Debug Module Activation

```php
<?php
// Add to beginning of your_module_activate()
function your_module_activate()
{
    $debug = true;
    $log = [];

    try {
        $log['start'] = date('Y-m-d H:i:s');
        $log['checking_db'] = true;

        // Check database connection
        $pdo = Capsule::connection()->getPdo();
        $log['db_connected'] = true;

        // Check permissions
        $log['tables_to_create'] = ['mod_your_module'];
        foreach ($log['tables_to_create'] as $table) {
            $exists = Capsule::connection()->getPdo()
                ->query("SHOW TABLES LIKE '{$table}'")->rowCount() > 0;
            $log["table_{$table}_exists"] = $exists;
        }

        // Create tables
        $query = "CREATE TABLE IF NOT EXISTS `mod_your_module` (
            `id` INT NOT NULL AUTO_INCREMENT,
            `data` TEXT,
            `created_at` DATETIME DEFAULT CURRENT_TIMESTAMP,
            PRIMARY KEY (`id`)
        ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4";

        full_query($query);
        $log['table_created'] = true;

        // ... rest of activation

        $log['success'] = true;

    } catch (\Exception $e) {
        $log['error'] = $e->getMessage();
        $log['trace'] = $e->getTraceAsString();
    }

    if ($debug) {
        error_log("Module Activation Debug: " . json_encode($log, JSON_PRETTY_PRINT));
    }

    // Return result
    return [
        'status' => ($log['success'] ?? false) ? 'success' : 'error',
        'description' => $log['error'] ?? 'Module activated'
    ];
}
```

### Debug Hook Parameters

```php
<?php
// Add this to any hook to inspect parameters
add_hook('ClientAreaPage', 1, function($vars) {
    $logFile = dirname(__DIR__) . '/storage/logs/hook_params.log';
    $log = [
        'hook' => 'ClientAreaPage',
        'timestamp' => date('Y-m-d H:i:s'),
        'session' => [
            'uid' => $_SESSION['uid'] ?? null,
            'adminid' => $_SESSION['adminid'] ?? null
        ],
        'params' => $vars,
        'server' => $_SERVER,
        'request' => $_REQUEST
    ];
    file_put_contents($logFile, json_encode($log, JSON_PRETTY_PRINT) . "\n\n", FILE_APPEND);
    return $vars;
});
```

## Step 7: Memory Profiling

```php
<?php
// Memory debugging utility
class MemoryProfiler
{
    private static $checkpoints = [];

    public static function checkpoint(string $name): void
    {
        self::$checkpoints[$name] = [
            'memory' => memory_get_usage(true),
            'peak' => memory_get_peak_usage(true),
            'time' => microtime(true)
        ];
    }

    public static function report(): array
    {
        $report = [];
        $prev = null;

        foreach (self::$checkpoints as $name => $data) {
            $report[$name] = [
                'memory' => self::formatBytes($data['memory']),
                'peak' => self::formatBytes($data['peak']),
                'time' => date('H:i:s', $data['time'])
            ];

            if ($prev) {
                $memDiff = $data['memory'] - self::$checkpoints[$prev]['memory'];
                $timeDiff = $data['time'] - self::$checkpoints[$prev]['time'];
                $report[$name]['memory_delta'] = self::formatBytes($memDiff);
                $report[$name]['time_delta'] = round($timeDiff * 1000) . 'ms';
            }
            $prev = $name;
        }

        return $report;
    }

    private static function formatBytes(int $bytes): string
    {
        $units = ['B', 'KB', 'MB', 'GB'];
        $i = 0;
        while ($bytes >= 1024 && $i < 3) {
            $bytes /= 1024;
            $i++;
        }
        return round($bytes, 2) . ' ' . $units[$i];
    }
}

// Usage
MemoryProfiler::checkpoint('start');
// ... your code ...
MemoryProfiler::checkpoint('after_query');
// ... more code ...
MemoryProfiler::checkpoint('end');

print_r(MemoryProfiler::report());
```

## Step 8: Browser Developer Tools

### Network Tab Debugging

```javascript
// Console script to capture WHMCS AJAX requests
(function() {
    const originalFetch = window.fetch;
    window.fetch = async function(...args) {
        console.group('Fetch: ' + (args[1]?.method || 'GET'));
        console.log('URL:', args[0]);
        console.log('Options:', args[1]);

        const response = await originalFetch.apply(this, args);
        const clone = response.clone();

        clone.text().then(body => {
            console.log('Response:', body.substring(0, 500));
        });

        console.groupEnd();
        return response;
    };
})();
```

### Local Storage Debugging

```javascript
// Debug WHMCS session data
console.log('WHMCS Config:', JSON.parse(localStorage.getItem('whmcsAdminConfig') || '{}'));
console.log('WHMCS User:', JSON.parse(localStorage.getItem('whmcsAdminUser') || '{}'));
```

## Verification Checklist

- [ ] Debug mode enabled in config.php
- [ ] Error logging configured
- [ ] Module debug logger implemented
- [ ] Database query logging enabled
- [ ] Hook debugging active
- [ ] API request/response logging configured
- [ ] Xdebug installed and configured
- [ ] Browser dev tools ready
- [ ] Memory profiling utility available
- [ ] Debug log rotation configured

## Common Issues & Solutions

### Module Not Loading
- Check file permissions: 644 for files, 755 for directories
- Verify module filename matches directory name
- Check PHP error log for fatal errors
- Clear WHMCS cache: `php artisan cache:clear`

### Hook Not Firing
- Verify hook is registered: Check `tblhooks` table
- Enable hook debugging and check logs
- Verify hook priority is correct (1-100)
- Check if hook point exists

### Database Connection Issues
- Verify credentials in config.php
- Check MySQL server is running
- Test connection with: `mysql -u user -p -h host`
- Check MySQL user has required permissions

### API Timeouts
- Increase PHP max_execution_time
- Check firewall rules
- Verify SSL certificate validity
- Enable verbose logging for cURL
