# WHMCS Configuration Class Reference

**Version:** 8.x | **Updated:** 2026-05-29
**Related Skills:** `whmcs-configuration-management`, `whmcs-security-checklist`

---

## Overview

The WHMCS Configuration class (`WHMCS\Config\Setting`) provides access to system-wide configuration settings. WHMCS also supports additional configuration mechanisms including module settings, environment variables, and encrypted credential storage.

---

## WHMCS Settings Table

WHMCS stores system settings in the `tblconfiguration` table. The `Setting` class provides the primary interface for reading and writing these values.

### Reading Settings

```php
<?php
use WHMCS\Config\Setting;

// Get single setting value
$companyName = Setting::getValue('CompanyName');

// Get setting with default
$apiKey = Setting::getValue('ApiKey', 'default_value');

// Get setting with type hint
$amount = (float) Setting::getValue('DefaultCreditLimit', 0);
```

### Writing Settings

```php
use WHMCS\Config\Setting;

// Set a value
Setting::setValue('CompanyName', 'My Hosting Company');

// Set multiple values
Setting::setValues([
    'CompanyName' => 'My Hosting Company',
    ' TorreEmail' => 'support@example.com',
    'DefaultCurrency' => 1,
]);
```

### Common System Settings

| Setting Key | Type | Description |
|-------------|------|-------------|
| `CompanyName` | string | Company name displayed in emails |
| `Email' | string | Default from email address |
| `DefaultCurrency` | int | Default currency ID |
| `PaymentGateway` | string | Default payment gateway |
| `SystemURL` | string | WHMCS installation URL |
| `SystemSSLURL` | string | HTTPS URL |
| `Template` | string | Active client template |
| `AdminTemplate` | string | Active admin template |
| `MaintenanceMode` | bool | Maintenance mode status |
| `apitoken` | string | API token value |
| `apirefresh` | string | API refresh token |

---

## Module Configuration

### Per-Module Settings

Modules store their configuration in `tblappconfig` or through module-specific functions:

```php
/**
 * Module configuration function
 */
function mymodule_config(): array {
    return [
        'name' => [
            'Type' => 'System',
            'Value' => 'My Module',
        ],
        'api_key' => [
            'FriendlyName' => 'API Key',
            'Type' => 'password',
            'Size' => '40',
            'Description' => 'Enter your API key',
        ],
        'environment' => [
            'FriendlyName' => 'Environment',
            'Type' => 'dropdown',
            'Options' => [
                'production' => 'Production',
                'sandbox' => 'Sandbox',
            ],
            'Default' => 'production',
        ],
    ];
}
```

### Reading Module Settings

```php
/**
 * Get module configuration from params array
 * @param array $params Standard WHMCS params array
 */
function mymodule_activate(array $params): array {
    $apiKey = $params['api_key'];
    $env = $params['environment'];

    // Store module-specific settings
    Capsule::table('mod_mymodule_settings')->insert([
        'setting_key' => 'api_key',
        'setting_value' => encrypt($encrypted) ?: $apiKey,
        'setting_key' => 'environment',
        'setting_value' => $env,
    ]);

    return ['status' => 'success'];
}
```

### Settings Manager Class

```php
<?php
/**
 * Module Settings Manager
 */
class ModuleSettings {
    private string $module;
    private array $cache = [];

    public function __construct(string $module) {
        $this->module = $module;
    }

    /**
     * Get a setting value
     */
    public function get(string $key, $default = null) {
        if (isset($this->cache[$key])) {
            return $this->cache[$key];
        }

        $row = Capsule::table('mod_' . $this->module . '_settings')
            ->where('setting_key', $key)
            ->first();

        $value = $row ? $row->setting_value : $default;

        $this->cache[$key] = $value;
        return $value;
    }

    /**
     * Set a setting value
     */
    public function set(string $key, $value): void {
        Capsule::table('mod_' . $this->module . '_settings')
            ->updateOrInsert(
                ['setting_key' => $key],
                [
                    'setting_value' => is_array($value) ? json_encode($value) : $value,
                    'updated_at' => date('Y-m-d H:i:s'),
                ]
            );

        $this->cache[$key] = $value;
    }

    /**
     * Get all settings
     */
    public function all(): array {
        if (!empty($this->cache)) {
            return $this->cache;
        }

        $rows = Capsule::table('mod_' . $this->module . '_settings')
            ->get();

        foreach ($rows as $row) {
            $this->cache[$row->setting_key] = $row->setting_value;
        }

        return $this->cache;
    }

    /**
     * Delete a setting
     */
    public function delete(string $key): void {
        Capsule::table('mod_' . $this->module . '_settings')
            ->where('setting_key', $key)
            ->delete();

        unset($this->cache[$key]);
    }

    /**
     * Clear settings cache
     */
    public function flush(): void {
        $this->cache = [];
    }
}
```

---

## Encrypted Credential Storage

### Using Laravel's Crypt Facade

```php
<?php
use Illuminate\Support\Facades\Crypt;
use WHMCS\Config\Setting;

/**
 * Store encrypted credentials
 */
function storeApiCredentials(array $credentials): void {
    Setting::setValue('ApiKey', Crypt::encrypt($credentials['api_key']));
    Setting::setValue('ApiSecret', Crypt::encrypt($credentials['api_secret']));
}

/**
 * Retrieve decrypted credentials
 */
function getApiCredentials(): array {
    return [
        'api_key' => Crypt::decrypt(Setting::getValue('ApiKey')),
        'api_secret' => Crypt::decrypt(Setting::getValue('ApiSecret')),
    ];
}
```

### Custom Encryption Helper

```php
<?php
/**
 * Secure configuration storage with module-specific encryption
 */
class SecureConfig {
    private string $module;
    private string $encryptionKey;

    public function __construct(string $module) {
        $this->module = $module;
        $this->encryptionKey = $this->getModuleKey();
    }

    private function getModuleKey(): string {
        // Derive key from module name and WHMCS license
        $licenseKey = Setting::getValue('LicenseKey');
        return hash('sha256', $this->module . $licenseKey);
    }

    /**
     * Encrypt and store sensitive data
     */
    public function storeEncrypted(string $key, string $value): void {
        $iv = random_bytes(16);
        $encrypted = openssl_encrypt($value, 'AES-256-CBC', $this->encryptionKey, 0, $iv);

        Capsule::table('mod_' . $this->module . '_secrets')->updateOrInsert(
            ['secret_key' => $key],
            [
                'secret_value' => base64_encode($iv . $encrypted),
                'updated_at' => date('Y-m-d H:i:s'),
            ]
        );
    }

    /**
     * Retrieve and decrypt data
     */
    public function getDecrypted(string $key, string $default = ''): string {
        $row = Capsule::table('mod_' . $this->module . '_secrets')
            ->where('secret_key', $key)
            ->first();

        if (!$row) {
            return $default;
        }

        $data = base64_decode($row->secret_value);
        $iv = substr($data, 0, 16);
        $encrypted = substr($data, 16);

        return openssl_decrypt($encrypted, 'AES-256-CBC', $this->encryptionKey, 0, $iv)
            ?: $default;
    }
}
```

---

## Environment Configuration

### Environment Variables

```php
<?php
/**
 * Environment-based configuration
 */
class EnvConfig {
    private static string $environment;

    public static function getEnvironment(): string {
        if (self::$environment === null) {
            self::$environment = getenv('WHMCS_ENV') ?: 'production';
        }
        return self::$environment;
    }

    public static function isProduction(): bool {
        return self::getEnvironment() === 'production';
    }

    public static function isSandbox(): bool {
        return self::getEnvironment() === 'sandbox';
    }

    public static function isDevelopment(): bool {
        return self::getEnvironment() === 'development';
    }
}

// Usage
if (EnvConfig::isSandbox()) {
    $apiUrl = 'https://sandbox-api.example.com';
    $debugMode = true;
} else {
    $apiUrl = 'https://api.example.com';
    $debugMode = false;
}
```

### Database Configuration

```php
/**
 * Multi-database configuration
 */
class DatabaseConfig {
    public static function getConnection(string $connection = 'default'): array {
        $connections = [
            'default' => [
                'host' => Setting::getValue('mysql_host'),
                'database' => Setting::getValue('mysql_db'),
                'username' => Setting::getValue('mysql_username'),
                'password' => Setting::getValue('mysql_password'),
            ],
            'reporting' => [
                'host' => Setting::getValue('reporting_host'),
                'database' => Setting::getValue('reporting_db'),
                'username' => Setting::getValue('reporting_user'),
                'password' => Setting::getValue('reporting_pass'),
            ],
        ];

        return $connections[$connection] ?? $connections['default'];
    }
}
```

---

## Configuration File Constants

### Core Configuration Constants

```php
<?php
// From configuration.php
defined('WHMCS') or die('Direct access prohibited');

// Database
define('DB_Host', 'localhost');
define('DB_USERNAME', 'whmcs_user');
define('DB_PASSWORD', 'secure_password');
define('DB_DATABASE', 'whmcs_db');
define('DB_PREFIX', 'tbl');

// System paths
define('ROOTDIR', dirname(__DIR__));
define('WHDIR', ROOTDIR . '/whmcs');
define('INCLUDESDIR', INCROOT . '/includes');
define('TEMPLATESDIR', INCLUDESDIR . '/templates');
define('ATTACHMENTS_DIR', ROOTDIR . '/attachments');

// System URLs
define('WHMCS_SYSTEM_URL', 'https://whmcs.example.com');
define('WHMCS_SYSTEM_SSLURL', 'https://whmcs.example.com');
define('ADMIN_PATH', 'admin');

// App settings
define('CONFIGURATION_VERSION', '8.0');
define('PHP_BLOAT_REQUIRES_ADMIN_AUTH', true);
```

### Reading Configuration Constants

```php
<?php
// Include configuration if not already loaded
$configFile = dirname(__DIR__) . '/configuration.php';

if (file_exists($configFile)) {
    require_once $configFile;
}

// Use constants
$systemUrl = defined('WHMCS_SYSTEM_URL') ? WHMCS_SYSTEM_URL : '';
$dbPrefix = defined('DB_PREFIX') ? DB_PREFIX : 'tbl';
```

---

## Runtime Configuration

### Dynamic Feature Flags

```php
<?php
/**
 * Feature flag manager
 */
class FeatureFlags {
    private array $flags = [];

    public function __construct() {
        $this->loadFlags();
    }

    private function loadFlags(): void {
        $rows = Capsule::table('mod_feature_flags')
            ->where('is_active', 1)
            ->get();

        foreach ($rows as $row) {
            $this->flags[$row->flag_name] = json_decode($row->flag_value, true);
        }
    }

    public function isEnabled(string $flag, bool $default = false): bool {
        return $this->flags[$flag]['enabled'] ?? $default;
    }

    public function getValue(string $flag, $default = null) {
        return $this->flags[$flag]['value'] ?? $default;
    }

    public function setFlag(string $flag, array $value): void {
        Capsule::table('mod_feature_flags')
            ->updateOrInsert(
                ['flag_name' => $flag],
                [
                    'flag_value' => json_encode($value),
                    'is_active' => 1,
                    'updated_at' => date('Y-m-d H:i:s'),
                ]
            );

        $this->flags[$flag] = $value;
    }

    public function disableFlag(string $flag): void {
        Capsule::table('mod_feature_flags')
            ->where('flag_name', $flag)
            ->update(['is_active' => 0]);

        unset($this->flags[$flag]);
    }
}
```

---

## Configuration Migration

### Upgrade Settings on Module Activation

```php
function mymodule_upgrade(array $vars): void {
    $currentVersion = $vars['version'];
    $targetVersion = 200;

    if ($currentVersion < 100) {
        // v1.0 -> v1.1: Add new settings
        Capsule::table('mod_mymodule_settings')->insert([
            ['setting_key' => 'new_setting_1', 'setting_value' => 'default'],
            ['setting_key' => 'new_setting_2', 'setting_value' => 'default'],
        ]);
    }

    if ($currentVersion < 200) {
        // v1.1 -> v2.0: Migrate old settings format
        $oldSettings = Capsule::table('mod_mymodule_config')
            ->get();

        foreach ($oldSettings as $setting) {
            Capsule::table('mod_mymodule_settings')->updateOrInsert(
                ['setting_key' => $setting->config_key],
                ['setting_value' => $setting->config_value]
            );
        }
    }
}
```

---

## Best Practices

1. **Encrypt sensitive data** - Never store API keys or secrets in plain text
2. **Use module-specific tables** - Avoid polluting `tblconfiguration`
3. **Cache frequently-read settings** - Avoid repeated database queries
4. **Validate on write** - Validate settings before storing
5. **Provide defaults** - Always have fallback values for missing settings
6. **Document settings** - Create admin-facing documentation for module settings
7. **Use typed retrieval** - Cast settings to appropriate types

---

## Related Documentation

- [Configuration Management Skill](../skills/whmcs-configuration-management)
- [Security Best Practices](security-best-practices.md)
- [Module Security Standards](module-security-standards.md)
