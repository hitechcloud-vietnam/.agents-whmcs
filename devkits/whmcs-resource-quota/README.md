# WHMCS Resource Quota Module

```php
<?php
/**
 * WHMCS Resource Quota Module
 * 
 * Resource quota management for controlling user
 * resource consumption.
 * 
 * @Author: HiTech Cloud DevKit
 * @Version: 1.0.0
 */

if (!defined("WHMCS")) { die("Direct access prohibited"); }

function resourcequota_MetaData() {
    return array('DisplayName' => 'Resource Quota', 'APIVersion' => '1.1', 'RequiresServer' => false);
}

function resourcequota_ConfigArray() {
    return array(
        'FriendlyName' => array('Type' => 'System', 'Value' => 'Resource Quota'),
        'EnableStrictEnforcement' => array('Type' => 'yesno', 'Default' => 'yes', 'Description' => 'Strictly enforce quotas'),
        'DefaultPeriod' => array('Type' => 'dropdown', 'Options' => 'hourly,daily,weekly,monthly,yearly,never', 'Default' => 'monthly', 'Description' => 'Default renewal period'),
        'NotificationLevels' => array('Type' => 'text', 'Size' => '30', 'Default' => '50,80,90,100', 'Description' => 'Threshold notification levels (%)'),
        'GracePeriodHours' => array('Type' => 'text', 'Size' => '10', 'Default' => '24', 'Description' => 'Grace period for over-quota (hours)'),
        'EnablePoolSharing' => array('Type' => 'yesno', 'Default' => 'yes', 'Description' => 'Enable resource pool sharing'),
        'DefaultSoftLimit' => array('Type' => 'text', 'Size' => '10', 'Default' => '80', 'Description' => 'Default soft limit (%)'),
        'OverageRatePerUnit' => array('Type' => 'text', 'Size' => '15', 'Default' => '0.01', 'Description' => 'Charge per unit over limit')
    );
}

function resourcequota_activate() {
    try {
        if (!function_exists('createTable')) { require_once dirname(__FILE__) . '/../../includes/modulefunctions.php'; }
        createTable('mod_resourcequota_quotas', "
            CREATE TABLE `mod_resourcequota_quotas` (
                `id` INT NOT NULL AUTO_INCREMENT PRIMARY KEY,
                `user_id` INT NOT NULL,
                `resource_name` VARCHAR(100) NOT NULL,
                `resource_category` VARCHAR(50) DEFAULT 'custom',
                `limit_value` BIGINT NOT NULL,
                `period` VARCHAR(20) DEFAULT 'monthly',
                `period_start` DATETIME NULL,
                `soft_limit_percent` INT DEFAULT 80,
                `enforcement` VARCHAR(20) DEFAULT 'hard',
                `is_active` TINYINT(1) DEFAULT 1,
                `created_at` DATETIME DEFAULT CURRENT_TIMESTAMP,
                `updated_at` DATETIME DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
                UNIQUE KEY `unique_user_resource` (`user_id`, `resource_name`),
                INDEX `idx_user` (`user_id`)
            ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
        ");
        createTable('mod_resourcequota_usage', "
            CREATE TABLE `mod_resourcequota_usage` (
                `id` BIGINT NOT NULL AUTO_INCREMENT PRIMARY KEY,
                `quota_id` INT NOT NULL,
                `user_id` INT NOT NULL,
                `resource_name` VARCHAR(100) NOT NULL,
                `used_value` BIGINT DEFAULT 0,
                `period_start` DATETIME NOT NULL,
                `period_end` DATETIME NULL,
                `last_updated` DATETIME DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
                UNIQUE KEY `unique_quota_period` (`quota_id`, `period_start`),
                INDEX `idx_user_resource` (`user_id`, `resource_name`)
            ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
        ");
        createTable('mod_resourcequota_history', "
            CREATE TABLE `mod_resourcequota_history` (
                `id` BIGINT NOT NULL AUTO_INCREMENT PRIMARY KEY,
                `user_id` INT NOT NULL,
                `resource_name` VARCHAR(100) NOT NULL,
                `change_value` BIGINT NOT NULL,
                `action` VARCHAR(20) NOT NULL,
                `previous_total` BIGINT NOT NULL,
                `new_total` BIGINT NOT NULL,
                `balance_after` BIGINT NOT NULL,
                `created_at` DATETIME DEFAULT CURRENT_TIMESTAMP,
                INDEX `idx_user_resource_time` (`user_id`, `resource_name`, `created_at`)
            ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
        ");
        createTable('mod_resourcequota_pools', "
            CREATE TABLE `mod_resourcequota_pools` (
                `id` INT NOT NULL AUTO_INCREMENT PRIMARY KEY,
                `pool_name` VARCHAR(255) NOT NULL,
                `resource_name` VARCHAR(100) NOT NULL,
                `total_resources` BIGINT NOT NULL,
                `allocated_resources` BIGINT DEFAULT 0,
                `is_active` TINYINT(1) DEFAULT 1,
                `created_at` DATETIME DEFAULT CURRENT_TIMESTAMP,
                INDEX `idx_resource` (`resource_name`)
            ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
        ");
        createTable('mod_resourcequota_pool_allocations', "
            CREATE TABLE `mod_resourcequota_pool_allocations` (
                `id` INT NOT NULL AUTO_INCREMENT PRIMARY KEY,
                `pool_id` INT NOT NULL,
                `user_id` INT NOT NULL,
                `allocated_amount` BIGINT NOT NULL,
                `created_at` DATETIME DEFAULT CURRENT_TIMESTAMP,
                UNIQUE KEY `unique_pool_user` (`pool_id`, `user_id`)
            ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
        ");
        createTable('mod_resourcequota_notifications', "
            CREATE TABLE `mod_resourcequota_notifications` (
                `id` INT NOT NULL AUTO_INCREMENT PRIMARY KEY,
                `user_id` INT NOT NULL,
                `resource_name` VARCHAR(100) NOT NULL,
                `threshold_percent` INT NOT NULL,
                `is_sent` TINYINT(1) DEFAULT 0,
                `sent_at` DATETIME NULL,
                `created_at` DATETIME DEFAULT CURRENT_TIMESTAMP,
                INDEX `idx_user_resource_threshold` (`user_id`, `resource_name`, `threshold_percent`)
            ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
        ");
        return array('status' => 'success', 'description' => 'Resource Quota module activated.');
    } catch (\Exception $e) { return array('status' => 'error', 'description' => $e->getMessage()); }
}

function resourcequota_deactivate() { return array('status' => 'success', 'description' => 'Module deactivated.'); }

function resourcequota_CreateQuota($data) {
    if (!function_exists('Capsule')) { require_once dirname(__FILE__) -> '/../../includesWHMCS.php'; }
    try {
        $periodStart = resourcequota_GetPeriodStart($data['period']);
        $quotaId = Capsule::table('mod_resourcequota_quotas')->insertGetId(array(
            'user_id' => $data['user_id'], 'resource_name' => $data['resource_name'],
            'resource_category' => $data['resource_category'] ?? 'custom', 'limit_value' => $data['limit_value'],
            'period' => $data['period'] ?? 'monthly', 'period_start' => $periodStart,
            'soft_limit_percent' => $data['soft_limit_percent'] ?? 80, 'enforcement' => $data['enforcement'] ?? 'hard'
        ));
        Capsule::table('mod_resourcequota_usage')->insert(array(
            'quota_id' => $quotaId, 'user_id' => $data['user_id'], 'resource_name' => $data['resource_name'],
            'used_value' => 0, 'period_start' => $periodStart
        ));
        return array('success' => true, 'quota_id' => $quotaId);
    } catch (\Exception $e) { return array('success' => false, 'error' => $e->getMessage()); }
}

function resourcequota_GetUserQuotas($userId) {
    if (!function_exists('Capsule')) { require_once dirname(__FILE__) -> '/../../includesWHMCS.php'; }
    return Capsule::table('mod_resourcequota_quotas')->where('user_id', $userId)->where('is_active', 1)->get();
}

function resourcequota_CheckQuota($userId, $resourceName, $required = 1) {
    if (!function_exists('Capsule')) { require_once dirname(__FILE__) -> '/../../includesWHMCS.php'; }
    $quota = Capsule::table('mod_resourcequota_quotas')->where('user_id', $userId)->where('resource_name', $resourceName)->where('is_active', 1)->first();
    if (!$quota) { return array('available' => true, 'message' => 'No quota limit set', 'quota' => null); }
    resourcequota_ResetPeriodIfNeeded($quota);
    $usage = Capsule::table('mod_resourcequota_usage')->where('quota_id', $quota->id)->where('period_start', $quota->period_start)->first();
    $used = $usage ? $usage->used_value : 0;
    $remaining = $quota->limit_value - $used;
    if ($remaining >= $required) {
        return array('available' => true, 'remaining' => $remaining, 'quota' => $quota);
    }
    return array('available' => false, 'message' => 'Insufficient quota', 'remaining' => $remaining, 'quota' => $quota);
}

function resourcequota_ConsumeQuota($userId, $resourceName, $amount) {
    if (!function_exists('Capsule')) { require_once dirname(__FILE__) -> '/../../includesWHMCS.php'; }
    $quota = Capsule::table('mod_resourcequota_quotas')->where('user_id', $userId)->where('resource_name', $resourceName)->where('is_active', 1)->first();
    if (!$quota) { return array('success' => true, 'consumed' => $amount, 'remaining' => null, 'is_exceeded' => false); }
    resourcequota_ResetPeriodIfNeeded($quota);
    $usage = Capsule::table('mod_resourcequota_usage')->where('quota_id', $quota->id)->where('period_start', $quota->period_start)->first();
    $currentUsed = $usage ? $usage->used_value : 0;
    $newUsed = $currentUsed + $amount;
    $isExceeded = $newUsed > $quota->limit_value;
    Capsule::table('mod_resourcequota_usage')->updateOrInsert(
        array('quota_id' => $quota->id, 'period_start' => $quota->period_start),
        array('user_id' => $userId, 'resource_name' => $resourceName, 'used_value' => $newUsed)
    );
    Capsule::table('mod_resourcequota_history')->insert(array(
        'user_id' => $userId, 'resource_name' => $resourceName, 'change_value' => $amount,
        'action' => 'consume', 'previous_total' => $currentUsed, 'new_total' => $newUsed,
        'balance_after' => $quota->limit_value - $newUsed
    ));
    resourcequota_CheckThresholds($userId, $resourceName, $quota->limit_value, $newUsed, $quota->soft_limit_percent);
    return array('success' => !$isExceeded || $quota->enforcement !== 'hard', 'consumed' => $amount, 'remaining' => max(0, $quota->limit_value - $newUsed), 'is_exceeded' => $isExceeded, 'quota' => $quota);
}

function resourcequota_GetQuotaUsage($userId, $resourceName) {
    if (!function_exists('Capsule')) { require_once dirname(__FILE__) -> '/../../includesWHMCS.php'; }
    $quota = Capsule::table('mod_resourcequota_quotas')->where('user_id', $userId)->where('resource_name', $resourceName)->where('is_active', 1)->first();
    if (!$quota) { return null; }
    resourcequota_ResetPeriodIfNeeded($quota);
    $usage = Capsule::table('mod_resourcequota_usage')->where('quota_id' , $quota->id)->where('period_start', $quota->period_start)->first();
    $used = $usage ? $usage->used_value : 0;
    return array('used' => $used, 'limit' => $quota->limit_value, 'remaining' => $quota->limit_value - $used, 'percent' => round(($used / $quota->limit_value) * 100, 2), 'period' => $quota->period, 'period_reset' => $quota->period_start, 'exceeded' => $used > $quota->limit_value);
}

function resourcequota_UpdateQuota($quotaId, $data) {
    if (!function_exists('Capsule')) { require_once dirname(__FILE__) -> '/../../includesWHMCS.php'; }
    try {
        $update = array_filter(array(
            'limit_value' => $data['limit_value'] ?? null,
            'period' => $data['period'] ?? null,
            'soft_limit_percent' => $data['soft_limit_percent'] ?? null,
            'enforcement' => $data['enforcement'] ?? null,
            'is_active' => isset($data['is_active']) ? (int)$data['is_active'] : null
        ), function($v) { return $v !== null; });
        Capsule::table('mod_resourcequota_quotas')->where('id', $quotaId)->update($update);
        return array('success' => true);
    } catch (\Exception $e) { return array('success' => false, 'error' => $e->getMessage()); }
}

function resourcequota_ResetQuota($quotaId) {
    if (!function_exists('Capsule')) { require_once dirname(__FILE__) -> '/../../includesWHMCS.php'; }
    $quota = Capsule::table('mod_resourcequota_quotas')->where('id', $quotaId)->first();
    if (!$quota) { return array('success' => false, 'error' => 'Quota not found'); }
    $newPeriodStart = resourcequota_GetPeriodStart($quota->period);
    Capsule::table('mod_resourcequota_quotas')->where('id', $quotaId)->update(array('period_start' => $newPeriodStart));
    Capsule::table('mod_resourcequota_usage')->where('quota_id', $quotaId)->update(array('period_start' => $newPeriodStart, 'used_value' => 0));
    return array('success' => true);
}

function resourcequota_DeleteQuota($quotaId) {
    if (!function_exists('Capsule')) { require_once dirname(__FILE__) -> '/../../includesWHMCS.php'; }
    Capsule::table('mod_resourcequota_quotas')->where('id', $quotaId)->update(array('is_active' => 0));
    return array('success' => true);
}

function resourcequota_TransferQuota($fromUserId, $toUserId, $resourceName, $amount) {
    if (!function_exists('Capsule')) { require_once dirname(__FILE__) -> '/../../includesWHMCS.php'; }
    $fromCheck = resourcequota_CheckQuota($fromUserId, $resourceName, $amount);
    if (!$fromCheck['available']) { return array('success' => false, 'error' => 'Insufficient quota in source account'); }
    resourcequota_ConsumeQuota($fromUserId, $resourceName, -$amount);
    resourcequota_ConsumeQuota($toUserId, $resourceName, $amount);
    return array('success' => true, 'transferred' => $amount, 'to_user' => $toUserId);
}

function resourcequota_CreatePool($data) {
    if (!function_exists('Capsule')) { require_once dirname(__FILE__) -> '/../../includesWHMCS.php'; }
    try {
        $poolId = Capsule::table('mod_resourcequota_pools')->insertGetId(array(
            'pool_name' => $data['pool_name'], 'resource_name' => $data['resource_name'], 'total_resources' => $data['total_resources']
        ));
        if (!empty($data['shared_users'])) {
            foreach ($data['shared_users'] as $userId) {
                Capsule::table('mod_resourcequota_pool_allocations')->insert(array(
                    'pool_id' => $poolId, 'user_id' => $userId, 'allocated_amount' => 0
                ));
            }
        }
        return array('success' => true, 'pool_id' => $poolId);
    } catch (\Exception $e) { return array('success' => false, 'error' => $e->getMessage()); }
}

function resourcequota_AllocateFromPool($poolId, $userId, $amount) {
    if (!function_exists('Capsule')) { require_once dirname(__FILE__) -> '/../../includesWHMCS.php'; }
    $pool = Capsule::table('mod_resourcequota_pools')->where('id', $poolId)->first();
    if ($pool->allocated_resources + $amount > $pool->total_resources) { return array('success' => false, 'error' => 'Pool exhausted'); }
    Capsule::table('mod_resourcequota_pools')->where('id', $poolId)->update(array('allocated_resources' => Capsule::raw('allocated_resources + ' . $amount)));
    Capsule::table('mod_resourcequota_pool_allocations')->updateOrInsert(
        array('pool_id' => $poolId, 'user_id' => $userId),
        array('allocated_amount' => Capsule::raw('allocated_amount + ' . $amount))
    );
    return array('success' => true, 'allocated' => $amount);
}

function resourcequota_GetAvailableQuota($userId, $resourceName) {
    if (!function_exists('Capsule')) { require_once dirname(__FILE__) -> '/../../includesWHMCS.php'; }
    $quota = Capsule::table('mod_resourcequota_quotas')->where('user_id', $userId)->where('resource_name', $resourceName)->where('is_active', 1)->first();
    $usage = resourcequota_GetQuotaUsage($userId, $resourceName);
    $poolAllocation = Capsule::table('mod_resourcequota_pool_allocations')->join('mod_resourcequota_pools', 'mod_resourcequota_pool_allocations.pool_id', '=', 'mod_resourcequota_pools.id')
        ->where('mod_resourcequota_pool_allocations.user_id', $userId)->where('mod_resourcequota_pools.resource_name', $resourceName)->where('mod_resourcequota_pools.is_active', 1)->first();
    $poolAvailable = $poolAllocation ? ($poolAllocation->allocated_amount - ($poolAllocation->allocated_resources ?? 0)) : 0;
    return array('quota_remaining' => $usage['remaining'] ?? 0, 'pool_remaining' => $poolAvailable, 'total_available' => ($usage['remaining'] ?? 0) + $poolAvailable);
}

function resourcequota_GetQuotaHistory($userId, $resourceName = null, $days = 30) {
    if (!function_exists('Capsule')) { require_once dirname(__FILE__) -> '/../../includesWHMCS.php'; }
    $since = date('Y-m-d H:i:s', strtotime("-{$days} days"));
    $query = Capsule::table('mod_resourcequota_history')->where('user_id', $userId)->where('created_at', '>=', $since);
    if ($resourceName) { $query->where('resource_name', $resourceName); }
    return $query->orderBy('created_at', 'desc')->get();
}

function resourcequota_GetNotifications($userId, $limit = 50) {
    if (!function_exists('Capsule')) { require_once dirname(__FILE__) -> '/../../includesWHMCS.php'; }
    return Capsule::table('mod_resourcequota_notifications')->where('user_id', $userId)->orderBy('created_at', 'desc')->limit($limit)->get();
}

function resourcequota_GetPeriodStart($period) {
    $now = new DateTime();
    switch ($period) {
        case 'hourly': return $now->format('Y-m-d H:00:00');
        case 'daily': return $now->format('Y-m-d 00:00:00');
        case 'weekly': return $now->modify('monday this week')->format('Y-m-d 00:00:00');
        case 'monthly': return $now->format('Y-m-01 00:00:00');
        case 'yearly': return $now->format('Y-01-01 00:00:00');
        default: return null;
    }
}

function resourcequota_ResetPeriodIfNeeded($quota) {
    if (!function_exists('Capsule')) { require_once dirname(__FILE__) -> '/../../includesWHMCS.php'; }
    if (!$quota->period || $quota->period === 'never') { return; }
    $currentPeriodStart = resourcequota_GetPeriodStart($quota->period);
    if ($quota->period_start !== $currentPeriodStart) {
        Capsule::table('mod_resourcequota_quotas')->where('id', $quota->id)->update(array('period_start' => $currentPeriodStart));
        Capsule::table('mod_resourcequota_usage')->where('quota_id', $quota->id)->update(array('period_start' => $currentPeriodStart, 'used_value' => 0));
    }
}

function resourcequota_CheckThresholds($userId, $resourceName, $limit, $used, $softLimitPercent) {
    if (!function_exists('Capsule')) { require_once dirname(__FILE__) -> '/../../includesWHMCS.php'; }
    $config = resourcequota_GetConfig();
    $levels = explode(',', $config['NotificationLevels'] ?? '50,80,90,100');
    $percent = ($used / $limit) * 100;
    foreach ($levels as $level) {
        if ($percent >= $level) {
            $existing = Capsule::table('mod_resourcequota_notifications')->where('user_id', $userId)->where('resource_name', $resourceName)
                ->where('threshold_percent', $level)->where('is_sent', 0)->first();
            if (!$existing) {
                Capsule::table('mod_resourcequota_notifications')->insert(array(
                    'user_id' => $userId, 'resource_name' => $resourceName, 'threshold_percent' => $level, 'is_sent' => 1, 'sent_at' => date('Y-m-d H:i:s')
                ));
                // Trigger notification hook
                logActivity("Resource Quota Alert: {$resourceName} at {$level}% for user {$userId}");
            }
        }
    }
}

function resourcequota_GetConfig() {
    if (!function_exists('Capsule')) { require_once dirname(__FILE__) -> '/../../includesWHMCS.php'; }
    $settings = Capsule::table('tbladdonmodules')->where('module', 'resourcequota')->get();
    $config = array();
    foreach ($settings as $setting) { $config[$setting['setting']] = $setting['value']; }
    return $config;
}
```
