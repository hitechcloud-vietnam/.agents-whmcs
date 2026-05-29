# WHMCS Module Best Practices

## Overview

This guide covers best practices for developing WHMCS modules that are secure, performant, maintainable, and follow WHMCS coding standards.

### What Are WHMCS Modules?
Modules in WHMCS extend the core functionality of the platform. They can integrate with payment gateways, provisioning servers, support tools, and more.

### Why Follow Best Practices?
- Ensure compatibility with future WHMCS versions
- Maintain security standards
- Provide consistent user experience
- Enable easy troubleshooting and support

## Technical Details

### Directory Structure

```
addons/
  mymodule/
    ├── VERSION
    ├── README.md
    ├── admin/
    │   └── templates/
    ├── assets/
    │   ├── css/
    │   └── js/
    ├── hooks/
    │   └── HookFunction.php
    ├── lib/
    │   ├── MymoduleApi.php
    │   └── MymoduleHelper.php
    ├── templates/
    ├── lang/
    │   ├── english.php
    │   └── vietnamese.php
    ├── views/
    │   ├── client/
    │   └── admin/
    ├── MymoduleModule.php
    └── bootstrap.php
```

### Module Metadata (VERSION file)

```json
{
    "schema": 2,
    "name": "My Module",
    "version": "1.0.0",
    "description": "Module description here",
    "author": "Your Name",
    "homepage": "https://example.com",
    "keywords": ["keyword1", "keyword2"],
    "fields": {
        "license_key": {
            "type": "text",
            "default": "",
            "description": "Your license key"
        }
    }
}
```

## Code Examples

### Basic Module Structure

```php
<?php
/**
 * Module Name
 *
 * @package    Vendor\MyModule
 * @author     Your Name
 * @copyright  2024 Your Name
 * @license    MIT
 */

namespace Vendor\MyModule;

use WHMCS\Module\Widget\ChartInterface;

class MymoduleModule implements ChartInterface
{
    protected $params = [];

    public function __construct(array $params = [])
    {
        $this->params = $params;
    }

    public function widgetUpdate($params)
    {
        return [
            'target' => $params['target'],
            'value' => $this->calculateValue(),
        ];
    }

    private function calculateValue()
    {
        // Module logic here
        return 0;
    }
}
```

### Configuration Page

```php
<?php
/**
 * Configuration Page Handler
 *
 * @param array $vars Configuration variables from VERSION
 */
function mymodule_config($vars)
{
    return [
        'name' => 'My Module',
        'description' => 'Module description',
        'version' => '1.0.0',
        'author' => 'Your Name',
        'fields' => [
            'api_key' => [
                'Type' => 'text',
                'Size' => '50',
                'Description' => 'Your API Key',
                'Default' => '',
            ],
            'environment' => [
                'Type' => 'dropdown',
                'Options' => 'Production,Sandbox',
                'Description' => 'Select environment',
                'Default' => 'Production',
            ],
            'debug_mode' => [
                'Type' => 'yesno',
                'Description' => 'Enable debug logging',
            ],
        ],
    ];
}
```

## Configuration Options

### Common Configuration Field Types

| Type | Description | Usage |
|------|-------------|-------|
| text | Single line text input | API keys, usernames |
| password | Password field (masked) | Secrets, tokens |
| textarea | Multi-line text area | Large texts, JSON |
| yesno | Yes/No checkbox | Toggle features |
| dropdown | Select dropdown | Environment selection |
| radio | Radio button group | Mode selection |
| checkbox | Multiple checkboxes | Feature flags |
| datetime | Date/time picker | Scheduled tasks |

### Advanced Configuration

```php
function mymodule_config($vars)
{
    return [
        'name' => 'Advanced Module',
        'fields' => [
            // Text with validation
            'webhook_url' => [
                'Type' => 'text',
                'Size' => '80',
                'Description' => 'Webhook endpoint URL',
                'Placeholder' => 'https://',
            ],

            // Dropdown with options
            'log_level' => [
                'Type' => 'dropdown',
                'Options' => 'Debug,Info,Warning,Error',
                'Description' => 'Logging verbosity',
                'Default' => 'Info',
            ],

            // Multi-select
            'features' => [
                'Type' => 'dropdown',
                'Options' => [
                    'full' => 'Full Features',
                    'basic' => 'Basic Only',
                    'custom' => 'Custom Configuration',
                ],
            ],

            // Connection timeout
            'timeout' => [
                'Type' => 'text',
                'Size' => '10',
                'Description' => 'Request timeout in seconds',
                'Default' => '30',
            ],
        ],
    ];
}
```

## Troubleshooting Tips

### Common Issues and Solutions

#### 1. Module Not Appearing in Admin
- Verify module files are in correct directory
- Check file permissions (644 for files, 755 for directories)
- Clear WHMCS cache from Admin > System > Cache
- Check PHP error logs for loading issues

#### 2. Configuration Not Saving
- Verify form action URL is correct
- Check CSRF token handling
- Ensure write permissions on configuration storage

#### 3. Hooks Not Firing
- Verify hook registration in bootstrap
- Check hook priority if multiple modules
- Enable debug logging to see hook calls

### Debug Mode Configuration

```php
// In your module bootstrap
if ($vars['debug_mode'] ?? false) {
    ini_set('display_errors', 1);
    error_reporting(E_ALL);

    // Enable file logging
    $logFile = __DIR__ . '/debug.log';
    set_error_handler(function($errno, $errstr, $errfile, $errline) use ($logFile) {
        file_put_contents($logFile, date('[Y-m-d H:i:s]') . " $errstr in $errfile:$errline\n", FILE_APPEND);
        return false;
    });
}
```

### Testing Checklist

- [ ] Module activates without errors
- [ ] Configuration saves and loads correctly
- [ ] All hooks fire as expected
- [ ] API calls succeed in test environment
- [ ] Error handling works correctly
- [ ] Logging captures necessary information
- [ ] Uninstall removes all module data
- [ ] Upgrade path works correctly

## Security Best Practices

### Input Validation

```php
<?php
function sanitizeInput($data) {
    return [
        'string' => filter_var($data['string'] ?? '', FILTER_SANITIZE_FULL_SPECIAL_CHARS),
        'email' => filter_var($data['email'] ?? '', FILTER_VALIDATE_EMAIL),
        'int' => (int) ($data['int'] ?? 0),
        'url' => filter_var($data['url'] ?? '', FILTER_VALIDATE_URL),
        'ip' => filter_var($data['ip'] ?? '', FILTER_VALIDATE_IP),
    ];
}
```

### SQL Injection Prevention

```php
<?php
// Use prepared statements
$stmt = Capsule::connection()->getPdo()
    ->prepare("SELECT * FROM mod_mymodule WHERE user_id = :user_id");
$stmt->execute(['user_id' => $userId]);
$result = $stmt->fetchAll();
```

### File Path Security

```php
<?php
// Never trust user input for file paths
$filename = basename($input); // Strip directory components
$filepath = __DIR__ . '/uploads/' . $filename; // Enforce directory
```

## Performance Optimization

### Caching Strategy

```php
<?php
// Cache expensive operations
use WHMCS\Caching\CacheFactory;

$cache = CacheFactory::factory('custom');
$cacheKey = 'module_data_' . $clientId;

if ($cached = $cache->get($cacheKey)) {
    return $cached;
}

$data = expensiveOperation();
$cache->set($cacheKey, $data, 3600); // 1 hour TTL

return $data;
```

### Lazy Loading

```php
<?php
// Only load heavy dependencies when needed
public function runHeavyOperation() {
    if (!class_exists('HeavyClass')) {
        require_once __DIR__ . '/lib/HeavyClass.php';
    }
    $instance = new HeavyClass();
    return $instance->process();
}
```

## Version Compatibility

### Minimum Version Support

```php
<?php
/**
 * Module requirements
 * @since 8.0 Required for WHMCS 8.0+
 * @requires WHMCS 8.0
 */
```

### Version Detection

```php
<?php
$whmcsVersion = App::getVersion();
if (version_compare($whmcsVersion, '8.0', '>=')) {
    // Use WHMCS 8.0+ features
} else {
    // Fallback for older versions
}
```

## Support and Updates

### Update Notification System

```php
<?php
function mymodule_check_update() {
    $currentVersion = file_get_contents(__DIR__ . '/VERSION');
    $remoteVersion = file_get_contents('https://yourdomain.com/module/version');

    if (version_compare($remoteVersion, $currentVersion, '>')) {
        return [
            'hasUpdate' => true,
            'current' => $currentVersion,
            'available' => $remoteVersion,
            'url' => 'https://yourdomain.com/module/download',
        ];
    }

    return ['hasUpdate' => false];
}
```

## Additional Resources

- WHMCS Developer Documentation: https://developers.whmcs.com/
- WHMCS Module SDK: https://github.com/whmcs/whmcs-module-sdk
- WHMCS Community Forums: https://forum.whmcs.com/