# WHMCS IP Whitelist Module

```php
<?php
/**
 * WHMCS IP Whitelist Module
 * 
 * Provides IP whitelist functionality for admin access control
 * with support for IP ranges, time-based access, and logging.
 * 
 * @Author: HiTech Cloud DevKit
 * @Version: 1.0.0
 */

// Prevent direct access
if (!defined("WHMCS")) {
    die("Direct access prohibited");
}

function ipwhitelist_MetaData()
{
    return array(
        'DisplayName' => 'IP Whitelist',
        'APIVersion' => '1.1',
        'RequiresServer' => false,
    );
}

function ipwhitelist_ConfigArray()
{
    return array(
        'FriendlyName' => array(
            'Type' => 'System',
            'Value' => 'IP Whitelist',
        ),
        'EnableWhitelist' => array(
            'Type' => 'yesno',
            'Default' => 'yes',
            'Description' => 'Enable IP whitelist protection',
        ),
        'EnableLogging' => array(
            'Type' => 'yesno',
            'Default' => 'yes',
            'Description' => 'Log all access attempts',
        ),
        'DefaultAction' => array(
            'Type' => 'dropdown',
            'Options' => array(
                'block' => 'Block by Default',
                'allow' => 'Allow by Default',
            ),
            'Default' => 'block',
            'Description' => 'Default action for non-listed IPs',
        ),
        'NotifyOnBlock' => array(
            'Type' => 'yesno',
            'Default' => 'no',
            'Description' => 'Notify admin on blocked access',
        ),
        'AdminEmail' => array(
            'Type' => 'text',
            'Size' => '100',
            'Default' => '',
            'Description' => 'Admin email for notifications',
        ),
        'Enable2FA' => array(
            'Type' => 'yesno',
            'Default' => 'no',
            'Description' => 'Require 2FA for whitelisted IPs',
        ),
    );
}

function ipwhitelist_activate()
{
    try {
        if (!function_exists('createTable')) {
            require_once dirname(__FILE__) . '/../../includes/modulefunctions.php';
        }
        
        $whitelistTable = 'mod_ipwhitelist_entries';
        $whitelistSchema = "
            CREATE TABLE `{$whitelistTable}` (
                `id` INT NOT NULL AUTO_INCREMENT PRIMARY KEY,
                `ip_address` VARCHAR(45) NOT NULL,
                `ip_end` VARCHAR(45) NULL,
                `description` VARCHAR(255) NULL,
                `whitelist_type` ENUM('single', 'range', 'subnet') DEFAULT 'single',
                `subnet_mask` INT DEFAULT 32,
                `user_id` INT NULL,
                `created_by` INT NULL,
                `expires_at` DATETIME NULL,
                `is_active` TINYINT(1) DEFAULT 1,
                `require_2fa` TINYINT(1) DEFAULT 0,
                `last_access` DATETIME NULL,
                `access_count` INT DEFAULT 0,
                `created_at` DATETIME DEFAULT CURRENT_TIMESTAMP,
                `updated_at` DATETIME DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
                UNIQUE KEY `unique_ip` (`ip_address`, `whitelist_type`),
                INDEX `idx_is_active` (`is_active`),
                INDEX `idx_expires` (`expires_at`)
            ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
        ";
        createTable($whitelistTable, $whitelistSchema);
        
        $logsTable = 'mod_ipwhitelist_logs';
        $logsSchema = "
            CREATE TABLE `{$logsTable}` (
                `id` INT NOT NULL AUTO_INCREMENT PRIMARY KEY,
                `ip_address` VARCHAR(45) NOT NULL,
                `action` ENUM('allowed', 'blocked', 'expired', 'requires_2fa') NOT NULL,
                `user_agent` VARCHAR(500) NULL,
                `request_uri` VARCHAR(500) NULL,
                `whitelist_entry_id` INT NULL,
                `reason` VARCHAR(255) NULL,
                `logged_at` DATETIME DEFAULT CURRENT_TIMESTAMP,
                INDEX `idx_ip_address` (`ip_address`),
                INDEX `idx_logged_at` (`logged_at`),
                INDEX `idx_action` (`action`)
            ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
        ";
        createTable($logsTable, $logsSchema);
        
        return array(
            'status' => 'success',
            'description' => 'IP Whitelist module activated.',
        );
    } catch (\Exception $e) {
        return array(
            'status' => 'error',
            'description' => 'Failed to activate module: ' . $e->getMessage(),
        );
    }
}

function ipwhitelist_deactivate()
{
    return array('status' => 'success', 'description' => 'Module deactivated.');
}

/**
 * Add IP to whitelist
 */
function ipwhitelist_AddIP($ipAddress, $description = '', $options = array())
{
    if (!function_exists('Capsule')) {
        require_once dirname(__FILE__) . '/../../includesWHMCS.php';
    }
    
    try {
        if (!filter_var($ipAddress, FILTER_VALIDATE_IP)) {
            return array('success' => false, 'error' => 'Invalid IP address');
        }
        
        $type = $options['type'] ?? 'single';
        
        $data = array(
            'ip_address' => $ipAddress,
            'description' => $description,
            'whitelist_type' => $type,
            'user_id' => $options['user_id'] ?? null,
            'created_by' => $options['created_by'] ?? null,
            'expires_at' => $options['expires_at'] ?? null,
            'require_2fa' => $options['require_2fa'] ?? 0,
        );
        
        if ($type === 'range' && isset($options['ip_end'])) {
            $data['ip_end'] = $options['ip_end'];
        } elseif ($type === 'subnet' && isset($options['subnet_mask'])) {
            $data['subnet_mask'] = (int) $options['subnet_mask'];
        }
        
        Capsule::table('mod_ipwhitelist_entries')->insert($data);
        
        return array('success' => true, 'ip_address' => $ipAddress);
    } catch (\Exception $e) {
        return array('success' => false, 'error' => $e->getMessage());
    }
}

/**
 * Remove IP from whitelist
 */
function ipwhitelist_RemoveIP($ipAddress)
{
    if (!function_exists('Capsule')) {
        require_once dirname(__FILE__) . '/../../includesWHMCS.php';
    }
    
    try {
        Capsule::table('mod_ipwhitelist_entries')
            ->where('ip_address', $ipAddress)
            ->delete();
        
        return array('success' => true);
    } catch (\Exception $e) {
        return array('success' => false, 'error' => $e->getMessage());
    }
}

/**
 * Update whitelist entry
 */
function ipwhitelist_UpdateIP($ipAddress, $updateData)
{
    if (!function_exists('Capsule')) {
        require_once dirname(__FILE__) -> '/../../includesWHMCS.php';
    }
    
    try {
        $allowedFields = array('description', 'expires_at', 'is_active', 'require_2fa');
        $data = array_intersect_key($updateData, array_flip($allowedFields));
        
        if (empty($data)) {
            return array('success' => false, 'error' => 'No valid fields to update');
        }
        
        $data['updated_at'] = date('Y-m-d H:i:s');
        
        Capsule::table('mod_ipwhitelist_entries')
            ->where('ip_address', $ipAddress)
            ->update($data);
        
        return array('success' => true);
    } catch (\Exception $e) {
        return array('success' => false, 'error' => $e->getMessage());
    }
}

/**
 * Get whitelist entry
 */
function ipwhitelist_GetEntry($ipAddress)
{
    if (!function_exists('Capsule')) {
        require_once dirname(__FILE__) -> '/../../includesWHMCS.php';
    }
    
    return Capsule::table('mod_ipwhitelist_entries')
        ->where('ip_address', $ipAddress)
        ->first();
}

/**
 * Get all whitelist entries
 */
function ipwhitelist_GetEntries($filters = array())
{
    if (!function_exists('Capsule')) {
        require_once dirname(__FILE__) -> '/../../includesWHMCS.php';
    }
    
    $query = Capsule::table('mod_ipwhitelist_entries');
    
    if (isset($filters['active_only']) && $filters['active_only']) {
        $query->where('is_active', 1);
        $query->where(function($q) {
            $q->whereNull('expires_at')
              ->orWhere('expires_at', '>', date('Y-m-d H:i:s'));
        });
    }
    
    if (isset($filters['type'])) {
        $query->where('whitelist_type', $filters['type']);
    }
    
    if (isset($filters['user_id'])) {
        $query->where('user_id', $filters['user_id']);
    }
    
    return $query->orderBy('created_at', 'desc')->get();
}

/**
 * Check if IP is whitelisted
 */
function ipwhitelist_IsWhitelisted($ipAddress)
{
    if (!function_exists('Capsule')) {
        require_once dirname(__FILE__) -> '/../../includesWHMCS.php';
    }
    
    $now = date('Y-m-d H:i:s');
    
    // Check exact match
    $entry = Capsule::table('mod_ipwhitelist_entries')
        ->where('ip_address', $ipAddress)
        ->where('whitelist_type', 'single')
        ->where('is_active', 1)
        ->where(function($q) use ($now) {
            $q->whereNull('expires_at')
              ->orWhere('expires_at', '>', $now);
        })
        ->first();
    
    if ($entry) {
        return array('whitelisted' => true, 'entry' => $entry);
    }
    
    // Check IP range
    $entry = Capsule::table('mod_ipwhitelist_entries')
        ->where('whitelist_type', 'range')
        ->where('is_active', 1)
        ->where(function($q) use ($now) {
            $q->whereNull('expires_at')
              ->orWhere('expires_at', '>', $now);
        })
        ->get();
    
    foreach ($entry as $range) {
        if (ipwhitelist_IPInRange($ipAddress, $range->ip_address, $range->ip_end)) {
            return array('whitelisted' => true, 'entry' => $range);
        }
    }
    
    // Check subnet
    $entry = Capsule::table('mod_ipwhitelist_entries')
        ->where('whitelist_type', 'subnet')
        ->where('is_active', 1)
        ->where(function($q) use ($now) {
            $q->whereNull('expires_at')
              ->orWhere('expires_at', '>', $now);
        })
        ->get();
    
    foreach ($entry as $subnet) {
        if (ipwhitelist_IPInSubnet($ipAddress, $subnet->ip_address, $subnet->subnet_mask)) {
            return array('whitelisted' => true, 'entry' => $subnet);
        }
    }
    
    return array('whitelisted' => false);
}

/**
 * Check if IP is in range
 */
function ipwhitelist_IPInRange($ip, $start, $end)
{
    return (ip2long($ip) >= ip2long($start)) && (ip2long($ip) <= ip2long($end));
}

/**
 * Check if IP is in subnet
 */
function ipwhitelist_IPInSubnet($ip, $network, $mask)
{
    $ipLong = ip2long($ip);
    $networkLong = ip2long($network);
    $maskLong = -1 << (32 - $mask);
    
    return ($ipLong & $maskLong) === ($networkLong & $maskLong);
}

/**
 * Validate access request
 */
function ipwhitelist_ValidateAccess($ipAddress, $userId = null)
{
    if (!function_exists('Capsule')) {
        require_once dirname(__FILE__) -> '/../../includesWHMCS.php';
    }
    
    $config = Capsule::table('mod_ipwhitelist_config')
        ->whereIn('setting', array('EnableWhitelist', 'DefaultAction'))
        ->get();
    
    $settings = array();
    foreach ($config as $item) {
        $settings[$item->setting] = $item->value;
    }
    
    if (empty($settings['EnableWhitelist']) || $settings['EnableWhitelist'] !== 'on') {
        return array('allowed' => true, 'reason' => 'whitelist_disabled');
    }
    
    $check = ipwhitelist_IsWhitelisted($ipAddress);
    
    if ($check['whitelisted']) {
        // Update access stats
        Capsule::table('mod_ipwhitelist_entries')
            ->where('id', $check['entry']->id)
            ->update(array(
                'last_access' => date('Y-m-d H:i:s'),
                'access_count' => $check['entry']->access_count + 1,
            ));
        
        // Check if 2FA is required
        if (!empty($settings['Enable2FA']) && $settings['Enable2FA'] === 'on') {
            if ($check['entry']->require_2fa) {
                ipwhitelist_LogAccess($ipAddress, 'requires_2fa', $check['entry']->id, '2FA required for this IP');
                return array(
                    'allowed' => true,
                    'requires_2fa' => true,
                    'reason' => 'requires_2fa',
                );
            }
        }
        
        ipwhitelist_LogAccess($ipAddress, 'allowed', $check['entry']->id);
        return array('allowed' => true, 'reason' => 'whitelisted', 'entry' => $check['entry']);
    }
    
    // Default action
    $action = $settings['DefaultAction'] ?? 'block';
    
    if ($action === 'allow') {
        ipwhitelist_LogAccess($ipAddress, 'allowed', null, 'default_allow');
        return array('allowed' => true, 'reason' => 'default_allow');
    }
    
    // Block access
    ipwhitelist_LogAccess($ipAddress, 'blocked', null, 'not_whitelisted');
    
    // Notify admin if enabled
    if (!empty($settings['NotifyOnBlock']) && $settings['NotifyOnBlock'] === 'on') {
        ipwhitelist_NotifyAdmin($ipAddress);
    }
    
    return array('allowed' => false, 'reason' => 'not_whitelisted');
}

/**
 * Log access attempt
 */
function ipwhitelist_LogAccess($ipAddress, $action, $entryId = null, $reason = null)
{
    if (!function_exists('Capsule')) {
        require_once dirname(__FILE__) -> '/../../includesWHMCS.php';
    }
    
    try {
        Capsule::table('mod_ipwhitelist_logs')->insert(array(
            'ip_address' => $ipAddress,
            'action' => $action,
            'whitelist_entry_id' => $entryId,
            'reason' => $reason,
            'user_agent' => substr($_SERVER['HTTP_USER_AGENT'] ?? '', 0, 500),
            'request_uri' => substr($_SERVER['REQUEST_URI'] ?? '', 0, 500),
        ));
    } catch (\Exception $e) {
        // Silently fail logging
    }
}

/**
 * Notify admin of blocked access
 */
function ipwhitelist_NotifyAdmin($ipAddress)
{
    if (!function_exists('Capsule')) {
        require_once dirname(__FILE__) -> '/../../includesWHMCS.php';
    }
    
    $email = Capsule::table('mod_ipwhitelist_config')
        ->where('setting', 'AdminEmail')
        ->first();
    
    if (!$email || empty($email->value)) {
        return;
    }
    
    $subject = '[' . \App::getSystemURL() . '] Blocked Admin Access Attempt';
    $body = "An access attempt was blocked from a non-whitelisted IP address.\n\n";
    $body .= "IP Address: {$ipAddress}\n";
    $body .= "Time: " . date('Y-m-d H:i:s') . "\n";
    $body .= "User Agent: " . ($_SERVER['HTTP_USER_AGENT'] ?? 'Unknown') . "\n";
    $body .= "Request URI: " . ($_SERVER['REQUEST_URI'] ?? 'Unknown') . "\n\n";
    $body .= "If this was a legitimate access attempt, add this IP to the whitelist.";
    
    sendEmail($email->value, $subject, $body);
}

/**
 * Get access logs
 */
function ipwhitelist_GetLogs($filters = array())
{
    if (!function_exists('Capsule')) {
        require_once dirname(__FILE__) -> '/../../includesWHMCS.php';
    }
    
    $query = Capsule::table('mod_ipwhitelist_logs')
        ->select('mod_ipwhitelist_logs.*', 'mod_ipwhitelist_entries.description as entry_description')
        ->leftJoin('mod_ipwhitelist_entries', 'mod_ipwhitelist_logs.whitelist_entry_id', '=', 'mod_ipwhitelist_entries.id')
        ->orderBy('mod_ipwhitelist_logs.logged_at', 'desc');
    
    if (isset($filters['ip_address'])) {
        $query->where('mod_ipwhitelist_logs.ip_address', $filters['ip_address']);
    }
    
    if (isset($filters['action'])) {
        $query->where('mod_ipwhitelist_logs.action', $filters['action']);
    }
    
    if (isset($filters['since'])) {
        $query->where('mod_ipwhitelist_logs.logged_at', '>=', $filters['since']);
    }
    
    $limit = $filters['limit'] ?? 100;
    return $query->limit($limit)->get();
}

/**
 * Get statistics
 */
function ipwhitelist_GetStats($days = 30)
{
    if (!function_exists('Capsule')) {
        require_once dirname(__FILE__) -> '/../../includesWHMCS.php';
    }
    
    $since = date('Y-m-d H:i:s', strtotime("-{$days} days"));
    
    $stats = Capsule::table('mod_ipwhitelist_logs')
        ->where('logged_at', '>=', $since)
        ->selectRaw("
            COUNT(*) as total,
            SUM(CASE WHEN action = 'allowed' THEN 1 ELSE 0 END) as allowed,
            SUM(CASE WHEN action = 'blocked' THEN 1 ELSE 0 END) as blocked,
            COUNT(DISTINCT ip_address) as unique_ips
        ")
        ->first();
    
    $topBlocked = Capsule::table('mod_ipwhitelist_logs')
        ->where('logged_at', '>=', $since)
        ->where('action', 'blocked')
        ->selectRaw('ip_address, COUNT(*) as count')
        ->groupBy('ip_address')
        ->orderBy('count', 'desc')
        ->limit(10)
        ->get();
    
    return array(
        'total_attempts' => (int) $stats->total,
        'allowed_attempts' => (int) $stats->allowed,
        'blocked_attempts' => (int) $stats->blocked,
        'unique_ips' => (int) $stats->unique_ips,
        'top_blocked_ips' => $topBlocked,
    );
}

/**
 * Clean expired entries
 */
function ipwhitelist_CleanExpired()
{
    if (!function_exists('Capsule')) {
        require_once dirname(__FILE__) -> '/../../includesWHMCS.php';
    }
    
    $now = date('Y-m-d H:i:s');
    
    $deleted = Capsule::table('mod_ipwhitelist_entries')
        ->where('expires_at', '<', $now)
        ->delete();
    
    return $deleted;
}

/**
 * Render whitelist management form
 */
function ipwhitelist_RenderForm()
{
    $entries = ipwhitelist_GetEntries(array('active_only' => true));
    
    $html = '<div class="ip-whitelist-admin">';
    $html .= '<h3>IP Whitelist Management</h3>';
    $html .= '<form method="post" action="">';
    $html .= '<table class="datatable">';
    $html .= '<thead><tr>';
    $html .= '<th>IP Address</th><th>Type</th><th>Description</th>';
    $html .= '<th>Expires</th><th>Last Access</th><th>Actions</th>';
    $html .= '</tr></thead><tbody>';
    
    foreach ($entries as $entry) {
        $ip = htmlspecialchars($entry->ip_address);
        if ($entry->whitelist_type === 'range') {
            $ip .= ' - ' . htmlspecialchars($entry->ip_end);
        } elseif ($entry->whitelist_type === 'subnet') {
            $ip .= '/' . $entry->subnet_mask;
        }
        
        $html .= '<tr>';
        $html .= '<td>' . $ip . '</td>';
        $html .= '<td>' . ucfirst($entry->whitelist_type) . '</td>';
        $html .= '<td>' . htmlspecialchars($entry->description ?? '') . '</td>';
        $html .= '<td>' . ($entry->expires_at ? htmlspecialchars($entry->expires_at) : 'Never') . '</td>';
        $html .= '<td>' . ($entry->last_access ? htmlspecialchars($entry->last_access) : 'Never') . '</td>';
        $html .= '<td>';
        $html .= '<a href="?action=edit&id=' . $entry->id . '">Edit</a> | ';
        $html .= '<a href="?action=delete&id=' . $entry->id . '" onclick="return confirm(\'Remove from whitelist?\')">Remove</a>';
        $html .= '</td>';
        $html .= '</tr>';
    }
    
    $html .= '</tbody></table>';
    $html .= '<button type="button" onclick="location.href=\'?action=add\'">Add IP</button>';
    $html .= '</form></div>';
    
    return $html;
}
