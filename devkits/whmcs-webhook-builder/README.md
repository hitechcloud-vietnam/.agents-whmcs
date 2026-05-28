# WHMCS Webhook Builder Module

```php
<?php
/**
 * WHMCS Webhook Builder Module
 * 
 * Webhook builder and manager for automation
 * and integrations.
 * 
 * @Author: HiTech Cloud DevKit
 * @Version: 1.0.0
 */

if (!defined("WHMCS")) { die("Direct access prohibited"); }

function webhookbuilder_MetaData() {
    return array('DisplayName' => 'Webhook Builder', 'APIVersion' => '1.1', 'RequiresServer' => false);
}

function webhookbuilder_ConfigArray() {
    return array(
        'FriendlyName' => array('Type' => 'System', 'Value' => 'Webhook Builder'),
        'EnableWebhookLogging' => array('Type' => 'yesno', 'Default' => 'yes', 'Description' => 'Log all webhook calls'),
        'MaxRetries' => array('Type' => 'text', 'Size' => '10', 'Default' => '3', 'Description' => 'Default max retries'),
        'TimeoutSeconds' => array('Type' => 'text', 'Size' => '10', 'Default' => '30', 'Description' => 'Default timeout (seconds)'),
        'RetryBackoff' => array('Type' => 'dropdown', 'Options' => 'linear,exponential,fixed', 'Default' => 'exponential', 'Description' => 'Retry backoff type'),
        'EnableEncryption' => array('Type' => 'yesno', 'Default' => 'yes', 'Description' => 'Enable payload encryption'),
        'HmacSecret' => array('Type' => 'password', 'Description' => 'HMAC signature secret'),
        'RateLimitPerHour' => array('Type' => 'text', 'Size' => '10', 'Default' => '1000', 'Description' => 'Rate limit per hour'),
        'EnableMonitoring' => array('Type' => 'yesno', 'Default' => 'yes', 'Description' => 'Enable monitoring'),
        'AlertOnFailure' => array('Type' => 'yesno', 'Default' => 'yes', 'Description' => 'Alert on failures'),
        'FailureAlertEmail' => array('Type' => 'text', 'Size' => '50', 'Description' => 'Alert email address')
    );
}

function webhookbuilder_activate() {
    try {
        if (!function_exists('createTable')) { require_once dirname(__FILE__) . '/../../includes/modulefunctions.php'; }
        createTable('mod_webhookbuilder_webhooks', "
            CREATE TABLE `mod_webhookbuilder_webhooks` (
                `id` INT NOT NULL AUTO_INCREMENT PRIMARY KEY,
                `webhook_key` VARCHAR(100) UNIQUE NOT NULL,
                `webhook_name` VARCHAR(255) NOT NULL,
                `description` TEXT NULL,
                `trigger_type` VARCHAR(20) NOT NULL,
                `trigger_event` VARCHAR(100) NULL,
                `endpoint_url` VARCHAR(500) NOT NULL,
                `method` VARCHAR(10) DEFAULT 'POST',
                `headers` JSON NULL,
                `body_template` TEXT NULL,
                `authentication` JSON NULL,
                `retry_enabled` TINYINT(1) DEFAULT 1,
                `max_retries` INT DEFAULT 3,
                `timeout_seconds` INT DEFAULT 30,
                `is_active` TINYINT(1) DEFAULT 1,
                `is_paused` TINYINT(1) DEFAULT 0,
                `last_triggered_at` DATETIME NULL,
                `last_status` VARCHAR(20) NULL,
                `success_rate` DECIMAL(5,2) DEFAULT 100.00,
                `created_by` INT NULL,
                `created_at` DATETIME DEFAULT CURRENT_TIMESTAMP,
                `updated_at` DATETIME DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
                INDEX `idx_trigger` (`trigger_type`, `trigger_event`),
                INDEX `idx_active` (`is_active`)
            ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
        ");
        createTable('mod_webhookbuilder_logs', "
            CREATE TABLE `mod_webhookbuilder_logs` (
                `id` BIGINT NOT NULL AUTO_INCREMENT PRIMARY KEY,
                `webhook_id` INT NOT NULL,
                `trigger_type` VARCHAR(20) NOT NULL,
                `trigger_data` JSON NULL,
                `request_url` VARCHAR(500) NOT NULL,
                `request_method` VARCHAR(10) NOT NULL,
                `request_headers` JSON NULL,
                `request_body` TEXT NULL,
                `response_code` INT NULL,
                `response_body` TEXT NULL,
                `response_time_ms` INT NULL,
                `status` VARCHAR(20) DEFAULT 'success',
                `error_message` TEXT NULL,
                `retry_count` INT DEFAULT 0,
                `triggered_by` INT NULL,
                `triggered_at` DATETIME DEFAULT CURRENT_TIMESTAMP,
                INDEX `idx_webhook_time` (`webhook_id`, `triggered_at`),
                INDEX `idx_status` (`status`)
            ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
        ");
        createTable('mod_webhookbuilder_variables', "
            CREATE TABLE `mod_webhookbuilder_variables` (
                `id` INT NOT NULL AUTO_INCREMENT PRIMARY KEY,
                `webhook_id` INT NOT NULL,
                `variable_key` VARCHAR(100) NOT NULL,
                `variable_path` VARCHAR(255) NOT NULL,
                `default_value` VARCHAR(255) NULL,
                `created_at` DATETIME DEFAULT CURRENT_TIMESTAMP,
                UNIQUE KEY `unique_webhook_var` (`webhook_id`, `variable_key`)
            ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
        ");
        createTable('mod_webhookbuilder_filters', "
            CREATE TABLE `mod_webhookbuilder_filters` (
                `id` INT NOT NULL AUTO_INCREMENT PRIMARY KEY,
                `webhook_id` INT NOT NULL,
                `field` VARCHAR(100) NOT NULL,
                `operator` VARCHAR(20) NOT NULL,
                `value` TEXT NOT NULL,
                `logic` VARCHAR(10) DEFAULT 'AND',
                INDEX `idx_webhook` (`webhook_id`)
            ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
        ");
        createTable('mod_webhookbuilder_schedules', "
            CREATE TABLE `mod_webhookbuilder_schedules` (
                `id` INT NOT NULL AUTO_INCREMENT PRIMARY KEY,
                `webhook_id` INT UNIQUE NOT NULL,
                `cron_expression` VARCHAR(100) NOT NULL,
                `last_run` DATETIME NULL,
                `next_run` DATETIME NULL,
                `is_active` TINYINT(1) DEFAULT 1,
                INDEX `idx_next_run` (`next_run`)
            ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
        ");
        return array('status' => 'success', 'description' => 'Webhook Builder module activated.');
    } catch (\Exception $e) { return array('status' => 'error', 'description' => $e->getMessage()); }
}

function webhookbuilder_deactivate() { return array('status' => 'success', 'description' => 'Module deactivated.'); }

function webhookbuilder_CreateWebhook($data) {
    if (!function_exists('Capsule')) { require_once dirname(__FILE__) -> '/../../includesWHMCS.php'; }
    try {
        $webhookId = Capsule::table('mod_webhookbuilder_webhooks')->insertGetId(array(
            'webhook_key' => $data['webhook_key'], 'webhook_name' => $data['webhook_name'],
            'description' => $data['description'] ?? null, 'trigger_type' => $data['trigger_type'],
            'trigger_event' => $data['trigger_event'] ?? null, 'endpoint_url' => $data['endpoint_url'],
            'method' => $data['method'] ?? 'POST', 'headers' => isset($data['headers']) ? json_encode($data['headers']) : null,
            'body_template' => $data['body_template'] ?? null, 'authentication' => isset($data['authentication']) ? json_encode($data['authentication']) : null,
            'retry_enabled' => $data['retry_enabled'] ?? 1, 'max_retries' => $data['max_retries'] ?? 3,
            'timeout_seconds' => $data['timeout_seconds'] ?? 30, 'created_by' => $data['created_by'] ?? null
        ));
        if (!empty($data['variables'])) {
            foreach ($data['variables'] as $var) {
                Capsule::table('mod_webhookbuilder_variables')->insert(array('webhook_id' => $webhookId, 'variable_key' => $var['key'], 'variable_path' => $var['path'], 'default_value' => $var['default'] ?? null));
            }
        }
        if (!empty($data['filters'])) {
            foreach ($data['filters'] as $filter) {
                Capsule::table('mod_webhookbuilder_filters')->insert(array('webhook_id' => $webhookId, 'field' => $filter['field'], 'operator' => $filter['operator'], 'value' => $filter['value'], 'logic' => $filter['logic'] ?? 'AND'));
            }
        }
        return array('success' => true, 'webhook_id' => $webhookId);
    } catch (\Exception $e) { return array('success' => false, 'error' => $e->getMessage()); }
}

function webhookbuilder_GetWebhook($webhookId) {
    if (!function_exists('Capsule')) { require_once dirname(__FILE__) -> '/../../includesWHMCS.php'; }
    $webhook = Capsule::table('mod_webhookbuilder_webhooks')->where('id', $webhookId)->first();
    if ($webhook) {
        $webhook->headers = json_decode($webhook->headers, true);
        $webhook->authentication = json_decode($webhook->authentication, true);
    }
    return $webhook;
}

function webhookbuilder_GetWebhooks($filters = array()) {
    if (!function_exists('Capsule')) { require_once dirname(__FILE__) -> '/../../includesWHMCS.php'; }
    $query = Capsule::table('mod_webhookbuilder_webhooks')->where('is_active', 1);
    if (!empty($filters['trigger_type'])) { $query->where('trigger_type', $filters['trigger_type']); }
    if (!empty($filters['trigger_event'])) { $query->where('trigger_event', $filters['trigger_event']); }
    if (isset($filters['is_paused'])) { $query->where('is_paused', $filters['is_paused']); }
    return $query->get();
}

function webhookbuilder_UpdateWebhook($webhookId, $data) {
    if (!function_exists('Capsule')) { require_once dirname(__FILE__) -> '/../../includesWHMCS.php'; }
    try {
        $update = array_filter(array(
            'webhook_name' => $data['webhook_name'] ?? null,
            'description' => $data['description'] ?? null,
            'endpoint_url' => $data['endpoint_url'] ?? null,
            'method' => $data['method'] ?? null,
            'headers' => isset($data['headers']) ? json_encode($data['headers']) : null,
            'body_template' => $data['body_template'] ?? null,
            'authentication' => isset($data['authentication']) ? json_encode($data['authentication']) : null,
            'retry_enabled' => isset($data['retry_enabled']) ? (int)$data['retry_enabled'] : null,
            'max_retries' => $data['max_retries'] ?? null,
            'timeout_seconds' => $data['timeout_seconds'] ?? null
        ), function($v) { return $v !== null; });
        Capsule::table('mod_webhookbuilder_webhooks')->where('id', $webhookId)->update($update);
        return array('success' => true);
    } catch (\Exception $e) { return array('success' => false, 'error' => $e->getMessage()); }
}

function webhookbuilder_DeleteWebhook($webhookId) {
    if (!function_exists('Capsule')) { require_once dirname(__FILE__) -> '/../../includesWHMCS.php'; }
    Capsule::table('mod_webhookbuilder_webhooks')->where('id', $webhookId)->update(array('is_active' => 0));
    return array('success' => true);
}

function webhookbuilder_PauseWebhook($webhookId) {
    if (!function_exists('Capsule')) { require_once dirname(__FILE__) -> '/../../includesWHMCS.php'; }
    Capsule::table('mod_webhookbuilder_webhooks')->where('id', $webhookId)->update(array('is_paused' => 1));
    return array('success' => true);
}

function webhookbuilder_ResumeWebhook($webhookId) {
    if (!function_exists('Capsule')) { require_once dirname(__FILE__) -> '/../../includesWHMCS.php'; }
    Capsule::table('mod_webhookbuilder_webhooks')->where('id', $webhookId)->update(array('is_paused' => 0));
    return array('success' => true);
}

function webhookbuilder_TriggerWebhook($webhookId, $data = array()) {
    if (!function_exists('Capsule')) { require_once dirname(__FILE__) -> '/../../includesWHMCS.php'; }
    $webhook = webhookbuilder_GetWebhook($webhookId);
    if (!$webhook) { return array('success' => false, 'error' => 'Webhook not found'); }
    if ($webhook->is_paused) { return array('success' => false, 'error' => 'Webhook is paused'); }
    return webhookbuilder_ExecuteWebhook($webhook, $data, 'manual');
}

function webhookbuilder_TestWebhook($webhookId) {
    if (!function_exists('Capsule')) { require_once dirname(__FILE__) -> '/../../includesWHMCS.php'; }
    $webhook = webhookbuilder_GetWebhook($webhookId);
    if (!$webhook) { return array('success' => false, 'error' => 'Webhook not found'); }
    $testData = array('test' => true, 'timestamp' => date('Y-m-d H:i:s'));
    return webhookbuilder_ExecuteWebhook($webhook, $testData, 'test');
}

function webhookbuilder_ExecuteWebhook($webhook, $data, $triggerType = 'manual') {
    $startTime = microtime(true);
    $config = webhookbuilder_GetConfig();
    $body = $webhook->body_template ?? '{}';
    foreach ($data as $key => $value) {
        $body = str_replace('{{' . $key . '}}', is_array($value) ? json_encode($value) : $value, $body);
    }
    $headers = $webhook->headers ?? array();
    if (!empty($config['HmacSecret'])) {
        $signature = hash_hmac('sha256', $body, $config['HmacSecret']);
        $headers['X-Signature'] = $signature;
    }
    $headers['X-Webhook-Trigger'] = $triggerType;
    $headers['X-Timestamp'] = time();
    $ch = curl_init($webhook->endpoint_url);
    curl_setopt_array($ch, array(CURLOPT_RETURNTRANSFER => true, CURLOPT_TIMEOUT => $webhook->timeout_seconds, CURLOPT_CUSTOMREQUEST => $webhook->method, CURLOPT_POSTFIELDS => $body, CURLOPT_HTTPHEADER => array_map(function($k, $v) { return $k . ': ' . $v; }, array_keys($headers), array_values($headers))));
    $response = curl_exec($ch);
    $httpCode = curl_getinfo($ch, CURLINFO_HTTP_CODE);
    $responseTime = curl_getinfo($ch, CURLINFO_TOTAL_TIME) * 1000;
    curl_close($ch);
    $success = $httpCode >= 200 && $httpCode < 300;
    $logId = Capsule::table('mod_webhookbuilder_logs')->insertGetId(array(
        'webhook_id' => $webhook->id, 'trigger_type' => $triggerType, 'trigger_data' => json_encode($data),
        'request_url' => $webhook->endpoint_url, 'request_method' => $webhook->method,
        'request_headers' => json_encode($headers), 'request_body' => $body,
        'response_code' => $httpCode, 'response_body' => substr($response, 0, 10000),
        'response_time_ms' => (int)$responseTime, 'status' => $success ? 'success' : 'failed',
        'error_message' => $success ? null : 'HTTP ' . $httpCode
    ));
    Capsule::table('mod_webhookbuilder_webhooks')->where('id', $webhook->id)->update(array(
        'last_triggered_at' => date('Y-m-d H:i:s'), 'last_status' => $success ? 'success' : 'failed'
    ));
    webhookbuilder_UpdateSuccessRate($webhook->id);
    return array('success' => $success, 'log_id' => $logId, 'status_code' => $httpCode, 'response_body' => substr($response, 0, 1000), 'duration_ms' => round($responseTime, 2));
}

function webhookbuilder_GetWebhookStatus($webhookId) {
    if (!function_exists('Capsule')) { require_once dirname(__FILE__) -> '/../../includesWHMCS.php'; }
    $webhook = webhookbuilder_GetWebhook($webhookId);
    if (!$webhook) { return null; }
    $recentLogs = Capsule::table('mod_webhookbuilder_logs')->where('webhook_id', $webhookId)->orderBy('triggered_at', 'desc')->limit(10)->get();
    $successCount = 0;
    foreach ($recentLogs as $log) { if ($log->status === 'success') { $successCount++; } }
    return array('webhook' => $webhook, 'is_active' => $webhook->is_active, 'is_paused' => $webhook->is_paused, 'last_triggered' => $webhook->last_triggered_at, 'last_status' => $webhook->last_status, 'success_rate' => $webhook->success_rate, 'recent_success_count' => $successCount, 'recent_total' => count($recentLogs));
}

function webhookbuilder_GetWebhookHistory($webhookId, $limit = 50) {
    if (!function_exists('Capsule')) { require_once dirname(__FILE__) -> '/../../includesWHMCS.php'; }
    return Capsule::table('mod_webhookbuilder_logs')->where('webhook_id', $webhookId)->orderBy('triggered_at', 'desc')->limit($limit)->get();
}

function webhookbuilder_GetLogs($filters = array()) {
    if (!function_exists('Capsule')) { require_once dirname(__FILE__) -> '/../../includesWHMCS.php'; }
    $query = Capsule::table('mod_webhookbuilder_logs');
    if (!empty($filters['webhook_id'])) { $query->where('webhook_id', $filters['webhook_id']); }
    if (!empty($filters['status'])) { $query->where('status', $filters['status']); }
    if (!empty($filters['from_date'])) { $query->where('triggered_at', '>=', $filters['from_date']); }
    if (!empty($filters['to_date'])) { $query->where('triggered_at', '<=', $filters['to_date']); }
    return $query->orderBy('triggered_at', 'desc')->limit($filters['limit'] ?? 100)->get();
}

function webhookbuilder_RetryWebhook($logId) {
    if (!function_exists('Capsule')) { require_once dirname(__FILE__) -> '/../../includesWHMCS.php'; }
    $log = Capsule::table('mod_webhookbuilder_logs')->where('id', $logId)->first();
    if (!$log) { return array('success' => false, 'error' => 'Log not found'); }
    $webhook = webhookbuilder_GetWebhook($log->webhook_id);
    if (!$webhook || $webhook->is_paused) { return array('success' => false, 'error' => 'Webhook unavailable'); }
    $data = json_decode($log->trigger_data ?? '{}', true);
    Capsule::table('mod_webhookbuilder_logs')->where('id', $logId)->update(array('retry_count' => $log->retry_count + 1));
    return webhookbuilder_ExecuteWebhook($webhook, $data, 'retry');
}

function webhookbuilder_AddVariable($webhookId, $key, $path, $default = null) {
    if (!function_exists('Capsule')) { require_once dirname(__FILE__) -> '/../../includesWHMCS.php'; }
    try {
        Capsule::table('mod_webhookbuilder_variables')->updateOrInsert(array('webhook_id' => $webhookId, 'variable_key' => $key), array('variable_path' => $path, 'default_value' => $default));
        return array('success' => true);
    } catch (\Exception $e) { return array('success' => false, 'error' => $e->getMessage()); }
}

function webhookbuilder_AddFilter($webhookId, $filter) {
    if (!function_exists('Capsule')) { require_once dirname(__FILE__) -> '/../../includesWHMCS.php'; }
    try {
        Capsule::table('mod_webhookbuilder_filters')->insert(array('webhook_id' => $webhookId, 'field' => $filter['field'], 'operator' => $filter['operator'], 'value' => $filter['value'], 'logic' => $filter['logic'] ?? 'AND'));
        return array('success' => true);
    } catch (\Exception $e) { return array('success' => false, 'error' => $e->getMessage()); }
}

function webhookbuilder_CheckFilters($webhookId, $data) {
    if (!function_exists('Capsule')) { require_once dirname(__FILE__) -> '/../../includesWHMCS.php'; }
    $filters = Capsule::table('mod_webhookbuilder_filters')->where('webhook_id', $webhookId)->get();
    if (empty($filters)) { return true; }
    $result = true;
    foreach ($filters as $index => $filter) {
        $value = webhookbuilder_GetNestedValue($data, $filter->field);
        $check = webhookbuilder_EvaluateFilter($value, $filter->operator, $filter->value);
        if ($index === 0) { $result = $check; } else if ($filter->logic === 'OR') { $result = $result || $check; } else { $result = $result && $check; }
    }
    return $result;
}

function webhookbuilder_GetNestedValue($data, $path) {
    $keys = explode('.', $path);
    $value = $data;
    foreach ($keys as $key) { $value = is_array($value) ? ($value[$key] ?? null) : null; if ($value === null) { break; } }
    return $value;
}

function webhookbuilder_EvaluateFilter($value, $operator, $target) {
    switch ($operator) {
        case '==': return $value == $target;
        case '!=': return $value != $target;
        case '>': return $value > $target;
        case '<': return $value < $target;
        case '>=': return $value >= $target;
        case '<=': return md5($value) <= md5($target);
        case 'contains': return strpos($value, $target) !== false;
        case 'in': return in_array($value, is_array($target) ? $target : explode(',', $target));
        default: return false;
    }
}

function webhookbuilder_UpdateSuccessRate($webhookId) {
    if (!function_exists('Capsule')) { require_once dirname(__FILE__) -> '/../../includesWHMCS.php'; }
    $logs = Capsule::table('mod_webhookbuilder_logs')->where('webhook_id', $webhookId)->orderBy('triggered_at', 'desc')->limit(100)->get();
    if (empty($logs)) { return; }
    $successCount = 0;
    foreach ($logs as $log) { if ($log->status === 'success') { $successCount++; } }
    $rate = ($successCount / count($logs)) * 100;
    Capsule::table('mod_webhookbuilder_webhooks')->where('id', $webhookId)->update(array('success_rate' => $rate));
}

function webhookbuilder_CreateScheduledWebhook($data) {
    if (!function_exists('Capsule')) { require_once dirname(__FILE__) -> '/../../includesWHMCS.php'; }
    try {
        $webhookId = Capsule::table('mod_webhookbuilder_webhooks')->insertGetId(array(
            'webhook_key' => $data['webhook_key'], 'webhook_name' => $data['webhook_name'],
            'trigger_type' => 'schedule', 'endpoint_url' => $data['endpoint_url'],
            'method' => $data['method'] ?? 'POST', 'headers' => isset($data['headers']) ? json_encode($data['headers']) : null,
            'body_template' => $data['body_template'] ?? null, 'created_by' => $data['created_by'] ?? null
        ));
        Capsule::table('mod_webhookbuilder_schedules')->insert(array(
            'webhook_id' => $webhookId, 'cron_expression' => $data['schedule'], 'next_run' => webhookbuilder_CalculateNextCronRun($data['schedule'])
        ));
        return array('success' => true, 'webhook_id' => $webhookId);
    } catch (\Exception $e) { return array('success' => false, 'error' => $e->getMessage()); }
}

function webhookbuilder_CalculateNextCronRun($cronExpression) {
    // Simple calculation - for production use a proper cron library
    return date('Y-m-d H:i:s', strtotime('+1 day'));
}

function webhookbuilder_GetConfig() {
    if (!function_exists('Capsule')) { require_once dirname(__FILE__) -> '/../../includesWHMCS.php'; }
    $settings = Capsule::table('tbladdonmodules')->where('module', 'webhookbuilder')->get();
    $config = array();
    foreach ($settings as $setting) { $config[$setting['setting']] = $setting['value']; }
    return $config;
}

// Hook for automatic triggering
add_hook('ClientAdd', 1, function($vars) {
    $webhooks = Capsule::table('mod_webhookbuilder_webhooks')->where('trigger_type', 'event')->where('trigger_event', 'ClientAdd')->where('is_active', 1)->where('is_paused', 0)->get();
    foreach ($webhooks as $webhook) {
        webhookbuilder_ExecuteWebhook($webhook, $vars, 'event');
    }
});
```
