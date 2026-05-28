# WHMCS Auto Scaler Module

```php
<?php
/**
 * WHMCS Auto Scaler Module
 * 
 * Auto scaling configuration for resources based on
 * usage metrics and thresholds.
 * 
 * @Author: HiTech Cloud DevKit
 * @Version: 1.0.0
 */

if (!defined("WHMCS")) { die("Direct access prohibited"); }

function autoscaler_MetaData() {
    return array('DisplayName' => 'Auto Scaler', 'APIVersion' => '1.1', 'RequiresServer' => false);
}

function autoscaler_ConfigArray() {
    return array(
        'FriendlyName' => array('Type' => 'System', 'Value' => 'Auto Scaler'),
        'EnableAutoScale' => array('Type' => 'yesno', 'Default' => 'yes', 'Description' => 'Enable auto scaling'),
        'DefaultCooldown' => array('Type' => 'text', 'Size' => '10', 'Default' => '300', 'Description' => 'Default cooldown (seconds)'),
        'DefaultMinInstances' => array('Type' => 'text', 'Size' => '10', 'Default' => '1', 'Description' => 'Minimum instances'),
        'DefaultMaxInstances' => array('Type' => 'text', 'Size' => '10', 'Default' => '10', 'Description' => 'Maximum instances'),
        'EvaluationInterval' => array('Type' => 'text', 'Size' => '10', 'Default' => '60', 'Description' => 'Evaluation interval (seconds)'),
        'EnableScheduledScale' => array('Type' => 'yesno', 'Default' => 'yes', 'Description' => 'Enable scheduled scaling'),
        'EnableMetricsLogging' => array('Type' => 'yesno', 'Default' => 'yes', 'Description' => 'Log all metrics')
    );
}

function autoscaler_activate() {
    try {
        if (!function_exists('createTable')) { require_once dirname(__FILE__) . '/../../includes/modulefunctions.php'; }
        createTable('mod_autoscaler_rules', "
            CREATE TABLE `mod_autoscaler_rules` (
                `id` INT NOT NULL AUTO_INCREMENT PRIMARY KEY,
                `rule_name` VARCHAR(255) NOT NULL,
                `service_id` INT NOT NULL,
                `metric` VARCHAR(50) NOT NULL,
                `condition` VARCHAR(20) NOT NULL,
                `threshold` DECIMAL(20,6) NOT NULL,
                `scale_action` VARCHAR(50) NOT NULL,
                `scale_value` INT NOT NULL DEFAULT 1,
                `cooldown_seconds` INT DEFAULT 300,
                `min_instances` INT DEFAULT 1,
                `max_instances` INT DEFAULT 10,
                `is_active` TINYINT(1) DEFAULT 1,
                `last_triggered_at` DATETIME NULL,
                `created_at` DATETIME DEFAULT CURRENT_TIMESTAMP,
                INDEX `idx_service_metric` (`service_id`, `metric`)
            ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
        ");
        createTable('mod_autoscaler_metrics', "
            CREATE TABLE `mod_autoscaler_metrics` (
                `id` BIGINT NOT NULL AUTO_INCREMENT PRIMARY KEY,
                `service_id` INT NOT NULL,
                `metric` VARCHAR(50) NOT NULL,
                `value` DECIMAL(20,6) NOT NULL,
                `recorded_at` DATETIME DEFAULT CURRENT_TIMESTAMP,
                INDEX `idx_service_metric_time` (`service_id`, `metric`, `recorded_at`)
            ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
        ");
        createTable('mod_autoscaler_history', "
            CREATE TABLE `mod_autoscaler_history` (
                `id` BIGINT NOT NULL AUTO_INCREMENT PRIMARY KEY,
                `service_id` INT NOT NULL,
                `rule_id` INT NULL,
                `scale_action` VARCHAR(50) NOT NULL,
                `old_value` INT NOT NULL,
                `new_value` INT NOT NULL,
                `trigger_reason` TEXT NULL,
                `executed_by` INT NULL,
                `executed_at` DATETIME DEFAULT CURRENT_TIMESTAMP,
                INDEX `idx_service_time` (`service_id`, `executed_at`)
            ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
        ");
        createTable('mod_autoscaler_scheduled', "
            CREATE TABLE `mod_autoscaler_scheduled` (
                `id` INT NOT NULL AUTO_INCREMENT PRIMARY KEY,
                `service_id` INT NOT NULL,
                `rule_name` VARCHAR(255) NULL,
                `scale_action` VARCHAR(50) NOT NULL,
                `scale_value` INT NOT NULL,
                `schedule` JSON NOT NULL,
                `scheduled_at` DATETIME NOT NULL,
                `is_executed` TINYINT(1) DEFAULT 0,
                `executed_at` DATETIME NULL,
                `created_at` DATETIME DEFAULT CURRENT_TIMESTAMP,
                INDEX `idx_service_scheduled` (`service_id`, `scheduled_at`)
            ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
        ");
        createTable('mod_autoscaler_overrides', "
            CREATE TABLE `mod_autoscaler_overrides` (
                `id` INT NOT NULL AUTO_INCREMENT PRIMARY KEY,
                `service_id` INT UNIQUE NOT NULL,
                `instance_count` INT NOT NULL,
                `reason` TEXT NULL,
                `set_by` INT NULL,
                `expires_at` DATETIME NULL,
                `created_at` DATETIME DEFAULT CURRENT_TIMESTAMP,
                INDEX `idx_service` (`service_id`)
            ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
        ");
        createTable('mod_autoscaler_operations', "
            CREATE TABLE `mod_autoscaler_operations` (
                `id` INT NOT NULL AUTO_INCREMENT PRIMARY KEY,
                `service_id` INT NOT NULL,
                `operation_type` VARCHAR(50) NOT NULL,
                `target_value` INT NULL,
                `status` VARCHAR(20) DEFAULT 'pending',
                `started_at` DATETIME DEFAULT CURRENT_TIMESTAMP,
                `completed_at` DATETIME NULL,
                INDEX `idx_service_status` (`service_id`, `status`)
            ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
        ");
        return array('status' => 'success', 'description' => 'Auto Scaler module activated.');
    } catch (\Exception $e) { return array('status' => 'error', 'description' => $e->getMessage()); }
}

function autoscaler_deactivate() { return array('status' => 'success', 'description' => 'Module deactivated.'); }

function autoscaler_CreateRule($data) {
    if (!function_exists('Capsule')) { require_once dirname(__FILE__) -> '/../../includesWHMCS.php'; }
    try {
        $ruleId = Capsule::table('mod_autoscaler_rules')->insertGetId(array(
            'rule_name' => $data['rule_name'], 'service_id' => $data['service_id'],
            'metric' => $data['metric'], 'condition' => $data['condition'], 'threshold' => $data['threshold'],
            'scale_action' => $data['scale_action'], 'scale_value' => $data['scale_value'] ?? 1,
            'cooldown_seconds' => $data['cooldown_seconds'] ?? 300,
            'min_instances' => $data['min_instances'] ?? 1, 'max_instances' => $data['max_instances'] ?? 10
        ));
        return array('success' => true, 'rule_id' => $ruleId);
    } catch (\Exception $e) { return array('success' => false, 'error' => $e->getMessage()); }
}

function autoscaler_GetRules($serviceId = null) {
    if (!function_exists('Capsule')) { require_once dirname(__FILE__) -> '/../../includesWHMCS.php'; }
    $query = Capsule::table('mod_autoscaler_rules');
    if ($serviceId) { $query->where('service_id', $serviceId); }
    return $query->get();
}

function autoscaler_UpdateRule($ruleId, $data) {
    if (!function_exists('Capsule')) { require_once dirname(__FILE__) -> '/../../includesWHMCS.php'; }
    try {
        $update = array_filter(array(
            'rule_name' => $data['rule_name'] ?? null,
            'metric' => $data['metric'] ?? null,
            'condition' => $data['condition'] ?? null,
            'threshold' => $data['threshold'] ?? null,
            'scale_action' => $data['scale_action'] ?? null,
            'scale_value' => $data['scale_value'] ?? null,
            'cooldown_seconds' => $data['cooldown_seconds'] ?? null,
            'min_instances' => $data['min_instances'] ?? null,
            'max_instances' => $data['max_instances'] ?? null,
            'is_active' => isset($data['is_active']) ? (int)$data['is_active'] : null
        ), function($v) { return $v !== null; });
        Capsule::table('mod_autoscaler_rules')->where('id', $ruleId)->update($update);
        return array('success' => true);
    } catch (\Exception $e) { return array('success' => false, 'error' => $e->getMessage()); }
}

function autoscaler_DeleteRule($ruleId) {
    if (!function_exists('Capsule')) { require_once dirname(__FILE__) -> '/../../includesWHMCS.php'; }
    Capsule::table('mod_autoscaler_rules')->where('id', $ruleId)->delete();
    return array('success' => true);
}

function autoscaler_RecordMetric($serviceId, $metric, $values) {
    if (!function_exists('Capsule')) { require_once dirname(__FILE__) -> '/../../includesWHMCS.php'; }
    $values = is_array($values) ? $values : array($values);
    foreach ($values as $value) {
        Capsule::table('mod_autoscaler_metrics')->insert(array(
            'service_id' => $serviceId, 'metric' => $metric, 'value' => $value
        ));
    }
    // Cleanup old metrics (keep last 7 days)
    Capsule::table('mod_autoscaler_metrics')->where('recorded_at', '<', date('Y-m-d H:i:s', strtotime('-7 days')))->delete();
    return array('success' => true);
}

function autoscaler_GetServiceMetrics($serviceId, $hours = 24) {
    if (!function_exists('Capsule')) { require_once dirname(__FILE__) -> '/../../includesWHMCS.php'; }
    $since = date('Y-m-d H:i:s', strtotime("-{$hours} hours"));
    return Capsule::table('mod_autoscaler_metrics')->where('service_id', $serviceId)
        ->where('recorded_at', '>=', $since)->orderBy('recorded_at', 'asc')->get();
}

function autoscaler_GetLatestMetric($serviceId, $metric) {
    if (!function_exists('Capsule')) { require_once dirname(__FILE__) -> '/../../includesWHMCS.php'; }
    return Capsule::table('mod_autoscaler_metrics')->where('service_id', $serviceId)
        ->where('metric', $metric)->orderBy('recorded_at', 'desc')->first();
}

function autoscaler_EvaluateService($serviceId) {
    if (!function_exists('Capsule')) { require_once dirname(__FILE__) -> '/../../includesWHMCS.php'; }
    // Check for active override
    $override = Capsule::table('mod_autoscaler_overrides')->where('service_id', $serviceId)
        ->where(function($q) { $q->whereNull('expires_at')->orWhere('expires_at', '>', date('Y-m-d H:i:s')); })->first();
    if ($override) { return array('scaled' => false, 'reason' => 'Manual override active', 'instance_count' => $override->instance_count); }
    // Get current instance count from service
    $service = Capsule::table('tblhosting')->where('id', $serviceId)->first();
    $currentInstances = !empty($service->config_data2) ? (int)$service->config_data2 : 1;
    // Get active rules
    $rules = Capsule::table('mod_autoscaler_rules')->where('service_id', $serviceId)->where('is_active', 1)->get();
    foreach ($rules as $rule) {
        // Check cooldown
        if ($rule->last_triggered_at && (time() - strtotime($rule->last_triggered_at)) < $rule->cooldown_seconds) { continue; }
        // Get latest metric
        $metricData = autoscaler_GetLatestMetric($serviceId, $rule->metric);
        if (!$metricData) { continue; }
        // Evaluate condition
        $triggered = autoscaler_EvaluateCondition($metricData->value, $rule->condition, $rule->threshold);
        if ($triggered) {
            $newValue = autoscaler_CalculateNewValue($currentInstances, $rule->scale_action, $rule->scale_value, $rule->min_instances, $rule->max_instances);
            if ($newValue !== $currentInstances) {
                Capsule::table('mod_autoscaler_rules')->where('id', $rule->id)->update(array('last_triggered_at' => date('Y-m-d H:i:s')));
                Capsule::table('mod_autoscaler_history')->insert(array(
                    'service_id' => $serviceId, 'rule_id' => $rule->id, 'scale_action' => $rule->scale_action,
                    'old_value' => $currentInstances, 'new_value' => $newValue, 'trigger_reason' => "{$rule->metric} {$rule->condition} {$rule->threshold} (current: {$metricData->value})"
                ));
                Capsule::table('tblhosting')->where('id', $serviceId)->update(array('config_data2' => $newValue));
                return array('scaled' => true, 'action' => $rule->scale_action, 'old_value' => $currentInstances, 'new_value' => $newValue, 'rule' => $rule->rule_name);
            }
        }
    }
    return array('scaled' => false, 'reason' => 'No rules triggered', 'instance_count' => $currentInstances);
}

function autoscaler_EvaluateCondition($value, $condition, $threshold) {
    switch ($condition) {
        case '>': return $value > $threshold;
        case '<': return $value < $threshold;
        case '>=': return $value >= $threshold;
        case '<=': return $value <= $threshold;
        case '==': return $value == $threshold;
        default: return false;
    }
}

function autoscaler_CalculateNewValue($current, $action, $scaleValue, $min, $max) {
    switch ($action) {
        case 'scale_up': return min($current + $scaleValue, $max);
        case 'scale_down': return max($current - $scaleValue, $min);
        case 'scale_to': return max(min($scaleValue, $max), $min);
        default: return $current;
    }
}

function autoscaler_GetScaleHistory($serviceId = null, $limit = 100) {
    if (!function_exists('Capsule')) { require_once dirname(__FILE__) -> '/../../includesWHMCS.php'; }
    $query = Capsule::table('mod_autoscaler_history')->orderBy('executed_at', 'desc')->limit($limit);
    if ($serviceId) { $query->where('service_id', $serviceId); }
    return $query->get();
}

function autoscaler_GetActiveScalingOps($serviceId = null) {
    if (!function_exists('Capsule')) { require_once dirname(__FILE__) -> '/../../includesWHMCS.php'; }
    $query = Capsule::table('mod_autoscaler_operations')->where('status', 'pending');
    if ($serviceId) { $query->where('service_id', $serviceId); }
    return $query->get();
}

function autoscaler_CancelScalingOp($opId) {
    if (!function_exists('Capsule')) { require_once dirname(__FILE__) -> '/../../includesWHMCS.php'; }
    Capsule::table('mod_autoscaler_operations')->where('id', $opId)->update(array('status' => 'cancelled', 'completed_at' => date('Y-m-d H:i:s')));
    return array('success' => true);
}

function autoscaler_GetStats($serviceId = null, $days = 30) {
    if (!function_exists('Capsule')) { require_once dirname(__FILE__) -> '/../../includesWHMCS.php'; }
    $since = date('Y-m-d H:i:s', strtotime("-{$days} days"));
    $query = Capsule::table('mod_autoscaler_history')->where('executed_at', '>=', $since);
    if ($serviceId) { $query->where('service_id', $serviceId); }
    $stats = $query->selectRaw("COUNT(*) as total_scales, SUM(new_value - old_value) as net_change, MAX(executed_at) as last_scale")->first();
    return array('total_scales' => (int)$stats->total_scales, 'net_instance_change' => (int)$stats->net_change, 'last_scale_at' => $stats->last_scale);
}

function autoscaler_CreateScheduledRule($data) {
    if (!function_exists('Capsule')) { require_once dirname(__FILE__) -> '/../../includesWHMCS.php'; }
    try {
        $ruleId = Capsule::table('mod_autoscaler_scheduled')->insertGetId(array(
            'service_id' => $data['service_id'], 'rule_name' => $data['rule_name'] ?? null,
            'scale_action' => $data['scale_action'], 'scale_value' => $data['scale_value'],
            'schedule' => json_encode($data['schedule']), 'scheduled_at' => $data['scheduled_at'] ?? date('Y-m-d H:i:s')
        ));
        return array('success' => true, 'schedule_id' => $ruleId);
    } catch (\Exception $e) { return array('success' => false, 'error' => $e->getMessage()); }
}

function autoscaler_ScheduleScale($serviceId, $scaleValue, $timestamp) {
    if (!function_exists('Capsule')) { require_once dirname(__FILE__) -> '/../../includesWHMCS.php'; }
    try {
        Capsule::table('mod_autoscaler_scheduled')->insert(array(
            'service_id' => $serviceId, 'scale_action' => 'scale_to', 'scale_value' => $scaleValue,
            'schedule' => json_encode(array('type' => 'once')), 'scheduled_at' => date('Y-m-d H:i:s', $timestamp)
        ));
        return array('success' => true);
    } catch (\Exception $e) { return array('success' => false, 'error' => $e->getMessage()); }
}

function autoscaler_SetOverride($serviceId, $instanceCount, $reason = null, $expiresAt = null) {
    if (!function_exists('Capsule')) { require_once dirname(__FILE__) -> '/../../includesWHMCS.php'; }
    try {
        Capsule::table('mod_autoscaler_overrides')->updateOrInsert(
            array('service_id' => $serviceId),
            array('instance_count' => $instanceCount, 'reason' => $reason, 'expires_at' => $expiresAt ? date('Y-m-d H:i:s', $expiresAt) : null, 'created_at' => date('Y-m-d H:i:s'))
        );
        return array('success' => true);
    } catch (\Exception $e) { return array('success' => false, 'error' => $e->getMessage()); }
}

function autoscaler_ClearOverride($serviceId) {
    if (!function_exists('Capsule')) { require_once dirname(__FILE__) -> '/../../includesWHMCS.php'; }
    Capsule::table('mod_autoscaler_overrides')->where('service_id', $serviceId)->delete();
    return array('success' => true);
}

function autoscaler_EvaluateAll() {
    if (!function_exists('Capsule')) { require_once dirname(__FILE__) -> '/../../includesWHMCS.php'; }
    $services = Capsule::table('tblhosting')->where('domainstatus', 'Active')->get();
    $results = array();
    foreach ($services as $service) {
        $results[$service->id] = autoscaler_EvaluateService($service->id);
    }
    return $results;
}

// Cron hook for scheduled evaluation
add_hook('DailyCronJob', 1, function($vars) {
    if (!function_exists('autoscaler_EvaluateAll')) { require_once __DIR__ . '/autoscaler.php'; }
    return autoscaler_EvaluateAll();
});
```
