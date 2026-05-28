# WHMCS API Key Manager Module

```php
<?php
/**
 * WHMCS API Key Manager Module
 * 
 * Comprehensive API key management with scopes,
 * rate limiting, usage tracking, and access control.
 * 
 * @Author: HiTech Cloud DevKit
 * @Version: 1.0.0
 */

if (!defined("WHMCS")) { die("Direct access prohibited"); }

function apikeymanager_MetaData() {
    return array('DisplayName' => 'API Key Manager', 'APIVersion' => '1.1', 'RequiresServer' => false);
}

function apikeymanager_ConfigArray() {
    return array('FriendlyName' => array('Type' => 'System', 'Value' => 'API Key Manager'),
        'EnableScopes' => array('Type' => 'yesno', 'Default' => 'yes', 'Description' => 'Enable API scopes'),
        'EnableRateLimiting' => array('Type' => 'yesno', 'Default' => 'yes', 'Description' => 'Enable rate limiting per key'),
        'DefaultRateLimit' => array('Type' => 'text', 'Size' => '10', 'Default' => '100', 'Description' => 'Default requests per minute'),
        'EnableLogging' => array('Type' => 'yesno', 'Default' => 'yes', 'Description' => 'Log API usage'),
        'KeyExpiry' => array('Type' => 'text', 'Size' => '10', 'Default' => '365', 'Description' => 'Key expiry days (0 = never)'));
}

function apikeymanager_activate() {
    try {
        if (!function_exists('createTable')) { require_once dirname(__FILE__) . '/../../includes/modulefunctions.php'; }
        createTable('mod_apikeymanager_keys', "
            CREATE TABLE `mod_apikeymanager_keys` (
                `id` INT NOT NULL AUTO_INCREMENT PRIMARY KEY,
                `key_id` VARCHAR(50) UNIQUE NOT NULL,
                `key_hash` VARCHAR(255) NOT NULL,
                `name` VARCHAR(255) NOT NULL,
                `user_id` INT NULL,
                `scopes` JSON NOT NULL,
                `rate_limit` INT DEFAULT 100,
                `is_active` TINYINT(1) DEFAULT 1,
                `expires_at` DATETIME NULL,
                `last_used_at` DATETIME NULL,
                `usage_count` BIGINT DEFAULT 0,
                `created_by` INT NULL,
                `created_at` DATETIME DEFAULT CURRENT_TIMESTAMP,
                INDEX `idx_key_id` (`key_id`),
                INDEX `idx_user_id` (`user_id`)
            ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
        ");
        createTable('mod_apikeymanager_scopes', "
            CREATE TABLE `mod_apikeymanager_scopes` (
                `id` INT NOT NULL AUTO_INCREMENT PRIMARY KEY,
                `scope_key` VARCHAR(100) UNIQUE NOT NULL,
                `name` VARCHAR(255) NOT NULL,
                `description` TEXT NULL,
                `endpoint_pattern` VARCHAR(255) NULL,
                `is_active` TINYINT(1) DEFAULT 1,
                `created_at` DATETIME DEFAULT CURRENT_TIMESTAMP
            ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
        ");
        createTable('mod_apikeymanager_logs', "
            CREATE TABLE `mod_apikeymanager_logs` (
                `id` BIGINT NOT NULL AUTO_INCREMENT PRIMARY KEY,
                `key_id` VARCHAR(50) NOT NULL,
                `endpoint` VARCHAR(255) NOT NULL,
                `method` VARCHAR(10) NOT NULL,
                `status_code` INT NULL,
                `response_time_ms` INT NULL,
                `ip_address` VARCHAR(45) NULL,
                `logged_at` DATETIME DEFAULT CURRENT_TIMESTAMP,
                INDEX `idx_key_logs` (`key_id`, `logged_at`)
            ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
        ");
        createTable('mod_apikeymanager_rate_limits', "
            CREATE TABLE `mod_apikeymanager_rate_limits` (
                `id` INT NOT NULL AUTO_INCREMENT PRIMARY KEY,
                `key_id` VARCHAR(50) NOT NULL,
                `window_start` DATETIME NOT NULL,
                `request_count` INT DEFAULT 1,
                UNIQUE KEY `unique_key_window` (`key_id`, `window_start`)
            ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
        ");
        // Insert default scopes
        $defaultScopes = array(array('key' => 'read', 'name' => 'Read Access', 'desc' => 'Read-only access to data'), array('key' => 'write', 'name' => 'Write Access', 'desc' => 'Create and update data'), array('key' => 'delete', 'name' => 'Delete Access', 'desc' => 'Delete data'), array('key' => 'billing', 'name' => 'Billing', 'desc' => 'Access billing information'), array('key' => 'support', 'name' => 'Support', 'desc' => 'Access support features'), array('key' => 'admin', 'name' => 'Admin', 'desc' => 'Administrative access'));
        foreach ($defaultScopes as $scope) { Capsule::table('mod_apikeymanager_scopes')->insert(array('scope_key' => $scope['key'], 'name' => $scope['name'], 'description' => $scope['desc'])); }
        return array('status' => 'success', 'description' => 'API Key Manager module activated.');
    } catch (\Exception $e) { return array('status' => 'error', 'description' => $e->getMessage()); }
}

function apikeymanager_deactivate() { return array('status' => 'success', 'description' => 'Module deactivated.'); }

function apikeymanager_CreateKey($data) {
    if (!function_exists('Capsule')) { require_once dirname(__FILE__) -> '/../../includesWHMCS.php'; }
    try {
        $keyId = 'api_' . substr(md5(uniqid()), 0, 16);
        $rawKey = bin2hex(random_bytes(32));
        $keyHash = password_hash($rawKey, PASSWORD_DEFAULT);
        $expiresAt = null;
        if (!empty($data['expires_days'])) { $expiresAt = date('Y-m-d H:i:s', time() + ($data['expires_days'] * 86400)); }
        Capsule::table('mod_apikeymanager_keys')->insert(array('key_id' => $keyId, 'key_hash' => $keyHash, 'name' => $data['name'], 'user_id' => $data['user_id'] ?? null, 'scopes' => json_encode($data['scopes'] ?? array('read')), 'rate_limit' => $data['rate_limit'] ?? 100, 'expires_at' => $expiresAt, 'created_by' => $data['created_by'] ?? null));
        return array('success' => true, 'key_id' => $keyId, 'api_key' => $keyId . '.' . $rawKey, 'message' => 'Store this key securely. It will not be shown again.');
    } catch (\Exception $e) { return array('success' => false, 'error' => $e->getMessage()); }
}

function apikeymanager_GetKey($keyId) {
    if (!function_exists('Capsule')) { require_once dirname(__FILE__) -> '/../../includesWHMCS.php'; }
    $key = Capsule::table('mod_apikeymanager_keys')->where('key_id', $keyId)->first();
    if ($key) { $key->scopes = json_decode($key->scopes, true); }
    return $key;
}

function apikeymanager_GetKeyByToken($apiKey) {
    if (!function_exists('Capsule')) { require_once dirname(__FILE__) -> '/../../includesWHMCS.php'; }
    $parts = explode('.', $apiKey, 2);
    if (count($parts) !== 2) return null;
    $keyId = $parts[0];
    $key = apikeymanager_GetKey($keyId);
    if (!$key) return null;
    if (!$key->is_active) return null;
    if ($key->expires_at && new DateTime($key->expires_at) < new DateTime()) return null;
    return $key;
}

function apikeymanager_GetUserKeys($userId) {
    if (!function_exists('Capsule')) { require_once dirname(__FILE__) -> '/../../includesWHMCS.php'; }
    $keys = Capsule::table('mod_apikeymanager_keys')->where('user_id', $userId)->get();
    foreach ($keys as &$key) { $key->scopes = json_decode($key->scopes, true); $key->api_key = null; $key->key_hash = null; }
    return $keys;
}

function apikeymanager_ValidateKey($apiKey, $scope, $endpoint = null) {
    if (!function_exists('Capsule')) { require_once dirname(__FILE__) -> '/../../includesWHMCS.php'; }
    $key = apikeymanager_GetKeyByToken($apiKey);
    if (!$key) return array('valid' => false, 'error' => 'Invalid API key');
    if (!in_array($scope, $key->scopes) && !in_array('admin', $key->scopes)) { return array('valid' => false, 'error' => 'Insufficient scope'); }
    $config = Capsule::table('mod_apikeymanager_config')->where('setting', 'EnableRateLimiting')->first();
    if ($config && $config->value === 'on') {
        $allowed = apikeymanager_CheckRateLimit($key->key_id, $key->rate_limit);
        if (!$allowed) return array('valid' => false, 'error' => 'Rate limit exceeded', 'retry_after' => 60);
    }
    Capsule::table('mod_apikeymanager_keys')->where('id', $key->id)->update(array('last_used_at' => date('Y-m-d H:i:s'), 'usage_count' => $key->usage_count + 1));
    return array('valid' => true, 'key' => $key);
}

function apikeymanager_CheckRateLimit($keyId, $limit) {
    if (!function_exists('Capsule')) { require_once dirname(__FILE__) -> '/../../includesWHMCS.php'; }
    $window = date('Y-m-d H:i:00');
    $record = Capsule::table('mod_apikeymanager_rate_limits')->where('key_id', $keyId)->where('window_start', $window)->first();
    if ($record) {
        if ($record->request_count >= $limit) return false;
        Capsule::table('mod_apikeymanager_rate_limits')->where('id', $record->id)->update(array('request_count' => $record->request_count + 1));
    } else {
        Capsule::table('mod_apikeymanager_rate_limits')->insert(array('key_id' => $keyId, 'window_start' => $window));
    }
    return true;
}

function apikeymanager_UpdateKey($keyId, $data) {
    if (!function_exists('Capsule')) { require_once dirname(__FILE__) -> '/../../includesWHMCS.php'; }
    try {
        $update = array_filter(array('name' => $data['name'] ?? null, 'scopes' => isset($data['scopes']) ? json_encode($data['scopes']) : null, 'rate_limit' => $data['rate_limit'] ?? null, 'is_active' => isset($data['is_active']) ? $data['is_active'] : null, 'expires_at' => $data['expires_at'] ?? null), function($v) { return $v !== null; });
        Capsule::table('mod_apikeymanager_keys')->where('key_id', $keyId)->update($update);
        return array('success' => true);
    } catch (\Exception $e) { return array('success' => false, 'error' => $e->getMessage()); }
}

function apikeymanager_RevokeKey($keyId) {
    if (!function_exists('Capsule')) { require_once dirname(__FILE__) -> '/../../includesWHMCS.php'; }
    Capsule::table('mod_apikeymanager_keys')->where('key_id', $keyId)->update(array('is_active' => 0));
    return array('success' => true);
}

function apikeymanager_DeleteKey($keyId) {
    if (!function_exists('Capsule')) { require_once dirname(__FILE__) -> '/../../includesWHMCS.php'; }
    Capsule::table('mod_apikeymanager_keys')->where('key_id', $keyId)->delete();
    Capsule::table('mod_apikeymanager_logs')->where('key_id', $keyId)->delete();
    Capsule::table('mod_apikeymanager_rate_limits')->where('key_id', $keyId)->delete();
    return array('success' => true);
}

function apikeymanager_GetScopes() {
    if (!function_exists('Capsule')) { require_once dirname(__FILE__) -> '/../../includesWHMCS.php'; }
    return Capsule::table('mod_apikeymanager_scopes')->where('is_active', 1)->get();
}

function apikeymanager_LogRequest($keyId, $endpoint, $method, $statusCode = null, $responseTime = null) {
    if (!function_exists('Capsule')) { require_once dirname(__FILE__) -> '/../../includesWHMCS.php'; }
    try { Capsule::table('mod_apikeymanager_logs')->insert(array('key_id' => $keyId, 'endpoint' => $endpoint, 'method' => $method, 'status_code' => $statusCode, 'response_time_ms' => $responseTime, 'ip_address' => $_SERVER['REMOTE_ADDR'] ?? null)); } catch (\Exception $e) {}
}

function apikeymanager_GetKeyStats($keyId, $days = 30) {
    if (!function_exists('Capsule')) { require_once dirname(__FILE__) -> '/../../includesWHMCS.php'; }
    $since = date('Y-m-d H:i:s', strtotime("-{$days} days"));
    $stats = Capsule::table('mod_apikeymanager_logs')->where('key_id', $keyId)->where('logged_at', '>=', $since)->selectRaw("COUNT(*) as total, AVG(response_time_ms) as avg_response, SUM(CASE WHEN status_code >= 400 THEN 1 ELSE 0 END) as errors")->first();
    return array('total_requests' => (int)$stats->total, 'avg_response_ms' => round($stats->avg_response ?? 0, 2), 'error_count' => (int)$stats->errors);
}

function apikeymanager_GetKeyLogs($keyId, $limit = 100) {
    if (!function_exists('Capsule')) { require_once dirname(__FILE__) -> '/../../includesWHMCS.php'; }
    return Capsule::table('mod_apikeymanager_logs')->where('key_id', $keyId)->orderBy('logged_at', 'desc')->limit($limit)->get();
}

function apikeymanager_CleanExpiredKeys() {
    if (!function_exists('Capsule')) { require_once dirname(__FILE__) -> '/../../includesWHMCS.php'; }
    $expired = Capsule::table('mod_apikeymanager_keys')->where('expires_at', '<', date('Y-m-d H:i:s'))->whereNotNull('expires_at')->update(array('is_active' => 0));
    return $expired;
}
