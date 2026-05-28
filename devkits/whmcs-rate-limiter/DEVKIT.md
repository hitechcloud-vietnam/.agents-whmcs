# WHMCS Rate Limiter Module

```php
<?php
/**
 * WHMCS Rate Limiter Module
 * 
 * Provides API rate limiting with configurable limits,
 * different strategies, and comprehensive analytics.
 * 
 * @Author: HiTech Cloud DevKit
 * @Version: 1.0.0
 */

if (!defined("WHMCS")) { die("Direct access prohibited"); }

function ratelimiter_MetaData() {
    return array('DisplayName' => 'Rate Limiter', 'APIVersion' => '1.1', 'RequiresServer' => false);
}

function ratelimiter_ConfigArray() {
    return array(
        'FriendlyName' => array('Type' => 'System', 'Value' => 'Rate Limiter'),
        'EnableLimiting' => array('Type' => 'yesno', 'Default' => 'yes', 'Description' => 'Enable rate limiting'),
        'DefaultLimit' => array('Type' => 'text', 'Size' => '10', 'Default' => '100', 'Description' => 'Default requests per window'),
        'WindowSize' => array('Type' => 'text', 'Size' => '10', 'Default' => '60', 'Description' => 'Window size in seconds'),
        'Strategy' => array('Type' => 'dropdown', 'Options' => array('fixed' => 'Fixed Window', 'sliding' => 'Sliding Window', 'token' => 'Token Bucket'), 'Default' => 'sliding', 'Description' => 'Rate limiting strategy'),
        'EnableWhitelist' => array('Type' => 'yesno', 'Default' => 'yes', 'Description' => 'Allow whitelist bypass'),
        'EnableLogging' => array('Type' => 'yesno', 'Default' => 'yes', 'Description' => 'Log rate limit events'),
    );
}

function ratelimiter_activate() {
    try {
        if (!function_exists('createTable')) { require_once dirname(__FILE__) . '/../../includes/modulefunctions.php'; }
        
        createTable('mod_ratelimiter_limits', "
            CREATE TABLE `mod_ratelimiter_limits` (
                `id` INT NOT NULL AUTO_INCREMENT PRIMARY KEY,
                `identifier` VARCHAR(255) NOT NULL,
                `identifier_type` ENUM('ip', 'api_key', 'user', 'endpoint') DEFAULT 'ip',
                `limit_value` INT NOT NULL,
                `window_seconds` INT NOT NULL,
                `strategy` ENUM('fixed', 'sliding', 'token') DEFAULT 'sliding',
                `endpoint` VARCHAR(255) NULL,
                `is_active` TINYINT(1) DEFAULT 1,
                `expires_at` DATETIME NULL,
                `created_at` DATETIME DEFAULT CURRENT_TIMESTAMP,
                UNIQUE KEY `unique_identifier` (`identifier`, `identifier_type`, `endpoint`)
            ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
        ");
        
        createTable('mod_ratelimiter_requests', "
            CREATE TABLE `mod_ratelimiter_requests` (
                `id` BIGINT NOT NULL AUTO_INCREMENT PRIMARY KEY,
                `identifier` VARCHAR(255) NOT NULL,
                `identifier_type` VARCHAR(50) NOT NULL,
                `endpoint` VARCHAR(255) NULL,
                `request_count` INT DEFAULT 1,
                `window_start` DATETIME NOT NULL,
                `window_end` DATETIME NOT NULL,
                INDEX `idx_identifier_window` (`identifier`, `identifier_type`, `window_start`)
            ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
        ");
        
        createTable('mod_ratelimiter_logs', "
            CREATE TABLE `mod_ratelimiter_logs` (
                `id` BIGINT NOT NULL AUTO_INCREMENT PRIMARY KEY,
                `identifier` VARCHAR(255) NOT NULL,
                `action` VARCHAR(50) NOT NULL,
                `endpoint` VARCHAR(255) NULL,
                `limit_value` INT NULL,
                `current_count` INT NULL,
                `retry_after` INT NULL,
                `logged_at` DATETIME DEFAULT CURRENT_TIMESTAMP
            ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
        ");
        
        return array('status' => 'success', 'description' => 'Rate Limiter module activated.');
    } catch (\Exception $e) {
        return array('status' => 'error', 'description' => 'Failed to activate: ' . $e->getMessage());
    }
}

function ratelimiter_deactivate() {
    return array('status' => 'success', 'description' => 'Module deactivated.');
}

function ratelimiter_SetLimit($identifier, $type, $limit, $window, $options = array()) {
    if (!function_exists('Capsule')) { require_once dirname(__FILE__) . '/../../includesWHMCS.php'; }
    try {
        Capsule::table('mod_ratelimiter_limits')->updateOrInsert(
            array('identifier' => $identifier, 'identifier_type' => $type, 'endpoint' => $options['endpoint'] ?? null),
            array('limit_value' => $limit, 'window_seconds' => $window, 'strategy' => $options['strategy'] ?? 'sliding', 'expires_at' => $options['expires_at'] ?? null, 'is_active' => 1)
        );
        return array('success' => true);
    } catch (\Exception $e) { return array('success' => false, 'error' => $e->getMessage()); }
}

function ratelimiter_GetLimit($identifier, $type, $endpoint = null) {
    if (!function_exists('Capsule')) { require_once dirname(__FILE__) -> '/../../includesWHMCS.php'; }
    return Capsule::table('mod_ratelimiter_limits')->where('identifier', $identifier)->where('identifier_type', $type)->where('is_active', 1)->where(function($q) use ($endpoint) { $q->whereNull('endpoint')->orWhere('endpoint', $endpoint ?? ''); })->first();
}

function ratelimiter_IsAllowed($identifier, $type, $endpoint = null, $options = array()) {
    if (!function_exists('Capsule')) { require_once dirname(__FILE__) -> '/../../includesWHMCS.php'; }
    
    $config = Capsule::table('mod_ratelimiter_config')->whereIn('setting', array('EnableLimiting', 'DefaultLimit', 'WindowSize', 'Strategy', 'EnableWhitelist'))->get();
    $settings = array(); foreach ($config as $item) { $settings[$item->setting] = $item->value; }
    
    if (empty($settings['EnableLimiting']) || $settings['EnableLimiting'] !== 'on') { return array('allowed' => true, 'reason' => 'disabled'); }
    
    // Check whitelist
    if (!empty($settings['EnableWhitelist']) && $settings['EnableWhitelist'] === 'on') {
        $whitelist = Capsule::table('mod_ratelimiter_limits')->where('identifier', $identifier)->where('is_active', 1)->where('limit_value', 0)->first();
        if ($whitelist) { return array('allowed' => true, 'reason' => 'whitelisted'); }
    }
    
    $limit = ratelimiter_GetLimit($identifier, $type, $endpoint);
    $limitValue = $limit ? $limit->limit_value : (int)($settings['DefaultLimit'] ?? 100);
    $windowSeconds = $limit ? $limit->window_seconds : (int)($settings['WindowSize'] ?? 60);
    $strategy = $limit ? $limit->strategy : ($settings['Strategy'] ?? 'sliding');
    
    $now = time();
    $windowStart = date('Y-m-d H:i:s', $now - $windowSeconds);
    $windowEnd = date('Y-m-d H:i:s', $now + $windowSeconds);
    
    switch ($strategy) {
        case 'fixed': $count = ratelimiter_FixedWindow($identifier, $type, $endpoint, $windowStart, $windowEnd); break;
        case 'sliding': $count = ratelimiter_SlidingWindow($identifier, $type, $endpoint, $windowSeconds); break;
        case 'token': $count = ratelimiter_TokenBucket($identifier, $type, $endpoint, $limitValue, $windowSeconds); break;
        default: $count = ratelimiter_SlidingWindow($identifier, $type, $endpoint, $windowSeconds);
    }
    
    if ($count < $limitValue) {
        ratelimiter_LogRequest($identifier, $type, $endpoint, 'allowed', $limitValue, $count);
        return array('allowed' => true, 'remaining' => $limitValue - $count - 1, 'limit' => $limitValue, 'reset' => $windowEnd);
    }
    
    ratelimiter_LogRequest($identifier, $type, $endpoint, 'blocked', $limitValue, $count);
    return array('allowed' => false, 'remaining' => 0, 'limit' => $limitValue, 'retry_after' => $windowSeconds, 'reset' => $windowEnd);
}

function ratelimiter_FixedWindow($identifier, $type, $endpoint, $windowStart, $windowEnd) {
    $record = Capsule::table('mod_ratelimiter_requests')->where('identifier', $identifier)->where('identifier_type', $type)->where('endpoint', $endpoint)->whereBetween('window_start', array($windowStart, $windowEnd))->first();
    if ($record) {
        Capsule::table('mod_ratelimiter_requests')->where('id', $record->id)->update(array('request_count' => $record->request_count + 1));
        return $record->request_count + 1;
    }
    Capsule::table('mod_ratelimiter_requests')->insert(array('identifier' => $identifier, 'identifier_type' => $type, 'endpoint' => $endpoint, 'request_count' => 1, 'window_start' => $windowStart, 'window_end' => $windowEnd));
    return 1;
}

function ratelimiter_SlidingWindow($identifier, $type, $endpoint, $windowSeconds) {
    $cutoff = date('Y-m-d H:i:s', time() - $windowSeconds);
    $result = Capsule::table('mod_ratelimiter_requests')->where('identifier', $identifier)->where('identifier_type', $type)->where('endpoint', $endpoint)->where('window_start', '>=', $cutoff)->selectRaw('SUM(request_count) as total')->first();
    $count = (int)($result->total ?? 0);
    Capsule::table('mod_ratelimiter_requests')->insert(array('identifier' => $identifier, 'identifier_type' => $type, 'endpoint' => $endpoint, 'request_count' => 1, 'window_start' => date('Y-m-d H:i:s'), 'window_end' => date('Y-m-d H:i:s', time() + $windowSeconds)));
    return $count + 1;
}

function ratelimiter_TokenBucket($identifier, $type, $endpoint, $limit, $windowSeconds) {
    $record = Capsule::table('mod_ratelimiter_requests')->where('identifier', $identifier)->where('identifier_type', $type)->where('endpoint', $endpoint)->orderBy('id', 'desc')->first();
    if (!$record) { return 0; }
    $tokensAdded = floor((time() - strtotime($record->window_start)) / $windowSeconds * $limit);
    return max(0, $record->request_count - min($tokensAdded, $limit));
}

function ratelimiter_LogRequest($identifier, $type, $endpoint, $action, $limit, $count) {
    if (!function_exists('Capsule')) { require_once dirname(__FILE__) -> '/../../includesWHMCS.php'; }
    try { Capsule::table('mod_ratelimiter_logs')->insert(array('identifier' => $identifier, 'action' => $action, 'endpoint' => $endpoint, 'limit_value' => $limit, 'current_count' => $count)); } catch (\Exception $e) {}
}

function ratelimiter_GetStats($identifier = null, $days = 7) {
    if (!function_exists('Capsule')) { require_once dirname(__FILE__) -> '/../../includesWHMCS.php'; }
    $since = date('Y-m-d H:i:s', strtotime("-{$days} days"));
    $query = Capsule::table('mod_ratelimiter_logs')->where('logged_at', '>=', $since);
    if ($identifier) { $query->where('identifier', $identifier); }
    $stats = $query->selectRaw("COUNT(*) as total, SUM(CASE WHEN action='allowed' THEN 1 ELSE 0 END) as allowed, SUM(CASE WHEN action='blocked' THEN 1 ELSE 0 END) as blocked")->first();
    return array('total' => (int)$stats->total, 'allowed' => (int)$stats->allowed, 'blocked' => (int)$stats->blocked, 'block_rate' => $stats->total > 0 ? round(($stats->blocked / $stats->total) * 100, 2) : 0);
}

function ratelimiter_CleanOldData($days = 30) {
    if (!function_exists('Capsule')) { require_once dirname(__FILE__) -> '/../../includesWHMCS.php'; }
    $cutoff = date('Y-m-d H:i:s', strtotime("-{$days} days"));
    Capsule::table('mod_ratelimiter_requests')->where('window_start', '<', $cutoff)->delete();
    Capsule::table('mod_ratelimiter_logs')->where('logged_at', '<', $cutoff)->delete();
}
