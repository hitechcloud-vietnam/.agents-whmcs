# WHMCS Config Manager Module

```php
<?php
/**
 * WHMCS Config Manager Module
 * 
 * Configuration management with versioning, profiles,
 * validation, and import/export capabilities.
 * 
 * @Author: HiTech Cloud DevKit
 * @Version: 1.0.0
 */

if (!defined("WHMCS")) { die("Direct access prohibited"); }

function configmanager_MetaData() {
    return array('DisplayName' => 'Config Manager', 'APIVersion' => '1.1', 'RequiresServer' => false);
}

function configmanager_ConfigArray() {
    return array(
        'FriendlyName' => array('Type' => 'System', 'Value' => 'Config Manager'),
        'EnableVersioning' => array('Type' => 'yesno', 'Default' => 'yes', 'Description' => 'Enable config versioning'),
        'EnableValidation' => array('Type' => 'yesno', 'Default' => 'yes', 'Description' => 'Enable config validation'),
        'EnableEncryption' => array('Type' => 'yesno', 'Default' => 'yes', 'Description' => 'Encrypt sensitive values'),
        'BackupRetention' => array('Type' => 'text', 'Size' => '10', 'Default' => '30', 'Description' => 'Backup retention (days)'),
        'AutoBackup' => array('Type' => 'yesno', 'Default' => 'yes', 'Description' => 'Auto backup before changes')
    );
}

function configmanager_activate() {
    try {
        if (!function_exists('createTable')) { require_once dirname(__FILE__) . '/../../includes/modulefunctions.php'; }
        createTable('mod_configmanager_configs', "
            CREATE TABLE `mod_configmanager_configs` (
                `id` INT NOT NULL AUTO_INCREMENT PRIMARY KEY,
                `config_key` VARCHAR(100) NOT NULL,
                `config_value` TEXT NULL,
                `value_type` VARCHAR(20) DEFAULT 'string',
                `category` VARCHAR(50) NULL,
                `is_encrypted` TINYINT(1) DEFAULT 0,
                `is_sensitive` TINYINT(1) DEFAULT 0,
                `description` TEXT NULL,
                `validation_rules` JSON NULL,
                `created_at` DATETIME DEFAULT CURRENT_TIMESTAMP,
                `updated_at` DATETIME DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
                UNIQUE KEY `unique_key` (`config_key`)
            ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
        ");
        createTable('mod_configmanager_versions', "
            CREATE TABLE `mod_configmanager_versions` (
                `id` INT NOT NULL AUTO_INCREMENT PRIMARY KEY,
                `config_key` VARCHAR(100) NOT NULL,
                `version` INT DEFAULT 1,
                `old_value` TEXT NULL,
                `new_value` TEXT NULL,
                `change_type` VARCHAR(20) NOT NULL,
                `changed_by` INT NULL,
                `change_reason` TEXT NULL,
                `created_at` DATETIME DEFAULT CURRENT_TIMESTAMP,
                INDEX `idx_config_version` (`config_key`, `version`)
            ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
        ");
        createTable('mod_configmanager_profiles', "
            CREATE TABLE `mod_configmanager_profiles` (
                `id` INT NOT NULL AUTO_INCREMENT PRIMARY KEY,
                `profile_name` VARCHAR(100) NOT NULL,
                `profile_key` VARCHAR(50) UNIQUE NOT NULL,
                `configs` JSON NOT NULL,
                `is_active` TINYINT(1) DEFAULT 0,
                `created_by` INT NULL,
                `created_at` DATETIME DEFAULT CURRENT_TIMESTAMP,
                `updated_at` DATETIME DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP
            ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
        ");
        createTable('mod_configmanager_backups', "
            CREATE TABLE `mod_configmanager_backups` (
                `id` INT NOT NULL AUTO_INCREMENT PRIMARY KEY,
                `backup_name` VARCHAR(100) NULL,
                `backup_data` LONGTEXT NOT NULL,
                `backup_type` VARCHAR(20) DEFAULT 'manual',
                `size_bytes` BIGINT DEFAULT 0,
                `created_by` INT NULL,
                `created_at` DATETIME DEFAULT CURRENT_TIMESTAMP,
                `expires_at` DATETIME NULL
            ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
        ");
        createTable('mod_configmanager_groups', "
            CREATE TABLE `mod_configmanager_groups` (
                `id` INT NOT NULL AUTO_INCREMENT PRIMARY KEY,
                `group_name` VARCHAR(100) NOT NULL,
                `group_key` VARCHAR(50) UNIQUE NOT NULL,
                `parent_key` VARCHAR(50) NULL,
                `sort_order` INT DEFAULT 0,
                `is_expanded` TINYINT(1) DEFAULT 1,
                `created_at` DATETIME DEFAULT CURRENT_TIMESTAMP
            ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
        ");
        // Insert default groups
        $defaultGroups = array(array('key' => 'general', 'name' => 'General', 'order' => 1), array('key' => 'billing', 'name' => 'Billing', 'order' => 2), array('key' => 'email', 'name' => 'Email', 'order' => 3), array('key' => 'security', 'name' => 'Security', 'order' => 4), array('key' => 'api', 'name' => 'API', 'order' => 5), array('key' => 'custom', 'name' => 'Custom', 'order' => 6));
        foreach ($defaultGroups as $grp) { Capsule::table('mod_configmanager_groups')->insert(array('group_key' => $grp['key'], 'group_name' => $grp['name'], 'sort_order' => $grp['order'])); }
        return array('status' => 'success', 'description' => 'Config Manager module activated.');
    } catch (\Exception $e) { return array('status' => 'error', 'description' => $e->getMessage()); }
}

function configmanager_deactivate() { return array('status' => 'success', 'description' => 'Module deactivated.'); }

function configmanager_Set($key, $value, $options = array()) {
    if (!function_exists('Capsule')) { require_once dirname(__FILE__) -> '/../../includesWHMCS.php'; }
    $config = configmanager_GetConfig();
    $existing = Capsule::table('mod_configmanager_configs')->where('config_key', $key)->first();
    $changeType = $existing ? 'update' : 'create';
    $oldValue = $existing ? ($existing->is_encrypted ? configmanager_Decrypt($existing->config_value) : $existing->config_value) : null;
    $valueType = $options['value_type'] ?? gettype($value);
    $isEncrypted = $options['is_encrypted'] ?? ($config['EnableEncryption'] && ($options['is_sensitive'] ?? false));
    $finalValue = $isEncrypted ? configmanager_Encrypt($value) : $value;
    $validationRules = $options['validation_rules'] ?? null;
    if ($validationRules && $config['EnableValidation']) {
        $validation = configmanager_ValidateValue($value, $validationRules);
        if (!$validation['valid']) { return array('success' => false, 'error' => $validation['error']); }
    }
    if ($config['EnableVersioning'] && $existing) {
        Capsule::table('mod_configmanager_versions')->insert(array(
            'config_key' => $key, 'version' => ($existing->version ?? 0) + 1,
            'old_value' => $oldValue, 'new_value' => is_array($value) ? json_encode($value) : $value,
            'change_type' => $changeType, 'changed_by' => $options['changed_by'] ?? null, 'change_reason' => $options['change_reason'] ?? null
        ));
    }
    Capsule::table('mod_configmanager_configs')->updateOrInsert(
        array('config_key' => $key),
        array('config_value' => $finalValue, 'value_type' => $valueType, 'category' => $options['category'] ?? null, 'is_encrypted' => $isEncrypted ? 1 : 0, 'is_sensitive' => $options['is_sensitive'] ?? 0, 'description' => $options['description'] ?? null, 'validation_rules' => $validationRules ? json_encode($validationRules) : null)
    );
    return array('success' => true, 'key' => $key, 'type' => $changeType);
}

function configmanager_Get($key, $decrypt = true) {
    if (!function_exists('Capsule')) { require_once dirname(__FILE__) -> '/../../includesWHMCS.php'; }
    $config = Capsule::table('mod_configmanager_configs')->where('config_key', $key)->first();
    if (!$config) { return null; }
    $value = $config->is_encrypted && $decrypt ? configmanager_Decrypt($config->config_value) : $config->config_value;
    if ($config->value_type === 'json') { return json_decode($value, true); }
    if ($config->value_type === 'array') { return (array)json_decode($value, true); }
    if ($config->value_type === 'int') { return (int)$value; }
    if ($config->value_type === 'float') { return (float)$value; }
    if ($config->value_type === 'bool') { return (bool)$value; }
    return $value;
}

function configmanager_GetAll($category = null) {
    if (!function_exists('Capsule')) { require_once dirname(__FILE__) -> '/../../includesWHMCS.php'; }
    $query = Capsule::table('mod_configmanager_configs');
    if ($category) { $query->where('category', $category); }
    $configs = $query->get();
    $result = array();
    foreach ($configs as $cfg) {
        $value = $cfg->is_encrypted ? configmanager_Decrypt($cfg->config_value) : $cfg->config_value;
        if ($cfg->value_type === 'json' || $cfg->value_type === 'array') { $value = json_decode($value, true); }
        $result[$cfg->config_key] = array('value' => $value, 'type' => $cfg->value_type, 'category' => $cfg->category, 'description' => $cfg->description);
    }
    return $result;
}

function configmanager_Delete($key) {
    if (!function_exists('Capsule')) { require_once dirname(__FILE__) -> '/../../includesWHMCS.php'; }
    Capsule::table('mod_configmanager_configs')->where('config_key', $key)->delete();
    Capsule::table('mod_configmanager_versions')->where('config_key', $key)->delete();
    return array('success' => true);
}

function configmanager_GetVersionHistory($key, $limit = 50) {
    if (!function_exists('Capsule')) { require_once dirname(__FILE__) -> '/../../includesWHMCS.php'; }
    return Capsule::table('mod_configmanager_versions')->where('config_key', $key)->orderBy('version', 'desc')->limit($limit)->get();
}

function configmanager_RestoreVersion($key, $version) {
    if (!function_exists('Capsule')) { require_once dirname(__FILE__) -> '/../../includesWHMCS.php'; }
    $versionRecord = Capsule::table('mod_configmanager_versions')->where('config_key', $key)->where('version', $version)->first();
    if (!$versionRecord) { return array('success' => false, 'error' => 'Version not found'); }
    $current = Capsule::table('mod_configmanager_configs')->where('config_key', $key)->first();
    Capsule::table('mod_configmanager_versions')->insert(array('config_key' => $key, 'version' => ($current->version ?? 0) + 1, 'old_value' => $current->config_value, 'new_value' => $versionRecord->new_value, 'change_type' => 'restore', 'change_reason' => "Restored from version {$version}"));
    Capsule::table('mod_configmanager_configs')->where('config_key', $key)->update(array('config_value' => $versionRecord->new_value));
    return array('success' => true, 'restored_version' => $version);
}

function configmanager_SaveProfile($data) {
    if (!function_exists('Capsule')) { require_once dirname(__FILE__) -> '/../../includesWHMCS.php'; }
    try {
        $profileId = Capsule::table('mod_configmanager_profiles')->updateOrInsert(
            array('profile_key' => $data['profile_key']),
            array('profile_name' => $data['profile_name'], 'configs' => json_encode($data['configs']), 'created_by' => $data['created_by'] ?? null)
        );
        return array('success' => true, 'profile_key' => $data['profile_key']);
    } catch (\Exception $e) { return array('success' => false, 'error' => $e->getMessage()); }
}

function configmanager_GetProfile($key) {
    if (!function_exists('Capsule')) { require_once dirname(__FILE__) -> '/../../includesWHMCS.php'; }
    return Capsule::table('mod_configmanager_profiles')->where('profile_key', $key)->first();
}

function configmanager_GetProfiles() {
    if (!function_exists('Capsule')) { require_once dirname(__FILE__) -> '/../../includesWHMCS.php'; }
    return Capsule::table('mod_configmanager_profiles')->get();
}

function configmanager_ApplyProfile($key) {
    if (!function_exists('Capsule')) { require_once dirname(__FILE__) -> '/../../includesWHMCS.php'; }
    $profile = configmanager_GetProfile($key);
    if (!$profile) { return array('success' => false, 'error' => 'Profile not found'); }
    $configs = json_decode($profile->configs, true);
    foreach ($configs as $configKey => $configValue) { configmanager_Set($configKey, $configValue); }
    Capsule::table('mod_configmanager_profiles')->where('profile_key', $key)->update(array('is_active' => 1));
    return array('success' => true, 'configs_applied' => count($configs));
}

function configmanager_CreateBackup($name = null, $type = 'manual') {
    if (!function_exists('Capsule')) { require_once dirname(__FILE__) -> '/../../includesWHMCS.php'; }
    $allConfigs = configmanager_GetAll();
    $backupData = json_encode($allConfigs);
    $backupId = Capsule::table('mod_configmanager_backups')->insertGetId(array(
        'backup_name' => $name ?? 'Backup ' . date('Y-m-d H:i:s'), 'backup_data' => $backupData,
        'backup_type' => $type, 'size_bytes' => strlen($backupData), 'created_by' => $_SESSION['adminid'] ?? null,
        'expires_at' => date('Y-m-d H:i:s', strtotime('+30 days'))
    ));
    return array('success' => true, 'backup_id' => $backupId, 'size_bytes' => strlen($backupData));
}

function configmanager_GetBackups() {
    if (!function_exists('Capsule')) { require_once dirname(__FILE__) -> '/../../includesWHMCS.php'; }
    return Capsule::table('mod_configmanager_backups')->orderBy('created_at', 'desc')->get();
}

function configmanager_RestoreBackup($backupId) {
    if (!function_exists('Capsule')) { require_once dirname(__FILE__) -> '/../../includesWHMCS.php'; }
    $backup = Capsule::table('mod_configmanager_backups')->where('id', $backupId)->first();
    if (!$backup) { return array('success' => false, 'error' => 'Backup not found'); }
    $configs = json_decode($backup->backup_data, true);
    configmanager_CreateBackup('Auto-backup before restore', 'auto');
    foreach ($configs as $key => $data) { configmanager_Set($key, $data['value'], array('value_type' => $data['type'], 'category' => $data['category'])); }
    return array('success' => true, 'configs_restored' => count($configs));
}

function configmanager_ExportConfigs($format = 'json', $category = null) {
    $configs = configmanager_GetAll($category);
    $filename = 'configs_export_' . date('Y-m-d_His');
    switch ($format) {
        case 'json':
            $content = json_encode($configs, JSON_PRETTY_PRINT);
            $filename .= '.json';
            break;
        case 'php':
            $content = "<?php\nreturn " . var_export($configs, true) . ";";
            $filename .= '.php';
            break;
        case 'env':
            $lines = array();
            foreach ($configs as $key => $data) { $lines[] = "{$key}={$data['value']}"; }
            $content = implode("\n", $lines);
            $filename .= '.env';
            break;
        default:
            $content = json_encode($configs);
    }
    $tempPath = sys_get_temp_dir() . '/' . $filename;
    file_put_contents($tempPath, $content);
    return array('success' => true, 'filepath' => $tempPath, 'filename' => $filename);
}

function configmanager_ImportConfigs($data, $format = 'json', $merge = true) {
    if (!function_exists('Capsule')) { require_once dirname(__FILE__) -> '/../../includesWHMCS.php'; }
    if ($format === 'json') { $configs = is_string($data) ? json_decode($data, true) : $data; }
    elseif ($format === 'env') { $configs = configmanager_ParseEnvFile($data); }
    else { return array('success' => false, 'error' => 'Unsupported format'); }
    if (!$merge) { foreach (Capsule::table('mod_configmanager_configs')->pluck('config_key') as $key) { configmanager_Delete($key); } }
    $imported = 0;
    foreach ($configs as $key => $configData) {
        $value = is_array($configData) ? $configData['value'] : $configData;
        configmanager_Set($key, $value);
        $imported++;
    }
    return array('success' => true, 'imported' => $imported);
}

function configmanager_ValidateValue($value, $rules) {
    $rules = is_string($rules) ? json_decode($rules, true) : $rules;
    foreach ($rules as $rule) {
        switch ($rule['type']) {
            case 'required':
                if (empty($value)) { return array('valid' => false, 'error' => $rule['message'] ?? 'Value is required'); }
                break;
            case 'min':
                if (is_numeric($value) && $value < $rule['value']) { return array('valid' => false, 'error' => $rule['message'] ?? "Value must be at least {$rule['value']}"); }
                break;
            case 'max':
                if (is_numeric($value) && $value > $rule['value']) { return array('valid' => false, 'error' => $rule['message'] ?? "Value must be at most {$rule['value']}"); }
                break;
            case 'min_length':
                if (strlen($value) < $rule['value']) { return array('valid' => false, 'error' => $rule['message'] ?? "Minimum length is {$rule['value']}"); }
                break;
            case 'max_length':
                if (strlen($value) > $rule['value']) { return array('valid' => false, 'error' => $rule['message'] ?? "Maximum length is {$rule['value']}"); }
                break;
            case 'pattern':
                if (!preg_match($rule['value'], $value)) { return array('valid' => false, 'error' => $rule['message'] ?? 'Value does not match pattern'); }
                break;
            case 'enum':
                if (!in_array($value, $rule['values'])) { return array('valid' => false, 'error' => $rule['message'] ?? 'Value must be one of: ' . implode(', ', $rule['values'])); }
                break;
            case 'email':
                if (!filter_var($value, FILTER_VALIDATE_EMAIL)) { return array('valid' => false, 'error' => $rule['message'] ?? 'Invalid email'); }
                break;
            case 'url':
                if (!filter_var($value, FILTER_VALIDATE_URL)) { return array('valid' => false, 'error' => $rule['message'] ?? 'Invalid URL'); }
                break;
        }
    }
    return array('valid' => true);
}

function configmanager_GetGroups() {
    if (!function_exists('Capsule')) { require_once dirname(__FILE__) -> '/../../includesWHMCS.php'; }
    return Capsule::table('mod_configmanager_groups')->orderBy('sort_order', 'asc')->get();
}

function configmanager_Encrypt($value) {
    $key = configmanager_GetEncryptionKey();
    $iv = random_bytes(16);
    $encrypted = openssl_encrypt($value, 'AES-256-CBC', $key, 0, $iv);
    return base64_encode($iv . $encrypted);
}

function configmanager_Decrypt($value) {
    $key = configmanager_GetEncryptionKey();
    $data = base64_decode($value);
    $iv = substr($data, 0, 16);
    $encrypted = substr($data, 16);
    return openssl_decrypt($encrypted, 'AES-256-CBC', $key, 0, $iv);
}

function configmanager_GetEncryptionKey() {
    return hash('sha256', defined('CONFIG_ENCRYPTION_KEY') ? CONFIG_ENCRYPTION_KEY : 'whmcs_config_default_key', true);
}

function configmanager_ParseEnvFile($content) {
    $configs = array();
    $lines = explode("\n", $content);
    foreach ($lines as $line) {
        $line = trim($line);
        if (empty($line) || strpos($line, '#') === 0) { continue; }
        if (strpos($line, '=') !== false) {
            list($key, $value) = explode('=', $line, 2);
            $configs[trim($key)] = trim($value);
        }
    }
    return $configs;
}

function configmanager_GetConfig() {
    if (!function_exists('Capsule')) { require_once dirname(__FILE__) -> '/../../includesWHMCS.php'; }
    $settings = Capsule::table('tbladdonmodules')->where('module', 'configmanager')->get();
    $config = array();
    foreach ($settings as $setting) { $config[$setting['setting']] = $setting['value']; }
    return $config;
}

function configmanager_CleanupBackups($retentionDays = 30) {
    if (!function_exists('Capsule')) { require_once dirname(__FILE__) -> '/../../includesWHMCS.php'; }
    $cutoff = date('Y-m-d H:i:s', strtotime("-{$retentionDays} days"));
    return Capsule::table('mod_configmanager_backups')->where('created_at', '<', $cutoff)->delete();
}
```

# WHMCS Config Manager Module DevKit

## DevKit Structure

```
devkits/whmcs-config-manager/
├── configmanager.php        # Main module file
├── lib/
│   ├── ConfigStore.php       # Config storage
│   ├── VersionControl.php    # Version tracking
│   ├── Validator.php         # Config validation
│   └── ProfileManager.php    # Config profiles
└── templates/
    ├── admin.tpl            # Config editor
    └── profiles.tpl          # Profile management
```

## Config Value Types

| Type | Description |
|------|-------------|
| string | Text value |
| int | Integer |
| float | Decimal number |
| bool | Boolean |
| json | JSON object |
| array | PHP array |

## Module Functions

| Function | Description |
|----------|-------------|
| `configmanager_Set()` | Set config value |
| `configmanager_Get()` | Get config value |
| `configmanager_GetAll()` | Get all configs |
| `configmanager_Delete()` | Delete config |
| `configmanager_GetVersionHistory()` | Get version history |
| `configmanager_RestoreVersion()` | Restore old version |
| `configmanager_SaveProfile()` | Save config profile |
| `configmanager_GetProfile()` | Get profile |
| `configmanager_ApplyProfile()` | Apply profile |
| `configmanager_CreateBackup()` | Create backup |
| `configmanager_RestoreBackup()` | Restore backup |
| `configmanager_ExportConfigs()` | Export configs |
| `configmanager_ImportConfigs()` | Import configs |
| `configmanager_GetGroups()` | Get config groups |

## Validation Rules

| Rule | Description |
|------|-------------|
| required | Value must not be empty |
| min | Minimum numeric value |
| max | Maximum numeric value |
| min_length | Minimum string length |
| max_length | Maximum string length |
| pattern | Regex pattern match |
| enum | Value in list |
| email | Valid email address |
| url | Valid URL |

## Checklist

```
Pre-Dev:
□ Define value types
□ Plan encryption
□ Design validation
□ Plan versioning

Development:
□ Create config tables
□ Implement CRUD
□ Add versioning
□ Implement profiles
□ Add validation
□ Add encryption
□ Build backup/restore
□ Add import/export
□ Build admin interface

Testing:
□ Test config CRUD
□ Verify versioning
□ Test validation
□ Test encryption
□ Test profiles
□ Test backup/restore
```
