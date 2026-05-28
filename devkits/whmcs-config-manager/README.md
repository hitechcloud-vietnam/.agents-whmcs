# WHMCS Config Manager Module

Configuration management with versioning, profiles, validation, and import/export.

## Features

- Version control for all configs
- Encryption for sensitive values
- Configuration profiles
- Validation rules
- Backup and restore
- Import/export (JSON, PHP, ENV)
- Group organization
- Change history

## Installation

1. Copy `configmanager.php` to `/path/to/whmcs/modules/addons/configmanager/`
2. Activate in WHMCS Admin > System Settings > Module Addons
3. Configure encryption key if needed

## Usage

```php
// Set config value
configmanager_Set('my_setting', 'value');
configmanager_Set('api_key', 'secret123', array(
    'category' => 'api',
    'is_sensitive' => true,
    'description' => 'API authentication key',
    'validation_rules' => array(
        array('type' => 'required'),
        array('type' => 'min_length', 'value' => 10)
    )
));

// Set with type
configmanager_Set('max_users', 100, array('value_type' => 'int'));
configmanager_Set('tax_rate', 0.085, array('value_type' => 'float'));
configmanager_Set('features', array('feature1', 'feature2'), array('value_type' => 'json'));

// Get config
$value = configmanager_Get('my_setting');
$key = configmanager_Get('api_key'); // Automatically decrypted

// Get all configs
$allConfigs = configmanager_GetAll();
$allConfigs = configmanager_GetAll('api'); // By category

// Delete config
configmanager_Delete('old_setting');

// Version history
$history = configmanager_GetVersionHistory('my_setting');

// Restore version
configmanager_RestoreVersion('my_setting', 3);

// Save profile
configmanager_SaveProfile(array(
    'profile_key' => 'production',
    'profile_name' => 'Production Settings',
    'configs' => array(
        'debug_mode' => false,
        'api_endpoint' => 'https://api.production.com',
        'log_level' => 'error'
    ),
    'created_by' => $adminId
));

// Get profile
$profile = configmanager_GetProfile('production');

// Get all profiles
$profiles = configmanager_GetProfiles();

// Apply profile
$result = configmanager_ApplyProfile('production');
// Returns: success, configs_applied

// Create backup
$backup = configmanager_CreateBackup('Pre-deployment backup', 'manual');
// Returns: success, backup_id, size_bytes

// Get backups
$backups = configmanager_GetBackups();

// Restore backup
$result = configmanager_RestoreBackup($backupId);
// Returns: success, configs_restored

// Export configs
$export = configmanager_ExportConfigs('json');
// Returns: success, filepath, filename

// Also supports 'php' and 'env' formats
$export = configmanager_ExportConfigs('env');

// Import configs
$import = configmanager_ImportConfigs($jsonString, 'json', true);
// Parameters: data, format, merge (true = merge, false = replace all)

// Import from ENV file
$envContent = file_get_contents('.env');
$import = configmanager_ImportConfigs($envContent, 'env');

// Get config groups
$groups = configmanager_GetGroups();

// Cleanup old backups
$deleted = configmanager_CleanupBackups(30);
```

## Configuration Options

| Option | Type | Default | Description |
|--------|------|---------|-------------|
| EnableVersioning | yesno | yes | Enable versioning |
| EnableValidation | yesno | yes | Enable validation |
| EnableEncryption | yesno | yes | Encrypt sensitive |
| BackupRetention | text | 30 | Retention (days) |
| AutoBackup | yesno | yes | Auto backup |

## Value Types

| Type | PHP Type | Example |
|------|----------|---------|
| string | string | "hello" |
| int | integer | 42 |
| float | float | 3.14 |
| bool | boolean | true |
| json | array | ["a","b"] |
| array | array | ['x' => 1] |

## Validation Rules

```php
// Required
array('type' => 'required', 'message' => 'This field is required')

// Numeric range
array('type' => 'min', 'value' => 0)
array('type' => 'max', 'value' => 100)

// String length
array('type' => 'min_length', 'value' => 5)
array('type' => 'max_length', 'value' => 255)

// Pattern
array('type' => 'pattern', 'value' => '/^[a-z]+$/')

// Enum
array('type' => 'enum', 'values' => ['option1', 'option2'])

// Email
array('type' => 'email')

// URL
array('type' => 'url')

// Combined example
'validation_rules' => array(
    array('type' => 'required'),
    array('type' => 'min_length', 'value' => 10),
    array('type' => 'max_length', 'value' => 100),
    array('type' => 'pattern', 'value' => '/^[a-zA-Z0-9_-]+$/')
)
```

## Encryption

Sensitive values are encrypted using AES-256-CBC:

```php
configmanager_Set('db_password', 'secret123', array(
    'is_sensitive' => true
));
```

Values with `is_sensitive` are automatically encrypted.

## Profiles

Profiles let you save and switch between configuration sets:

```php
// Development profile
configmanager_SaveProfile(array(
    'profile_key' => 'development',
    'profile_name' => 'Development',
    'configs' => array(
        'debug_mode' => true,
        'log_level' => 'debug',
        'cache_enabled' => false
    )
));

// Production profile
configmanager_SaveProfile(array(
    'profile_key' => 'production',
    'profile_name' => 'Production',
    'configs' => array(
        'debug_mode' => false,
        'log_level' => 'error',
        'cache_enabled' => true
    )
));

// Switch to production
configmanager_ApplyProfile('production');
```

## Database Tables

- `mod_configmanager_configs` - Configuration values
- `mod_configmanager_versions` - Version history
- `mod_configmanager_profiles` - Configuration profiles
- `mod_configmanager_backups` - Backup storage
- `mod_configmanager_groups` - Configuration groups
