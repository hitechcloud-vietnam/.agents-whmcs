# WHMCS Module Debugging Workflow

## Description
Systematic debugging approach for WHMCS modules and customizations.

## Steps

### Step 1: Enable Debug Mode
```php
// Add to configuration.php
$display_errors = true;
ini_set('display_errors', '1');
error_reporting(E_ALL);

// Enable WHMCS debug logging
define('WHMCS_DEBUG_LOG', true);
define('WHMCS_DEBUG_SQL', true);
```

### Step 2: Add Debug Logging
```php
<?php
// Debug helper function

function debug_log($message, $data = null, $level = 'info') {
    $logFile = __DIR__ . '/../logs/debug.log';
    $timestamp = date('Y-m-d H:i:s');
    
    $logEntry = "[$timestamp] [$level] $message";
    if ($data !== null) {
        $logEntry .= " | Data: " . json_encode($data);
    }
    $logEntry .= "\n";
    
    file_put_contents($logFile, $logEntry, FILE_APPEND);
}

// Usage in module
function clicodes_example_function($params) {
    debug_log('Function called', $params);
    
    try {
        $result = someOperation($params);
        debug_log('Operation successful', $result);
        return $result;
    } catch (Exception $e) {
        debug_log('Operation failed', [
            'error' => $e->getMessage(),
            'trace' => $e->getTraceAsString()
        ], 'error');
        throw $e;
    }
}
```

### Step 3: Use WHMCS Logging
```php
<?php
// Use WHMCS built-in logging

// Log to activity log
logActivity('Custom module message', $clientId);

// Log module calls
logModuleCall(
    'module_name',           // Module name
    'function_name',        // Function called
    $input,                 // Input data
    $output,                // Output data
    $result                 // Result
);
```

### Step 4: Database Debugging
```php
<?php
// Enable SQL debugging

// Add to configuration.php
$db_debug = true;

// Or enable in code
Capsule::connection()->enableQueryLog();

$results = Capsule::table('tblclients')->get();

// Get logged queries
$queries = Capsule::connection()->getQueryLog();
foreach ($queries as $query) {
    debug_log('SQL Query', [
        'query' => $query['query'],
        'bindings' => $query['bindings'],
        'time' => $query['time']
    ]);
}
```

### Step 5: Xdebug Setup
```bash
# Install Xdebug
apt install php-xdebug

# Configure Xdebug
cat >> /etc/php/8.2/fpm/conf.d/20-xdebug.ini << 'EOF'
xdebug.mode=debug
xdebug.start_with_request=trigger
xdebug.client_host=127.0.0.1
xdebug.client_port=9003
xdebug.log_level=0
EOF

# Or for PHPStorm
cat >> /etc/php/8.2/fpm/conf.d/20-xdebug.ini << 'EOF'
xdebug.mode=develop,debug
xdebug.start_with_request=yes
xdebug.client_host=host.docker.internal
xdebug.idekey=PHPSTORM
EOF

systemctl restart php8.2-fpm
```

### Step 6: Common Debugging Scenarios

**Hook Not Firing:**
```php
<?php
// Add debug hook
add_hook('AfterModuleCreate', 1, function($vars) {
    debug_log('Hook fired', $vars);
    
    // Check if expected data exists
    if (empty($vars['serviceid'])) {
        debug_log('Missing serviceid', $vars, 'error');
    }
});
```

**API Call Failing:**
```php
<?php
// Debug API calls
function apiRequest($url, $data, $headers) {
    debug_log('API Request', [
        'url' => $url,
        'data' => $data,
        'headers' => $headers
    ]);
    
    $ch = curl_init();
    curl_setopt_array($ch, [
        CURLOPT_URL => $url,
        CURLOPT_POST => true,
        CURLOPT_POSTFIELDS => json_encode($data),
        CURLOPT_HTTPHEADER => $headers,
        CURLOPT_RETURNTRANSFER => true,
    ]);
    
    $response = curl_exec($ch);
    $error = curl_error($ch);
    $httpCode = curl_getinfo($ch, CURLINFO_HTTP_CODE);
    
    debug_log('API Response', [
        'response' => $response,
        'error' => $error,
        'http_code' => $httpCode
    ]);
    
    curl_close($ch);
    
    return json_decode($response, true);
}
```

**Template Variables Missing:**
```php
<?php
// Debug template variables
add_hook('ClientAreaPageRun', 1, function($vars) {
    debug_log('Template variables', [
        'page' => $vars['templatefile'],
        'all_vars' => array_keys($vars)
    ]);
});
```

### Step 7: Memory Debugging
```php
<?php
// Check memory usage
function debug_memory($label = '') {
    $mem = memory_get_usage(true) / 1024 / 1024;
    $peak = memory_get_peak_usage(true) / 1024 / 1024;
    
    debug_log("Memory [$label]", [
        'current' => round($mem, 2) . ' MB',
        'peak' => round($peak, 2) . ' MB'
    ]);
}

// Use in functions
function processData($data) {
    debug_memory('start');
    
    // Processing...
    
    debug_memory('after_processing');
    
    return $result;
}
```

### Step 8: Remote Debugging
```php
<?php
// Add to configuration for remote debugging
if ($_SERVER['REMOTE_ADDR'] === 'YOUR_IP') {
    ini_set('display_errors', 1);
    error_reporting(E_ALL);
}

// Log to external service
function logToRemote($message, $data = []) {
    $webhookUrl = 'https://your-logging-service.com/webhook';
    
    $ch = curl_init($webhookUrl);
    curl_setopt_array($ch, [
        CURLOPT_POST => true,
        CURLOPT_POSTFIELDS => json_encode([
            'message' => $message,
            'data' => $data,
            'timestamp' => time(),
            'server' => gethostname()
        ]),
        CURLOPT_RETURNTRANSFER => true,
    ]);
    curl_exec($ch);
    curl_close($ch);
}
```

### Step 9: Debug Toolbar
```php
<?php
// Add debug toolbar hook
add_hook('ClientAreaPage', 1, function($vars) {
    if (!isAdmin()) return;
    
    $debugInfo = [
        'Memory' => round(memory_get_usage(true) / 1024 / 1024, 2) . ' MB',
        'Peak Memory' => round(memory_get_peak_usage(true) / 1024 / 1024, 2) . ' MB',
        'Execution Time' => microtime(true) - $_SERVER['REQUEST_TIME_FLOAT'],
        'DB Queries' => Capsule::connection()->getQueryLogCount(),
    ];
    
    return '<div class="debug-toolbar">' . 
           '<pre>' . print_r($debugInfo, true) . '</pre>' .
           '</div>';
});
```

### Step 10: Debug Commands
```bash
# Enable verbose mode in CLI
php -d display_errors=1 -d error_reporting=E_ALL /var/www/whmcs/cli/module.php

# Debug cron execution
php -d display_errors=1 /var/www/whmcs/crons/cron.php --verbose

# Test module directly
php -r "
define('WHMCS', true);
require '/var/www/whmcs/init.php';
require '/var/www/whmcs/modules/addons/clicodes_example/clicodes_example.php';
var_dump(clicodes_example_config());
"
```

## Debug Checklist
- [ ] Enable display_errors
- [ ] Enable error_reporting
- [ ] Check WHMCS logs (logs/)
- [ ] Check PHP error logs
- [ ] Check Apache/Nginx error logs
- [ ] Check database queries
- [ ] Verify hook execution
- [ ] Check API responses
- [ ] Test with clean cache

## Tags
- debugging
- troubleshooting
- development
- error-handling