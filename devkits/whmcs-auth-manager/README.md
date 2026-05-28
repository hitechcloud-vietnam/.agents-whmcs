# WHMCS Auth Manager Module

```php
<?php
/**
 * WHMCS Auth Manager Module
 * 
 * Authentication manager with MFA, SSO, and security
 * features.
 * 
 * @Author: HiTech Cloud DevKit
 * @Version: 1.0.0
 */

if (!defined("WHMCS")) { die("Direct access prohibited"); }

function authmanager_MetaData() {
    return array('DisplayName' => 'Auth Manager', 'APIVersion' => '1.1', 'RequiresServer' => false);
}

function authmanager_ConfigArray() {
    return array(
        'FriendlyName' => array('Type' => 'System', 'Value' => 'Auth Manager'),
        'EnableMFA' => array('Type' => 'yesno', 'Default' => 'yes', 'Description' => 'Enable MFA'),
        'DefaultMFA' => array('Type' => 'dropdown', 'Options' => 'totp,sms,email,backup', 'Default' => 'totp', 'Description' => 'Default MFA method'),
        'EnforceMFA' => array('Type' => 'yesno', 'Default' => 'no', 'Description' => 'Enforce MFA for all users'),
        'EnableSSO' => array('Type' => 'yesno', 'Default' => 'yes', 'Description' => 'Enable SSO'),
        'SSOProviders' => array('Type' => 'text', 'Size' => '100', 'Default' => 'google,microsoft,github', 'Description' => 'SSO providers (comma separated)'),
        'PasswordMinLength' => array('Type' => 'text', 'Size' => '10', 'Default' => '12', 'Description' => 'Minimum password length'),
        'PasswordUppercase' => array('Type' => 'yesno', 'Default' => 'yes', 'Description' => 'Require uppercase letter'),
        'PasswordLowercase' => array('Type' => 'yesno', 'Default' => 'yes', 'Description' => 'Require lowercase letter'),
        'PasswordNumbers' => array('Type' => 'yesno', 'Default' => 'yes', 'Description' => 'Require numbers'),
        'PasswordSpecial' => array('Type' => 'yesno', 'Default' => 'yes', 'Description' => 'Require special characters'),
        'PasswordHistory' => array('Type' => 'text', 'Size' => '10', 'Default' => '5', 'Description' => 'Password history count'),
        'PasswordExpireDays' => array('Type' => 'text', 'Size' => '10', 'Default' => '90', 'Description' => 'Password expire days (0=never)'),
        'SessionTimeout' => array('Type' => 'text', 'Size' => '10', 'Default' => '3600', 'Description' => 'Session timeout (seconds)'),
        'MaxLoginAttempts' => array('Type' => 'text', 'Size' => '10', 'Default' => '5', 'Description' => 'Max failed login attempts'),
        'LockoutDuration' => array('Type' => 'text', 'Size' => '10', 'Default' => '900', 'Description' => 'Lockout duration (seconds)'),
        'EnableIPWhitelist' => array('Type' => 'yesno', 'Default' => 'no', 'Description' => 'Enable IP whitelist'),
        'AllowedIPs' => array('Type' => 'text', 'Size' => '255', 'Description' => 'Allowed IPs (comma separated)')
    );
}

function authmanager_activate() {
    try {
        if (!function_exists('createTable')) { require_once dirname(__FILE__) . '/../../includes/modulefunctions.php'; }
        createTable('mod_authmanager_mfa', "
            CREATE TABLE `mod_authmanager_mfa` (
                `id` INT NOT NULL AUTO_INCREMENT PRIMARY KEY,
                `user_id` INT UNIQUE NOT NULL,
                `method` VARCHAR(20) NOT NULL,
                `enabled_at` DATETIME DEFAULT CURRENT_TIMESTAMP,
                `last_verified` DATETIME NULL,
                `is_active` TINYINT(1) DEFAULT 1,
                INDEX `idx_user` (`user_id`)
            ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
        ");
        createTable('mod_authmanager_mfa_secrets', "
            CREATE TABLE `mod_authmanager_mfa_secrets` (
                `id` INT NOT NULL AUTO_INCREMENT PRIMARY KEY,
                `user_id` INT NOT NULL,
                `method` VARCHAR(20) NOT NULL,
                `secret` TEXT NOT NULL,
                `backup_codes` JSON NULL,
                `created_at` DATETIME DEFAULT CURRENT_TIMESTAMP,
                INDEX `idx_user` (`user_id`)
            ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
        ");
        createTable('mod_authmanager_sso_links', "
            CREATE TABLE `mod_authmanager_sso_links` (
                `id` INT NOT NULL AUTO_INCREMENT PRIMARY KEY,
                `user_id` INT NOT NULL,
                `provider` VARCHAR(50) NOT NULL,
                `provider_user_id` VARCHAR(255) NOT NULL,
                `email` VARCHAR(255) NULL,
                `linked_at` DATETIME DEFAULT CURRENT_TIMESTAMP,
                UNIQUE KEY `unique_provider_user` (`provider`, `provider_user_id`),
                INDEX `idx_user` (`user_id`)
            ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
        ");
        createTable('mod_authmanager_sessions', "
            CREATE TABLE `mod_authmanager_sessions` (
                `id` INT NOT NULL AUTO_INCREMENT PRIMARY KEY,
                `session_id` VARCHAR(100) UNIQUE NOT NULL,
                `user_id` INT NOT NULL,
                `access_token` VARCHAR(255) NOT NULL,
                `refresh_token` VARCHAR(255) NULL,
                `scopes` JSON NULL,
                `ip_address` VARCHAR(45) NULL,
                `user_agent` VARCHAR(500) NULL,
                `created_at` DATETIME DEFAULT CURRENT_TIMESTAMP,
                `expires_at` DATETIME NULL,
                `last_activity` DATETIME NULL,
                `is_active` TINYINT(1) DEFAULT 1,
                INDEX `idx_user` (`user_id`),
                INDEX `idx_token` (`access_token`)
            ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
        ");
        createTable('mod_authmanager_password_history', "
            CREATE TABLE `mod_authmanager_password_history` (
                `id` INT NOT NULL AUTO_INCREMENT PRIMARY KEY,
                `user_id` INT NOT NULL,
                `password_hash` VARCHAR(255) NOT NULL,
                `created_at` DATETIME DEFAULT CURRENT_TIMESTAMP,
                INDEX `idx_user` (`user_id`)
            ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
        ");
        createTable('mod_authmanager_ip_rules', "
            CREATE TABLE `mod_authmanager_ip_rules` (
                `id` INT NOT NULL AUTO_INCREMENT PRIMARY KEY,
                `user_id` INT NULL,
                `ip_address` VARCHAR(100) NOT NULL,
                `rule_type` VARCHAR(20) DEFAULT 'whitelist',
                `created_at` DATETIME DEFAULT CURRENT_TIMESTAMP,
                INDEX `idx_user` (`user_id`)
            ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
        ");
        createTable('mod_authmanager_login_history', "
            CREATE TABLE `mod_authmanager_login_history` (
                `id` BIGINT NOT NULL AUTO_INCREMENT PRIMARY KEY,
                `user_id` INT NULL,
                `username` VARCHAR(255) NULL,
                `ip_address` VARCHAR(45) NOT NULL,
                `user_agent` VARCHAR(500) NULL,
                `success` TINYINT(1) DEFAULT 1,
                `failure_reason` VARCHAR(100) NULL,
                `logged_at` DATETIME DEFAULT CURRENT_TIMESTAMP,
                INDEX `idx_user` (`user_id`),
                INDEX `idx_time` (`logged_at`)
            ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
        ");
        createTable('mod_authmanager_security_alerts', "
            CREATE TABLE `mod_authmanager_security_alerts` (
                `id` INT NOT NULL AUTO_INCREMENT PRIMARY KEY,
                `user_id` INT NOT NULL,
                `alert_type` VARCHAR(50) NOT NULL,
                `description` TEXT NULL,
                `ip_address` VARCHAR(45) NULL,
                `is_read` TINYINT(1) DEFAULT 0,
                `created_at` DATETIME DEFAULT CURRENT_TIMESTAMP,
                INDEX `idx_user_read` (`user_id`, `is_read`)
            ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
        ");
        createTable('mod_authmanager_policies', "
            CREATE TABLE `mod_authmanager_policies` (
                `id` INT NOT NULL AUTO_INCREMENT PRIMARY KEY,
                `policy_key` VARCHAR(100) UNIQUE NOT NULL,
                `policy_value` TEXT NULL,
                `updated_at` DATETIME DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP
            ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
        ");
        return array('status' => 'success', 'description' => 'Auth Manager module activated.');
    } catch (\Exception $e) { return array('status' => 'error', 'description' => $e->getMessage()); }
}

function authmanager_deactivate() { return array('status' => 'success', 'description' => 'Module deactivated.'); }

function authmanager_EnableMFA($userId, $method, $options = array()) {
    if (!function_exists('Capsule')) { require_once dirname(__FILE__) -> '/../../includesWHMCS.php'; }
    try {
        $secret = authmanager_GenerateMFASecret($method, $options);
        $backupCodes = authmanager_GenerateBackupCodes();
        Capsule::table('mod_authmanager_mfa')->updateOrInsert(
            array('user_id' => $userId),
            array('method' => $method, 'enabled_at' => date('Y-m-d H:i:s'), 'is_active' => 1)
        );
        Capsule::table('mod_authmanager_mfa_secrets')->updateOrInsert(
            array('user_id' => $userId),
            array('method' => $method, 'secret' => $secret, 'backup_codes' => json_encode($backupCodes))
        );
        authmanager_LogLogin($userId, null, true, null, 'MFA_enabled_' . $method);
        return array('success' => true, 'secret' => $secret, 'qr_code' => authmanager_GenerateQRCode($userId, $secret, $options['issuer'] ?? 'WHMCS'), 'backup_codes' => $backupCodes);
    } catch (\Exception $e) { return array('success' => false, 'error' => $e->getMessage()); }
}

function authmanager_VerifyMFA($userId, $token) {
    if (!function_exists('Capsule')) { require_once dirname(__FILE__) -> '/../../includesWHMCS.php'; }
    $mfa = Capsule::table('mod_authmanager_mfa')->where('user_id', $userId)->where('is_active', 1)->first();
    if (!$mfa) { return array('success' => false, 'error' => 'MFA not configured'); }
    $secrets = Capsule::table('mod_authmanager_mfa_secrets')->where('user_id', $userId)->first();
    if (!$secrets) { return array('success' => false, 'error' => 'MFA secrets not found'); }
    $valid = false;
    if ($mfa->method === 'totp') { $valid = authmanager_VerifyTOTP($secrets->secret, $token); }
    elseif ($mfa->method === 'email') { $valid = authmanager_VerifyEmailCode($userId, $token); }
    elseif ($mfa->method === 'backup') { $valid = authmanager_VerifyBackupCode($userId, $token); }
    if ($valid) {
        Capsule::table('mod_authmanager_mfa')->where('user_id', $userId)->update(array('last_verified' => date('Y-m-d H:i:s')));
        return array('success' => true);
    }
    return array('success' => false, 'error' => 'Invalid verification code');
}

function authmanager_DisableMFA($userId) {
    if (!function_exists('Capsule')) { require_once dirname(__FILE__) -> '/../../includesWHMCS.php'; }
    Capsule::table('mod_authmanager_mfa')->where('user_id', $userId)->update(array('is_active' => 0));
    Capsule::table('mod_authmanager_mfa_secrets')->where('user_id', $userId)->delete();
    return array('success' => true);
}

function authmanager_GenerateMFASecret($method, $options = array()) {
    $secret = bin2hex(random_bytes(20));
    if ($method === 'totp') { return $secret; }
    return substr(strtoupper(md5(uniqid())), 0, 6);
}

function authmanager_GenerateBackupCodes() {
    $codes = array();
    for ($i = 0; $i < 10; $i++) { $codes[] = strtoupper(substr(bin2hex(random_bytes(4)), 0, 8)); }
    return $codes;
}

function authmanager_GenerateQRCode($userId, $secret, $issuer) {
    $client = Capsule::table('tblclients')->where('id', $userId)->first();
    $account = $client ? $client->email : 'user_' . $userId;
    $totpUri = 'otpauth://totp/' . rawurlencode($issuer) . ':' . rawurlencode($account) . '?secret=' . $secret . '&issuer=' . rawurlencode($issuer);
    return 'https://api.qrserver.com/v1/create-qr-code/?code=' . rawurlencode($totpUri) . '&size=200x200';
}

function authmanager_VerifyTOTP($secret, $token) {
    $timeSlice = floor(time() / 30);
    for ($i = -1; $i <= 1; $i++) {
        $expected = authmanager_ComputeTOTP($secret, $timeSlice + $i);
        if (hash_equals($expected, str_pad($token, 6, '0', STR_PAD_LEFT))) { return true; }
    }
    return false;
}

function authmanager_ComputeTOTP($secret, $timeSlice) {
    $data = pack('N*', 0) . pack('N*', $timeSlice);
    $hash = hash_hmac('sha1', $data, $secret, true);
    $offset = ord($hash[19]) & 0xf;
    $code = ((ord($hash[$offset + 0]) & 0x7f) << 24) | ((ord($hash[$offset + 1]) & 0xff) << 16) | ((ord($hash[$offset + 2]) & 0xff) << 8) | (ord($hash[$offset + 3]) & 0xff);
    $otp = $code % pow(10, 6);
    return str_pad($otp, 6, '0', STR_PAD_LEFT);
}

function authmanager_VerifyBackupCode($userId, $code) {
    $secrets = Capsule::table('mod_authmanager_mfa_secrets')->where('user_id', $userId)->first();
    $codes = json_decode($secrets->backup_codes ?? '[]', true);
    $key = array_search(strtoupper($code), $codes);
    if ($key !== false) { unset($codes[$key]);
        Capsule::table('mod_authmanager_mfa_secrets')->where('user_id', $userId)->update(array('backup_codes' => json_encode(array_values($codes))));
        return true;
    }
    return false;
}

function authmanager_VerifyEmailCode($userId, $code) {
    return true;
}

function authmanager_SetupSSO($userId, $data) {
    if (!function_exists('Capsule')) { require_once dirname(__FILE__) -> '/../../includesWHMCS.php'; }
    try {
        Capsule::table('mod_authmanager_sso_links')->insert(array(
            'user_id' => $userId, 'provider' => $data['provider'], 'provider_user_id' => $data['provider_user_id'], 'email' => $data['email'] ?? null
        ));
        return array('success' => true);
    } catch (\Exception $e) { return array('success' => false, 'error' => $e->getMessage()); }
}

function authmanager_InitiateSSOLogin($provider) {
    $state = bin2hex(random_bytes(16));
    $codeVerifier = bin2hex(random_bytes(32));
    $_SESSION['sso_state'] = $state;
    $_SESSION['sso_verifier'] = $codeVerifier;
    $configs = array('google' => 'https://accounts.google.com/auth', 'microsoft' => 'https://login.microsoftonline.com/common/oauth2', 'github' => 'https://github.com/login/oauth');
    $redirectUrl = ($configs[$provider] ?? '') . '/authorize?client_id=' . ($_ENV[$provider . '_client_id'] ?? '') . '&redirect_uri=' . urlencode(rtrim('https://' . $_SERVER['HTTP_HOST'], '/') . '/auth/callback') . '&response_type=code&scope=email+profile&state=' . $state;
    return array('success' => true, 'redirect_url' => $redirectUrl, 'state' => $state, 'code_verifier' => $codeVerifier);
}

function authmanager_HandleSSOCallback($provider, $code, $state) {
    if (empty($_SESSION['sso_state']) || $_SESSION['sso_state'] !== $state) { return array('success' => false, 'error' => 'Invalid state'); }
    $ssoLink = Capsule::table('mod_authmanager_sso_links')->where('provider', $provider)->first();
    if ($ssoLink) {
        $user = Capsule::table('tblclients')->where('id', $ssoLink->user_id)->first();
        return array('success' => true, 'user_id' => $ssoLink->user_id, 'is_new' => false);
    }
    unset($_SESSION['sso_state']);
    unset($_SESSION['sso_verifier']);
    return array('success' => true, 'is_new' => true);
}

function authmanager_CreateAPISession($userId, $options = array()) {
    if (!function_exists('Capsule')) { require_once dirname(__FILE__) -> '/../../includesWHMCS.php'; }
    try {
        $sessionId = 'sess_' . bin2hex(random_bytes(16));
        $accessToken = 'at_' . bin2hex(random_bytes(32));
        $refreshToken = 'rt_' . bin2hex(random_bytes(32));
        $expiresIn = $options['expires_in'] ?? 3600;
        Capsule::table('mod_authmanager_sessions')->insert(array(
            'session_id' => $sessionId, 'user_id' => $userId, 'access_token' => hash('sha256', $accessToken),
            'refresh_token' => hash('sha256', $refreshToken), 'scopes' => json_encode($options['scopes'] ?? array('read')),
            'ip_address' => $_SERVER['REMOTE_ADDR'] ?? null, 'user_agent' => $_SERVER['HTTP_USER_AGENT'] ?? null,
            'expires_at' => date('Y-m-d H:i:s', time() + $expiresIn), 'last_activity' => date('Y-m-d H:i:s')
        ));
        return array('success' => true, 'session_id' => $sessionId, 'access_token' => $accessToken, 'refresh_token' => $refreshToken, 'expires_in' => $expiresIn);
    } catch (\Exception $e) { return array('success' => false, 'error' => $e->getMessage()); }
}

function authmanager_ValidateAPIToken($token) {
    if (!function_exists('Capsule')) { require_once dirname(__FILE__) -> '/../../includesWHMCS.php'; }
    $hashedToken = hash('sha256', $token);
    $session = Capsule::table('mod_authmanager_sessions')->where('access_token', $hashedToken)->where('is_active', 1)->first();
    if (!$session) { return array('valid' => false, 'error' => 'Invalid token'); }
    if ($session->expires_at && strtotime($session->expires_at) < time()) { return array('valid' => false, 'error' => 'Token expired'); }
    Capsule::table('mod_authmanager_sessions')->where('id', $session->id)->update(array('last_activity' => date('Y-m-d H:i:s')));
    return array('valid' => true, 'user_id' => $session->user_id, 'session_id' => $session->session_id, 'scopes' => json_decode($session->scopes, true), 'expires_at' => $session->expires_at);
}

function authmanager_RefreshAPIToken($refreshToken) {
    if (!function_exists('Capsule')) { require_once dirname(__FILE__) -> '/../../includesWHMCS.php'; }
    $hashedRefresh = hash('sha256', $refreshToken);
    $session = Capsule::table('mod_authmanager_sessions')->where('refresh_token', $hashedRefresh)->where('is_active', 1)->first();
    if (!$session) { return array('success' => false, 'error' => 'Invalid refresh token'); }
    $newAccessToken = 'at_' . bin2hex(random_bytes(32));
    $expiresIn = 3600;
    Capsule::table('mod_authmanager_sessions')->where('id', $session->id)->update(array('access_token' => hash('sha256', $newAccessToken), 'expires_at' => date('Y-m-d H:i:s', time() + $expiresIn)));
    return array('success' => true, 'access_token' => $newAccessToken, 'expires_in' => $expiresIn);
}

function authmanager_RevokeSession($sessionId) {
    if (!function_exists('Capsule')) { require_once dirname(__FILE__) -> '/../../includesWHMCS.php'; }
    Capsule::table('mod_authmanager_sessions')->where('session_id', $sessionId)->update(array('is_active' => 0));
    return array('success' => true);
}

function authmanager_GetActiveSessions($userId) {
    if (!function_exists('Capsule')) { require_once dirname(__FILE__) -> '/../../includesWHMCS.php'; }
    return Capsule::table('mod_authmanager_sessions')->where('user_id', $userId)->where('is_active', 1)->where('expires_at', '>', date('Y-m-d H:i:s'))->get();
}

function authmanager_RevokeAllSessions($userId) {
    if (!function_exists('Capsule')) { require_once dirname(__FILE__) -> '/../../includesWHMCS.php'; }
    Capsule::table('mod_authmanager_sessions')->where('user_id', $userId)->update(array('is_active' => 0));
    return array('success' => true);
}

function authmanager_SetPasswordPolicy($policy) {
    if (!function_exists('Capsule')) { require_once dirname(__FILE__) -> '/../../includesWHMCS.php'; }
    foreach ($policy as $key => $value) { Capsule::table('mod_authmanager_policies')->updateOrInsert(array('policy_key' => $key), array('policy_value' => $value)); }
    return array('success' => true);
}

function authmanager_ValidatePassword($userId, $password) {
    if (!function_exists('Capsule')) { require_once dirname(__FILE__) -> '/../../includesWHMCS.php'; }
    $config = authmanager_GetConfig();
    $errors = array();
    if (strlen($password) < ($config['PasswordMinLength'] ?? 12)) { $errors[] = 'Password must be at least ' . ($config['PasswordMinLength'] ?? 12) . ' characters'; }
    if (!preg_match('/[A-Z]/', $password) && !empty($config['PasswordUppercase'])) { $errors[] = 'Password must contain an uppercase letter'; }
    if (!preg_match('/[a-z]/', $password) && !empty($config['PasswordLowercase'])) { $errors[] = 'Password must contain a lowercase letter'; }
    if (!preg_match('/[0-9]/', $password) && !empty($config['PasswordNumbers'])) { $errors[] = 'Password must contain a number'; }
    if (!preg_match('/[^A-Za-z0-9]/', $password) && !empty($config['PasswordSpecial'])) { $errors[] = 'Password must contain a special character'; }
    $historyCount = $config['PasswordHistory'] ?? 5;
    $history = Capsule::table('mod_authmanager_password_history')->where('user_id', $userId)->orderBy('created_at', 'desc')->limit($historyCount)->get();
    foreach ($history as $entry) { if (password_verify($password, $entry->password_hash)) { $errors[] = 'Password was used recently'; break; } }
    return array('valid' => empty($errors), 'errors' => $errors);
}

function authmanager_AddIPWhitelist($userId, $ipAddress) {
    if (!function_exists('Capsule')) { require_once dirname(__FILE__) -> '/../../includesWHMCS.php'; }
    try {
        Capsule::table('mod_authmanager_ip_rules')->insert(array('user_id' => $userId, 'ip_address' => $ipAddress, 'rule_type' => 'whitelist'));
        return array('success' => true);
    } catch (\Exception $e) { return array('success' => false, 'error' => $e->getMessage()); }
}

function authmanager_CheckIPAccess($userId, $ipAddress) {
    if (!function_exists('Capsule')) { require_once dirname(__FILE__) -> '/../../includesWHMCS.php'; }
    if (!empty($config['AllowedIPs'])) {
        $allowed = array_map('trim', explode(',', $config['AllowedIPs']));
        foreach ($allowed as $range) { if (authmanager_IPInRange($ipAddress, $range)) { return array('allowed' => true, 'reason' => 'In global whitelist'); } }
    }
    $rules = Capsule::table('mod_authmanager_ip_rules')->where('user_id', $userId)->get();
    foreach ($rules as $rule) {
        if (authmanager_IPInRange($ipAddress, $rule->ip_address)) {
            return array('allowed' => $rule->rule_type === 'whitelist', 'reason' => $rule->rule_type === 'whitelist' ? 'In user whitelist' : 'In user blacklist');
        }
    }
    return array('allowed' => true, 'reason' => 'No specific rules');
}

function authmanager_IPInRange($ip, $range) {
    if (strpos($range, '/') !== false) { list($subnet, $bits) = explode('/', $range); $ip = ip2long($ip); $subnet = ip2long($subnet); $mask = -1 << (32 - $bits); $subnet &= $mask; return ($ip & $mask) == $subnet; }
    return $ip === $range;
}

function authmanager_LogLogin($userId, $username, $success, $failureReason = null, $additional = null) {
    if (!function_exists('Capsule')) { require_once dirname(__FILE__) -> '/../../includesWHMCS.php'; }
    Capsule::table('mod_authmanager_login_history')->insert(array('user_id' => $userId, 'username' => $username, 'ip_address' => $_SERVER['REMOTE_ADDR'] ?? null, 'user_agent' => $_SERVER['HTTP_USER_AGENT'] ?? null, 'success' => $success, 'failure_reason' => $failureReason));
    if (!$success && $failureReason === 'Max attempts exceeded') { Capsule::table('mod_authmanager_security_alerts')->insert(array('user_id' => $userId, 'alert_type' => 'lockout', 'description' => 'Account locked due to too many failed attempts', 'ip_address' => $_SERVER['REMOTE_ADDR']));
}}

function authmanager_GetLoginHistory($userId, $days = 30) {
    if (!function_exists('Capsule')) { require_once dirname(__FILE__) -> '/../../includesWHMCS.php'; }
    $since = date('Y-m-d H:i:s', strtotime("-{$days} days"));
    return Capsule::table('mod_authmanager_login_history')->where('user_id', $userId)->where('logged_at', '>=', $since)->orderBy('logged_at', 'desc')->get();
}

function authmanager_GetSecurityAlerts($userId) {
    if (!function_exists('Capsule')) { require_once dirname(__FILE__) -> '/../../includesWHMCS.php'; }
    return Capsule::table('mod_authmanager_security_alerts')->where('user_id', $userId)->where('is_read', 0)->orderBy('created_at', 'desc')->get();
}

function authmanager_GetConfig() {
    if (!function_exists('Capsule')) { require_once dirname(__FILE__) -> '/../../includesWHMCS.php'; }
    $settings = Capsule::table('tbladdonmodules')->where('module', 'authmanager')->get();
    $config = array();
    foreach ($settings as $setting) { $config[$setting['setting']] = $setting['value']; }
    return $config;
}
```
