# WHMCS Session Manager Module

```php
<?php
/**
 * WHMCS Session Manager Module
 * 
 * Session management with monitoring, analytics,
 * and security features.
 * 
 * @Author: HiTech Cloud DevKit
 * @Version: 1.0.0
 */

if (!defined("WHMCS")) { die("Direct access prohibited"); }

function sessionmanager_MetaData() {
    return array('DisplayName' => 'Session Manager', 'APIVersion' => '1.1', 'RequiresServer' => false);
}

function sessionmanager_ConfigArray() {
    return array(
        'FriendlyName' => array('Type' => 'System', 'Value' => 'Session Manager'),
        'StorageType' => array('Type' => 'dropdown', 'Options' => 'database,redis,file', 'Default' => 'database', 'Description' => 'Session storage method'),
        'SessionTimeout' => array('Type' => 'text', 'Size' => '10', 'Default' => '1800', 'Description' => 'Default timeout (seconds)'),
        'RememberMeDuration' => array('Type' => 'text', 'Size' => '10', 'Default' => '2592000', 'Description' => 'Remember me duration (seconds)'),
        'MaxConcurrentSessions' => array('Type' => 'text', 'Size' => '10', 'Default' => '5', 'Description' => 'Max concurrent sessions'),
        'EnableEncryption' => array('Type' => 'yesno', 'Default' => 'yes', 'Description' => 'Encrypt session data'),
        'EnableIPTracking' => array('Type' => 'yesno', 'Default' => 'yes', 'Description' => 'Track IP addresses'),
        'EnableDeviceTracking' => array('Type' => 'yesno', 'Default' => 'yes', 'Description' => 'Track devices'),
        'ForceLogoutOnPasswordChange' => array('Type' => 'yesno', 'Default' => 'yes', 'Description' => 'Logout on password change'),
        'IdleTimeout' => array('Type' => 'text', 'Size' => '10', 'Default' => '600', 'Description' => 'Idle timeout (seconds)'),
        'RedisHost' => array('Type' => 'text', 'Size' => '50', 'Default' => 'localhost', 'Description' => 'Redis hostname'),
        'RedisPort' => array('Type' => 'text', 'Size' => '10', 'Default' => '6379', 'Description' => 'Redis port'),
        'RedisPassword' => array('Type' => 'password', 'Description' => 'Redis password')
    );
}

function sessionmanager_activate() {
    try {
        if (!function_exists('createTable')) { require_once dirname(__FILE__) . '/../../includes/modulefunctions.php'; }
        createTable('mod_sessionmanager_sessions', "
            CREATE TABLE `mod_sessionmanager_sessions` (
                `id` INT NOT NULL AUTO_INCREMENT PRIMARY KEY,
                `session_id` VARCHAR(100) UNIQUE NOT NULL,
                `user_id` INT NOT NULL,
                `token` VARCHAR(255) NOT NULL,
                `ip_address` VARCHAR(45) NULL,
                `user_agent` VARCHAR(500) NULL,
                `device_id` VARCHAR(100) NULL,
                `device_name` VARCHAR(100) NULL,
                `device_type` VARCHAR(50) NULL,
                `location` VARCHAR(100) NULL,
                `last_activity` DATETIME DEFAULT CURRENT_TIMESTAMP,
                `created_at` DATETIME DEFAULT CURRENT_TIMESTAMP,
                `expires_at` DATETIME NOT NULL,
                `is_active` TINYINT(1) DEFAULT 1,
                `is_remembered` TINYINT(1) DEFAULT 0,
                INDEX `idx_user` (`user_id`),
                INDEX `idx_token` (`token`),
                INDEX `idx_expires` (`expires_at`)
            ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
        ");
        createTable('mod_sessionmanager_history', "
            CREATE TABLE `mod_sessionmanager_history` (
                `id` BIGINT NOT NULL AUTO_INCREMENT PRIMARY KEY,
                `user_id` INT NOT NULL,
                `session_id` VARCHAR(100) NULL,
                `event` VARCHAR(50) NOT NULL,
                `ip_address` VARCHAR(45) NULL,
                `user_agent` VARCHAR(500) NULL,
                `device_id` VARCHAR(100) NULL,
                `metadata` JSON NULL,
                `created_at` DATETIME DEFAULT CURRENT_TIMESTAMP,
                INDEX `idx_user_time` (`user_id`, `created_at`)
            ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
        ");
        createTable('mod_sessionmanager_devices', "
            CREATE TABLE `mod_sessionmanager_devices` (
                `id` INT NOT NULL AUTO_INCREMENT PRIMARY KEY,
                `user_id` INT NOT NULL,
                `device_id` VARCHAR(100) UNIQUE NOT NULL,
                `device_name` VARCHAR(100) NULL,
                `device_type` VARCHAR(50) NULL,
                `user_agent` VARCHAR(500) NULL,
                `last_ip` VARCHAR(45) NULL,
                `first_seen` DATETIME DEFAULT CURRENT_TIMESTAMP,
                `last_seen` DATETIME DEFAULT CURRENT_TIMESTAMP,
                `is_trusted` TINYINT(1) DEFAULT 0,
                `login_count` INT DEFAULT 1,
                INDEX `idx_user` (`user_id`)
            ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
        ");
        createTable('mod_sessionmanager_limits', "
            CREATE TABLE `mod_sessionmanager_limits` (
                `id` INT NOT NULL AUTO_INCREMENT PRIMARY KEY,
                `user_id` INT UNIQUE NOT NULL,
                `max_concurrent` INT DEFAULT 5,
                `created_at` DATETIME DEFAULT CURRENT_TIMESTAMP,
                UNIQUE KEY `unique_user` (`user_id`)
            ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
        ");
        createTable('mod_sessionmanager_sharing', "
            CREATE TABLE `mod_sessionmanager_sharing` (
                `id` INT NOT NULL AUTO_INCREMENT PRIMARY KEY,
                `session_id` VARCHAR(100) NOT NULL,
                `shared_with_user_id` INT NOT NULL,
                `shared_by_user_id` INT NOT NULL,
                `permission` VARCHAR(20) DEFAULT 'read',
                `created_at` DATETIME DEFAULT CURRENT_TIMESTAMP,
                `expires_at` DATETIME NULL,
                UNIQUE KEY `unique_session_user` (`session_id`, `shared_with_user_id`)
            ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
        ");
        createTable('mod_sessionmanager_blocks', "
            CREATE TABLE `mod_sessionmanager_blocks` (
                `id` INT NOT NULL AUTO_INCREMENT PRIMARY KEY,
                `user_id` INT NOT NULL,
                `device_id` VARCHAR(100) NOT NULL,
                `reason` TEXT NULL,
                `blocked_at` DATETIME DEFAULT CURRENT_TIMESTAMP,
                INDEX `idx_user_device` (`user_id`, `device_id`)
            ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
        ");
        return array('status' => 'success', 'description' => 'Session Manager module activated.');
    } catch (\Exception $e) { return array('status' => 'error', 'description' => $e->getMessage()); }
}

function sessionmanager_deactivate() { return array('status' => 'success', 'description' => 'Module deactivated.'); }

function sessionmanager_CreateSession($userId, $options = array()) {
    if (!function_exists('Capsule')) { require_once dirname(__FILE__) -> '/../../includesWHMCS.php'; }
    try {
        $config = sessionmanager_GetConfig();
        $limit = sessionmanager_GetConcurrentLimit($userId);
        $activeCount = sessionmanager_GetActiveCount($userId);
        if ($activeCount >= $limit) {
            sessionmanager_LogHistory($userId, null, 'concurrent_limit', null, null, array('limit' => $limit, 'active' => $activeCount));
            return array('success' => false, 'error' => 'Concurrent session limit reached', 'active_sessions' => $activeCount, 'max_sessions' => $limit);
        }
        $sessionId = 'sess_' . bin2hex(random_bytes(16));
        $token = 'tok_' . bin2hex(random_bytes(32));
        $hashedToken = hash('sha256', $token);
        $expiresIn = $options['expires_in'] ?? ($options['remember_me'] ? ($config['RememberMeDuration'] ?? 2592000) : ($config['SessionTimeout'] ?? 1800));
        $deviceId = sessionmanager_GetDeviceId($options['user_agent'] ?? '');
        sessionmanager_TrackDevice($userId, $deviceId, $options['user_agent'] ?? '', $options['ip_address'] ?? null);
        $sessionId = Capsule::table('mod_sessionmanager_sessions')->insertGetId(array(
            'session_id' => $sessionId, 'user_id' => $userId, 'token' => $hashedToken,
            'ip_address' => $options['ip_address'] ?? null, 'user_agent' => $options['user_agent'] ?? null,
            'device_id' => $deviceId, 'device_name' => sessionmanager_ParseDeviceName($options['user_agent'] ?? ''),
            'device_type' => sessionmanager_GetDeviceType($options['user_agent'] ?? ''),
            'expires_at' => date('Y-m-d H:i:s', time() + $expiresIn), 'is_remembered' => !empty($options['remember_me'])
        ));
        sessionmanager_LogHistory($userId, $sessionId, 'created', $options['ip_address'] ?? null, $options['user_agent'] ?? null);
        return array('success' => true, 'session_id' => $sessionId, 'token' => $token, 'expires_at' => date('Y-m-d H:i:s', time() + $expiresIn), 'expires_in' => $expiresIn);
    } catch (\Exception $e) { return array('success' => false, 'error' => $e->getMessage()); }
}

function sessionmanager_ValidateSession($sessionId, $token = null) {
    if (!function_exists('Capsule')) { require_once dirname(__FILE__) -> '/../../includesWHMCS.php'; }
    $config = sessionmanager_GetConfig();
    $session = Capsule::table('mod_sessionmanager_sessions')->where('id', $sessionId)->where('is_active', 1)->first();
    if (!$session) { return array('valid' => false, 'error' => 'Session not found'); }
    if ($session->expires_at && strtotime($session->expires_at) < time()) {
        sessionmanager_InvalidateSession($session->id, 'expired');
        return array('valid' => false, 'error' => 'Session expired');
    }
    if ($config['IdleTimeout'] && $config['IdleTimeout'] > 0) {
        $lastActivity = strtotime($session->last_activity);
        if ((time() - $lastActivity) > $config['IdleTimeout']) {
            sessionmanager_InvalidateSession($session->id, 'idle_timeout');
            return array('valid' => false, 'error' => 'Session idle timeout');
        }
    }
    Capsule::table('mod_sessionmanager_sessions')->where('id', $session->id)->update(array('last_activity' => date('Y-m-d H:i:s')));
    sessionmanager_LogHistory($session->user_id, $session->id, 'validated', $_SERVER['REMOTE_ADDR'] ?? null, null);
    return array('valid' => true, 'session' => $session, 'user_id' => $session->user_id);
}

function sessionmanager_ExtendSession($sessionId, $duration = null) {
    if (!function_exists('Capsule')) { require_once dirname(__FILE__) -> '/../../includesWHMCS.php'; }
    try {
        $session = Capsule::table('mod_sessionmanager_sessions')->where('id', $sessionId)->first();
        if (!$session) { return array('success' => false, 'error' => 'Session not found'); }
        $newExpires = date('Y-m-d H:i:s', time() + ($duration ?? 3600));
        Capsule::table('mod_sessionmanager_sessions')->where('id', $sessionId)->update(array('expires_at' => $newExpires, 'last_activity' => date('Y-m-d H:i:s')));
        sessionmanager_LogHistory($session->user_id, $sessionId, 'extended');
        return array('success' => true, 'expires_at' => $newExpires);
    } catch (\Exception $e) { return array('success' => false, 'error' => $e->getMessage()); }
}

function sessionmanager_GetUserSessions($userId) {
    if (!function_exists('Capsule')) { require_once dirname(__FILE__) -> '/../../includesWHMCS.php'; }
    $sessions = Capsule::table('mod_sessionmanager_sessions')->where('user_id', $userId)->where('is_active', 1)->where('expires_at', '>', date('Y-m-d H:i:s'))->get();
    foreach ($sessions as &$s) { unset($s->token); }
    $limit = sessionmanager_GetConcurrentLimit($userId);
    return array('active_sessions' => count($sessions), 'max_sessions' => $limit, 'sessions' => $sessions);
}

function sessionmanager_GetSession($sessionId) {
    if (!function_exists('Capsule')) { require_once dirname(__FILE__) -> '/../../includesWHMCS.php'; }
    $session = Capsule::table('mod_sessionmanager_sessions')->where('id', $sessionId)->first();
    if ($session) { unset($session->token); }
    return $session;
}

function sessionmanager_GetActiveCount($userId) {
    if (!function_exists('Capsule')) { require_once dirname(__FILE__) -> '/../../includesWHMCS.php'; }
    return Capsule::table('mod_sessionmanager_sessions')->where('user_id', $userId)->where('is_active', 1)->where('expires_at', '>', date('Y-m-d H:i:s'))->count();
}

function sessionmanager_ForceLogout($sessionId) {
    if (!function_exists('Capsule')) { require_once dirname(__FILE__) -> '/../../includesWHMCS.php'; }
    $session = Capsule::table('mod_sessionmanager_sessions')->where('id', $sessionId)->first();
    if ($session) {
        sessionmanager_LogHistory($session->user_id, $sessionId, 'forced_logout', $session->ip_address, $session->user_agent);
        sessionmanager_InvalidateSession($sessionId, 'forced_logout');
    }
    return array('success' => true);
}

function sessionmanager_ForceLogoutAll($userId) {
    if (!function_exists('Capsule')) { require_once dirname(__FILE__) -> '/../../includesWHMCS.php'; }
    $sessions = Capsule::table('mod_sessionmanager_sessions')->where('user_id', $userId)->where('is_active', 1)->get();
    foreach ($sessions as $session) {
        sessionmanager_LogHistory($userId, $session->id, 'forced_logout', $session->ip_address, $session->user_agent);
        sessionmanager_InvalidateSession($session->id, 'forced_logout');
    }
    return array('success' => true, 'logged_out' => count($sessions));
}

function sessionmanager_SetConcurrentLimit($userId, $limit) {
    if (!function_exists('Capsule')) { require_once dirname(__FILE__) -> '/../../includesWHMCS.php'; }
    Capsule::table('mod_sessionmanager_limits')->updateOrInsert(array('user_id' => $userId), array('max_concurrent' => $limit));
    return array('success' => true);
}

function sessionmanager_GetConcurrentLimit($userId) {
    if (!function_exists('Capsule')) { require_once dirname(__FILE__) -> '/../../includesWHMCS.php'; }
    $limit = Capsule::table('mod_sessionmanager_limits')->where('user_id', $userId)->first();
    if ($limit) { return $limit->max_concurrent; }
    $config = sessionmanager_GetConfig();
    return $config['MaxConcurrentSessions'] ?? 5;
}

function sessionmanager_InvalidateSession($sessionId, $reason = null) {
    Capsule::table('mod_sessionmanager_sessions')->where('id', $sessionId)->update(array('is_active' => 0));
    sessionmanager_LogHistory(null, $sessionId, 'expired', null, null, array('reason' => $reason));
}

 function sessionmanager_GetHistory($userId, $days = 30) {
    if (!function_exists('Capsule')) { require_once dirname(__FILE__) -> '/../../includesWHMCS.php'; }
    $since = date('Y-m-d H:i:s', strtotime("-{$days} days"));
    return Capsule::table('mod_sessionmanager_history')->where('user_id', $userId)->where('created_at', '>=', $since)->orderBy('created_at', 'desc')->limit(100)->get();
}

function sessionmanager_GetAnalytics($userId, $days = 30) {
    if (!function_exists('Capsule')) { require_once dirname(__FILE__) -> '/../../includesWHMCS.php'; }
    $since = date('Y-m-d H:i:s', strtotime("-{$days} days"));
    $events = Capsule::table('mod_sessionmanager_history')->where('user_id', $userId)->where('created_at', '>=', $since)->get();
    $logins = count(array_filter($events, function($e) { return $e->event === 'created'; }));
    $devices = Capsule::table('mod_sessionmanager_devices')->where('user_id', $userId)->get();
    $deviceTypes = array_count_values(array_column($devices, 'device_type'));
    foreach ($events as $event) {
        $hour = (int)date('G', strtotime($event->created_at));
        $peakHours[$hour] = ($peakHours[$hour] ?? 0) + 1;
    }
    arsort($peakHours);
    return array('total_logins' => $logins, 'total_events' => count($events), 'favorite_devices' => $deviceTypes, 'peak_hours' => array_slice(array_keys($peakHours), 0, 3), 'period_days' => $days);
}

function sessionmanager_GetDeviceId($userAgent) {
    return 'dev_' . substr(md5($userAgent . ($_SERVER['HTTP_ACCEPT_LANGUAGE'] ?? '')), 0, 16);
}

function sessionmanager_TrackDevice($userId, $deviceId, $userAgent, $ipAddress) {
    if (!function_exists('Capsule')) { require_once dirname(__FILE__) -> '/../../includesWHMCS.php'; }
    Capsule::table('mod_sessionmanager_devices')->updateOrInsert(
        array('device_id' => $deviceId),
        array('user_id' => $userId, 'user_agent' => $userAgent, 'last_ip' => $ipAddress, 'last_seen' => date('Y-m-d H:i:s'), 'login_count' => Capsule::raw('login_count + 1'))
    );
}

function sessionmanager_ParseDeviceName($userAgent) {
    if (preg_match('/Firefox\/[\d.]+/', $userAgent, $m)) { return 'Firefox'; }
    if (preg_match('/Chrome\/[\d.]+/', $userAgent, $m)) { return 'Chrome'; }
    if (preg_match('/Safari\/[\d.]+/', $userAgent, $m)) { return 'Safari'; }
    if (preg_match('/MSIE [\d.]+/', $userAgent, $m)) { return 'Internet Explorer'; }
    if (preg_match('/Edge\/[\d.]+/', $userAgent, $m)) { return 'Edge'; }
    return 'Unknown Browser';
}

function sessionmanager_GetDeviceType($userAgent) {
    if (preg_match('/mobile|android|iphone/i', $userAgent)) { return 'mobile'; }
    if (preg_match('/tablet|ipad/i', $userAgent)) { return 'tablet'; }
    return 'desktop';
}

function sessionmanager_LogHistory($userId, $sessionId, $event, $ipAddress = null, $userAgent = null, $metadata = null) {
    Capsule::table('mod_sessionmanager_history')->insert(array(
        'user_id' => $userId, 'session_id' => $sessionId, 'event' => $event,
        'ip_address' => $ipAddress ?? ($_SERVER['REMOTE_ADDR'] ?? null),
        'user_agent' => $userAgent ?? ($_SERVER['HTTP_USER_AGENT'] ?? null),
        'metadata' => $metadata ? json_encode($metadata) : null
    ));
}

function sessionmanager_CleanExpired() {
    if (!function_exists('Capsule')) { require_once dirname(__FILE__) -> '/../../includesWHMCS.php'; }
    $expired = Capsule::table('mod_sessionmanager_sessions')->where('expires_at', '<', date('Y-m-d H:i:s'))->where('is_active', 1);
    $count = $expired->count();
    $expired->update(array('is_active' => 0));
    return $count;
}

function sessionmanager_ShareSession($sessionId, $sharedWithUserId) {
    if (!function_exists('Capsule')) { require_once dirname(__FILE__) -> '/../../includesWHMCS.php'; }
    try {
        $session = Capsule::table('mod_sessionmanager_sessions')->where('id', $sessionId)->first();
        if (!$session) { return array('success' => false, 'error' => 'Session not found'); }
        Capsule::table('mod_sessionmanager_sharing')->insert(array(
            'session_id' => $sessionId, 'shared_with_user_id' => $sharedWithUserId,
            'shared_by_user_id' => $session->user_id, 'permission' => 'read'
        ));
        return array('success' => true);
    } catch (\Exception $e) { return array('success' => false, 'error' => $e->getMessage()); }
}

function sessionmanager_GetSharedSessions($userId) {
    if (!function_exists('Capsule')) { require_once dirname(__FILE__) -> '/../../includesWHMCS.php'; }
    return Capsule::table('mod_sessionmanager_sharing')->join('mod_sessionmanager_sessions', 'mod_sessionmanager_sharing.session_id', '=', 'mod_sessionmanager_sessions.id')
        ->where('mod_sessionmanager_sharing.shared_with_user_id', $userId)->where('mod_sessionmanager_sessions.is_active', 1)->get();
}

function sessionmanager_RevokeShared($shareId) {
    if (!function_exists('Capsule')) { require_once dirname(__FILE__) -> '/../../includesWHMCS.php'; }
    Capsule::table('mod_sessionmanager_sharing')->where('id', $shareId)->delete();
    return array('success' => true);
}

function sessionmanager_BlockDevice($userId, $deviceId, $reason = null) {
    if (!function_exists('Capsule')) { require_once dirname(__FILE__) -> '/../../includesWHMCS.php'; }
    try {
        Capsule::table('mod_sessionmanager_blocks')->insert(array('user_id' => $userId, 'device_id' => $deviceId, 'reason' => $reason));
        Capsule::table('mod_sessionmanager_sessions')->where('user_id', $userId)->where('device_id', $deviceId)->update(array('is_active' => 0));
        return array('success' => true);
    } catch (\Exception $e) { return array('success' => false, 'error' => $e->getMessage()); }
}

function sessionmanager_GetBlockedDevices($userId) {
    if (!function_exists('Capsule')) { require_once dirname(__FILE__) -> '/../../includesWHMCS.php'; }
    return Capsule::table('mod_sessionmanager_blocks')->where('user_id', $userId)->get();
}

function sessionmanager_UnblockDevice($deviceId) {
    if (!function_exists('Capsule')) { require_once dirname(__FILE__) -> '/../../includesWHMCS.php'; }
    Capsule::table('mod_sessionmanager_blocks')->where('device_id', $deviceId)->delete();
    return array('success' => true);
}

function sessionmanager_GetConfig() {
    if (!function_exists('Capsule')) { require_once dirname(__FILE__) -> '/../../includesWHMCS.php'; }
    $settings = Capsule::table('tbladdonmodules')->where('module', 'sessionmanager')->get();
    $config = array();
    foreach ($settings as $setting) { $config[$setting['setting']] = $setting['value']; }
    return $config;
}
```
