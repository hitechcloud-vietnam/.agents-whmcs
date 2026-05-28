# WHMCS Secret Manager Module

```php
<?php
/**
 * WHMCS Secret Manager Module
 * 
 * Secrets/credentials management with encryption,
 * access control, audit logging, and rotation.
 * 
 * @Author: HiTech Cloud DevKit
 * @Version: 1.0.0
 */

if (!defined("WHMCS")) { die("Direct access prohibited"); }

function secretmanager_MetaData() {
    return array('DisplayName' => 'Secret Manager', 'APIVersion' => '1.1', 'RequiresServer' => false);
}

function secretmanager_ConfigArray() {
    return array(
        'FriendlyName' => array('Type' => 'System', 'Value' => 'Secret Manager'),
        'EncryptionAlgorithm' => array('Type' => 'dropdown', 'Options' => 'AES-256-GCM,AES-256-CBC', 'Default' => 'AES-256-GCM', 'Description' => 'Encryption algorithm'),
        'KeyDerivation' => array('Type' => 'dropdown', 'Options' => 'argon2id,bcrypt,pbkdf2', 'Default' => 'argon2id', 'Description' => 'Key derivation function'),
        'EnableAuditLog' => array('Type' => 'yesno', 'Default' => 'yes', 'Description' => 'Enable audit logging'),
        'EnableRotation' => array('Type' => 'yesno', 'Default' => 'yes', 'Description' => 'Enable auto-rotation'),
        'RotationDays' => array('Type' => 'text', 'Size' => '10', 'Default' => '90', 'Description' => 'Rotation period (days)'),
        'MasterKey' => array('Type' => 'password', 'Description' => 'Master encryption key')
    );
}

function secretmanager_activate() {
    try {
        if (!function_exists('createTable')) { require_once dirname(__FILE__) . '/../../includes/modulefunctions.php'; }
        createTable('mod_secretmanager_secrets', "
            CREATE TABLE `mod_secretmanager_secrets` (
                `id` INT NOT NULL AUTO_INCREMENT PRIMARY KEY,
                `secret_key` VARCHAR(100) NOT NULL,
                `secret_name` VARCHAR(255) NOT NULL,
                `encrypted_value` TEXT NOT NULL,
                `iv` VARCHAR(64) NULL,
                `tag` VARCHAR(64) NULL,
                `secret_type` VARCHAR(30) NOT NULL,
                `category` VARCHAR(50) NULL,
                `version` INT DEFAULT 1,
                `is_active` TINYINT(1) DEFAULT 1,
                `expires_at` DATETIME NULL,
                `last_rotated` DATETIME NULL,
                `rotation_policy` VARCHAR(20) NULL,
                `created_by` INT NULL,
                `created_at` DATETIME DEFAULT CURRENT_TIMESTAMP,
                `updated_at` DATETIME DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
                UNIQUE KEY `unique_key_version` (`secret_key`, `version`),
                INDEX `idx_category` (`category`)
            ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
        ");
        createTable('mod_secretmanager_access', "
            CREATE TABLE `mod_secretmanager_access` (
                `id` INT NOT NULL AUTO_INCREMENT PRIMARY KEY,
                `secret_id` INT NOT NULL,
                `user_id` INT NULL,
                `role_id` INT NULL,
                `permission` VARCHAR(20) NOT NULL,
                `granted_by` INT NULL,
                `granted_at` DATETIME DEFAULT CURRENT_TIMESTAMP,
                `expires_at` DATETIME NULL,
                INDEX `idx_secret_user` (`secret_id`, `user_id`)
            ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
        ");
        createTable('mod_secretmanager_audit', "
            CREATE TABLE `mod_secretmanager_audit` (
                `id` BIGINT NOT NULL AUTO_INCREMENT PRIMARY KEY,
                `secret_id` INT NOT NULL,
                `secret_key` VARCHAR(100) NOT NULL,
                `action` VARCHAR(30) NOT NULL,
                `performed_by` INT NULL,
                `performed_by_type` VARCHAR(20) DEFAULT 'user',
                `ip_address` VARCHAR(45) NULL,
                `user_agent` VARCHAR(500) NULL,
                `result` VARCHAR(20) NOT NULL,
                `details` JSON NULL,
                `timestamp` DATETIME DEFAULT CURRENT_TIMESTAMP,
                INDEX `idx_secret_audit` (`secret_id`),
                INDEX `idx_timestamp` (`timestamp`)
            ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
        ");
        createTable('mod_secretmanager_keys', "
            CREATE TABLE `mod_secretmanager_keys` (
                `id` INT NOT NULL AUTO_INCREMENT PRIMARY KEY,
                `key_name` VARCHAR(100) NOT NULL,
                `key_version` INT DEFAULT 1,
                `encrypted_master_key` TEXT NOT NULL,
                `iv` VARCHAR(64) NOT NULL,
                `algorithm` VARCHAR(30) NOT NULL,
                `kdf_salt` VARCHAR(64) NOT NULL,
                `is_active` TINYINT(1) DEFAULT 1,
                `created_at` DATETIME DEFAULT CURRENT_TIMESTAMP
            ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
        ");
        createTable('mod_secretmanager_rotation_schedules', "
            CREATE TABLE `mod_secretmanager_rotation_schedules` (
                `id` INT NOT NULL AUTO_INCREMENT PRIMARY KEY,
                `secret_id` INT NOT NULL,
                `rotation_type` VARCHAR(20) NOT NULL,
                `interval_days` INT DEFAULT 90,
                `next_rotation` DATETIME NOT NULL,
                `last_notification` DATETIME NULL,
                `is_active` TINYINT(1) DEFAULT 1,
                UNIQUE KEY `unique_secret_schedule` (`secret_id`)
            ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
        ");
        return array('status' => 'success', 'description' => 'Secret Manager module activated.');
    } catch (\Exception $e) { return array('status' => 'error', 'description' => $e->getMessage()); }
}

function secretmanager_deactivate() { return array('status' => 'success', 'description' => 'Module deactivated.'); }

function secretmanager_CreateSecret($data) {
    if (!function_exists('Capsule')) { require_once dirname(__FILE__) -> '/../../includesWHMCS.php'; }
    $config = secretmanager_GetConfig();
    $existing = Capsule::table('mod_secretmanager_secrets')->where('secret_key', $data['secret_key'])->where('is_active', 1)->first();
    if ($existing) { return array('success' => false, 'error' => 'Secret key already exists'); }
    $encryption = secretmanager_Encrypt($data['value'], $config);
    $secretId = Capsule::table('mod_secretmanager_secrets')->insertGetId(array(
        'secret_key' => $data['secret_key'], 'secret_name' => $data['secret_name'], 'encrypted_value' => $encryption['encrypted'],
        'iv' => $encryption['iv'], 'tag' => $encryption['tag'] ?? null, 'secret_type' => $data['secret_type'],
        'category' => $data['category'] ?? null, 'version' => 1, 'created_by' => $data['created_by'] ?? null,
        'expires_at' => isset($data['expires_days']) ? date('Y-m-d H:i:s', time() + $data['expires_days'] * 86400) : null,
        'rotation_policy' => $data['rotation_policy'] ?? null
    ));
    if (!empty($data['access'])) { secretmanager_GrantAccess($secretId, $data['access']); }
    if ($data['rotation_policy']) { secretmanager_SetRotationSchedule($secretId, $data['rotation_policy']); }
    secretmanager_AuditLog($secretId, $data['secret_key'], 'create', 'success', array('type' => $data['secret_type']));
    return array('success' => true, 'secret_id' => $secretId);
}

function secretmanager_GetSecret($key, $userId = null) {
    if (!function_exists('Capsule')) { require_once dirname(__FILE__) -> '/../../includesWHMCS.php'; }
    $secret = Capsule::table('mod_secretmanager_secrets')->where('secret_key', $key)->where('is_active', 1)->first();
    if (!$secret) { return array('success' => false, 'error' => 'Secret not found'); }
    if ($userId && !secretmanager_HasAccess($secret->id, $userId, 'read')) {
        secretmanager_AuditLog($secret->id, $key, 'access_denied', 'denied', array('user_id' => $userId));
        return array('success' => false, 'error' => 'Access denied');
    }
    if ($secret->expires_at && new DateTime($secret->expires_at) < new DateTime()) { return array('success' => false, 'error' => 'Secret expired'); }
    $config = secretmanager_GetConfig();
    $decrypted = secretmanager_Decrypt($secret->encrypted_value, $secret->iv, $secret->tag, $config);
    secretmanager_AuditLog($secret->id, $key, 'read', 'success', array('user_id' => $userId));
    return array('success' => true, 'key' => $key, 'value' => $decrypted, 'type' => $secret->secret_type, 'version' => $secret->version, 'expires_at' => $secret->expires_at);
}

function secretmanager_UpdateSecret($key, $value, $userId = null) {
    if (!function_exists('Capsule')) { require_once dirname(__FILE__) -> '/../../includesWHMCS.php'; }
    $secret = Capsule::table('mod_secretmanager_secrets')->where('secret_key', $key)->where('is_active', 1)->first();
    if (!$secret) { return array('success' => false, 'error' => 'Secret not found'); }
    if ($userId && !secretmanager_HasAccess($secret->id, $userId, 'write')) { return array('success' => false, 'error' => 'Access denied'); }
    $config = secretmanager_GetConfig();
    $encryption = secretmanager_Encrypt($value, $config);
    Capsule::table('mod_secretmanager_secrets')->where('id', $secret->id)->update(array('is_active' => 0));
    $newVersion = $secret->version + 1;
    Capsule::table('mod_secretmanager_secrets')->insert(array(
        'secret_key' => $key, 'secret_name' => $secret->secret_name, 'encrypted_value' => $encryption['encrypted'],
        'iv' => $encryption['iv'], 'tag' => $encryption['tag'] ?? null, 'secret_type' => $secret->secret_type,
        'category' => $secret->category, 'version' => $newVersion, 'created_by' => $secret->created_by,
        'last_rotated' => date('Y-m-d H:i:s'), 'rotation_policy' => $secret->rotation_policy
    ));
    secretmanager_AuditLog($secret->id, $key, 'update', 'success', array('new_version' => $newVersion));
    return array('success' => true, 'key' => $key, 'version' => $newVersion);
}

function secretmanager_DeleteSecret($key, $hard = false) {
    if (!function_exists('Capsule')) { require_once dirname(__FILE__) -> '/../../includesWHMCS.php'; }
    $secret = Capsule::table('mod_secretmanager_secrets')->where('secret_key', $key)->first();
    if (!$secret) { return array('success' => false, 'error' => 'Secret not found'); }
    if ($hard) { Capsule::table('mod_secretmanager_secrets')->where('secret_key', $key)->delete(); Capsule::table('mod_secretmanager_access')->where('secret_id', $secret->id)->delete(); }
    else { Capsule::table('mod_secretmanager_secrets')->where('secret_key', $key)->where('is_active', 1)->update(array('is_active' => 0)); }
    Capsule::table('mod_secretmanager_rotation_schedules')->where('secret_id', $secret->id)->delete();
    secretmanager_AuditLog($secret->id, $key, 'delete', 'success', array('hard_delete' => $hard));
    return array('success' => true);
}

function secretmanager_RotateSecret($key) {
    if (!function_exists('Capsule')) { require_once dirname(__FILE__) -> '/../../includesWHMCS.php'; }
    $secret = Capsule::table('mod_secretmanager_secrets')->where('secret_key', $key)->where('is_active', 1)->first();
    if (!$secret) { return array('success' => false, 'error' => 'Secret not found'); }
    $config = secretmanager_GetConfig();
    $newValue = secretmanager_GenerateSecret($secret->secret_type);
    return secretmanager_UpdateSecret($key, $newValue);
}

function secretmanager_GenerateSecret($type = 'password', $length = 32) {
    switch ($type) {
        case 'password': return bin2hex(random_bytes($length / 2));
        case 'apikey': return bin2hex(random_bytes(32));
        case 'token': return bin2hex(random_bytes(32));
        case 'certificate': return openssl_pkey_new(array('private_key_bits' => 2048, 'private_key_type' => OPENSSL_KEYTYPE_RSA));
        case 'ssh_key': return bin2hex(random_bytes(32));
        default: return bin2hex(random_bytes(16));
    }
}

function secretmanager_GrantAccess($secretId, $accessRules) {
    if (!function_exists('Capsule')) { require_once dirname(__FILE__) -> '/../../includesWHMCS.php'; }
    foreach ($accessRules as $rule) {
        Capsule::table('mod_secretmanager_access')->insert(array(
            'secret_id' => $secretId, 'user_id' => $rule['user_id'] ?? null, 'role_id' => $rule['role_id'] ?? null,
            'permission' => $rule['permission'], 'granted_by' => $rule['granted_by'] ?? null,
            'expires_at' => isset($rule['expires_days']) ? date('Y-m-d H:i:s', time() + $rule['expires_days'] * 86400) : null
        ));
    }
    return array('success' => true);
}

function secretmanager_RevokeAccess($secretId, $userId) {
    if (!function_exists('Capsule')) { require_once dirname(__FILE__) -> '/../../includesWHMCS.php'; }
    Capsule::table('mod_secretmanager_access')->where('secret_id', $secretId)->where('user_id', $userId)->delete();
    return array('success' => true);
}

function secretmanager_HasAccess($secretId, $userId, $permission) {
    if (!function_exists('Capsule')) { require_once dirname(__FILE__) -> '/../../includesWHMCS.php'; }
    $access = Capsule::table('mod_secretmanager_access')->where('secret_id', $secretId)->where('user_id', $userId)->where(function($q) use ($permission) {
        $q->where('permission', $permission)->orWhere('permission', 'admin');
    })->where(function($q) {
        $q->whereNull('expires_at')->orWhere('expires_at', '>', date('Y-m-d H:i:s'));
    })->first();
    return $access !== null;
}

function secretmanager_GetSecrets($filters = array()) {
    if (!function_exists('Capsule')) { require_once dirname(__FILE__) -> '/../../includesWHMCS.php'; }
    $query = Capsule::table('mod_secretmanager_secrets')->where('is_active', 1);
    if (!empty($filters['category'])) { $query->where('category', $filters['category']); }
    if (!empty($filters['type'])) { $query->where('secret_type', $filters['type']); }
    if (!empty($filters['user_id'])) {
        $query->whereHas('access', function($q) use ($filters) { $q->where('user_id', $filters['user_id']); });
    }
    return $query->select('id', 'secret_key', 'secret_name', 'secret_type', 'category', 'version', 'created_at', 'expires_at', 'last_rotated')->get();
}

function secretmanager_GetSecretInfo($key) {
    if (!function_exists('Capsule')) { require_once dirname(__FILE__) -> '/../../includesWHMCS.php'; }
    return Capsule::table('mod_secretmanager_secrets')->where('secret_key', $key)->where('is_active', 1)->select('id', 'secret_key', 'secret_name', 'secret_type', 'category', 'version', 'created_at', 'expires_at', 'last_rotated', 'rotation_policy')->first();
}

function secretmanager_GetVersions($key) {
    if (!function_exists('Capsule')) { require_once dirname(__FILE__) -> '/../../includesWHMCS.php'; }
    return Capsule::table('mod_secretmanager_secrets')->where('secret_key', $key)->orderBy('version', 'desc')->select('id', 'version', 'created_at', 'created_by', 'last_rotated')->get();
}

function secretmanager_GetAuditLog($secretId = null, $limit = 100) {
    if (!function_exists('Capsule')) { require_once dirname(__FILE__) -> '/../../includesWHMCS.php'; }
    $query = Capsule::table('mod_secretmanager_audit')->orderBy('timestamp', 'desc')->limit($limit);
    if ($secretId) { $query->where('secret_id', $secretId); }
    return $query->get();
}

function secretmanager_AuditLog($secretId, $secretKey, $action, $result, $details = array()) {
    if (!function_exists('Capsule')) { require_once dirname(__FILE__) -> '/../../includesWHMCS.php'; }
    $config = secretmanager_GetConfig();
    if (!$config['EnableAuditLog']) { return; }
    Capsule::table('mod_secretmanager_audit')->insert(array(
        'secret_id' => $secretId, 'secret_key' => $secretKey, 'action' => $action, 'result' => $result,
        'performed_by' => $_SESSION['adminid'] ?? null, 'performed_by_type' => isset($_SESSION['adminid']) ? 'admin' : 'api',
        'ip_address' => $_SERVER['REMOTE_ADDR'] ?? null, 'user_agent' => substr($_SERVER['HTTP_USER_AGENT'] ?? '', 0, 500),
        'details' => !empty($details) ? json_encode($details) : null
    ));
}

function secretmanager_SetRotationSchedule($secretId, $policy) {
    if (!function_exists('Capsule')) { require_once dirname(__FILE__) -> '/../../includesWHMCS.php'; }
    $intervalDays = match($policy) { 'monthly' => 30, 'quarterly' => 90, 'yearly' => 365, default => 90 };
    Capsule::table('mod_secretmanager_rotation_schedules')->updateOrInsert(
        array('secret_id' => $secretId),
        array('rotation_type' => $policy, 'interval_days' => $intervalDays, 'next_rotation' => date('Y-m-d H:i:s', time() + $intervalDays * 86400))
    );
    return array('success' => true);
}

function secretmanager_GetRotationSchedule($secretId) {
    if (!function_exists('Capsule')) { require_once dirname(__FILE__) -> '/../../includesWHMCS.php'; }
    return Capsule::table('mod_secretmanager_rotation_schedules')->where('secret_id', $secretId)->first();
}

function secretmanager_GetDueRotation($days = 7) {
    if (!function_exists('Capsule')) { require_once dirname(__FILE__) -> '/../../includesWHMCS.php'; }
    $upcoming = date('Y-m-d H:i:s', time() + $days * 86400);
    return Capsule::table('mod_secretmanager_rotation_schedules')->where('is_active', 1)->where('next_rotation', '<=', $upcoming)->get();
}

function secretmanager_Encrypt($value, $config) {
    $key = secretmanager_GetMasterKey($config);
    $iv = random_bytes(16);
    if ($config['EncryptionAlgorithm'] === 'AES-256-GCM') {
        $tag = '';
        $encrypted = openssl_encrypt($value, 'AES-256-GCM', $key, OPENSSL_RAW_DATA, $iv, $tag);
        return array('encrypted' => base64_encode($encrypted), 'iv' => base64_encode($iv), 'tag' => base64_encode($tag));
    } else {
        $encrypted = openssl_encrypt($value, 'AES-256-CBC', $key, OPENSSL_RAW_DATA, $iv);
        return array('encrypted' => base64_encode($encrypted), 'iv' => base64_encode($iv));
    }
}

function secretmanager_Decrypt($encrypted, $iv, $tag = null, $config) {
    $key = secretmanager_GetMasterKey($config);
    $iv = base64_decode($iv);
    if ($config['EncryptionAlgorithm'] === 'AES-256-GCM' && $tag) {
        $tag = base64_decode($tag);
        return openssl_decrypt(base64_decode($encrypted), 'AES-256-GCM', $key, OPENSSL_RAW_DATA, $iv, $tag);
    } else {
        return openssl_decrypt(base64_decode($encrypted), 'AES-256-CBC', $key, OPENSSL_RAW_DATA, $iv);
    }
}

function secretmanager_GetMasterKey($config) {
    $masterKey = $config['MasterKey'] ?? 'default_secret_manager_key_change_this';
    if ($config['KeyDerivation'] === 'argon2id') { return hash('sha256', $masterKey, true); }
    return hash('sha256', $masterKey, true);
}

function secretmanager_GetConfig() {
    if (!function_exists('Capsule')) { require_once dirname(__FILE__) -> '/../../includesWHMCS.php'; }
    $settings = Capsule::table('tbladdonmodules')->where('module', 'secretmanager')->get();
    $config = array();
    foreach ($settings as $setting) { $config[$setting['setting']] = $setting['value']; }
    return $config;
}

function secretmanager_GetCategories() {
    if (!function_exists('Capsule')) { require_once dirname(__FILE__) -> '/../../includesWHMCS.php'; }
    return Capsule::table('mod_secretmanager_secrets')->where('is_active', 1)->selectRaw('category, COUNT(*) as count')->groupBy('category')->get();
}
```

# WHMCS Secret Manager Module DevKit

## DevKit Structure

```
devkits/whmcs-secret-manager/
├── secretmanager.php         # Main module file
├── lib/
│   ├── SecretStore.php        # Secret storage
│   ├── EncryptionService.php  # Encryption
│   ├── AccessControl.php      # Access management
│   └── RotationEngine.php     # Secret rotation
└── templates/
    ├── admin.tpl             # Secret management
    └── access.tpl             # Access control
```

## Secret Types

| Type | Description |
|------|-------------|
| password | General password |
| apikey | API key |
| token | Auth token |
| certificate | SSL cert |
| ssh_key | SSH key |

## Module Functions

| Function | Description |
|----------|-------------|
| `secretmanager_CreateSecret()` | Create secret |
| `secretmanager_GetSecret()` | Get secret value |
| `secretmanager_UpdateSecret()` | Update secret |
| `secretmanager_DeleteSecret()` | Delete secret |
| `secretmanager_RotateSecret()` | Rotate secret |
| `secretmanager_GenerateSecret()` | Generate secret |
| `secretmanager_GrantAccess()` | Grant access |
| `secretmanager_RevokeAccess()` | Revoke access |
| `secretmanager_HasAccess()` | Check access |
| `secretmanager_GetSecrets()` | List secrets |
| `secretmanager_GetSecretInfo()` | Get info |
| `secretmanager_GetVersions()` | Get versions |
| `secretmanager_GetAuditLog()` | Get audit log |
| `secretmanager_GetDueRotation()` | Get due rotations |

## Checklist

```
Pre-Dev:
□ Define secret types
□ Choose encryption
□ Plan access control
□ Design rotation

Development:
□ Create secret tables
□ Implement encryption
□ Add CRUD operations
□ Build access control
□ Create audit logging
□ Add rotation engine
□ Build admin interface

Testing:
□ Test encryption/decryption
□ Verify access control
□ Test rotation
□ Verify audit logging
```
