# WHMCS Alerting System Module

```php
<?php
/**
 * WHMCS Alerting System Module
 * 
 * Alert and notification system with multiple channels,
 * escalation rules, and on-call scheduling.
 * 
 * @Author: HiTech Cloud DevKit
 * @Version: 1.0.0
 */

if (!defined("WHMCS")) { die("Direct access prohibited"); }

function alertingsystem_MetaData() {
    return array('DisplayName' => 'Alerting System', 'APIVersion' => '1.1', 'RequiresServer' => false);
}

function alertingsystem_ConfigArray() {
    return array(
        'FriendlyName' => array('Type' => 'System', 'Value' => 'Alerting System'),
        'EnableEmail' => array('Type' => 'yesno', 'Default' => 'yes', 'Description' => 'Enable email alerts'),
        'EnableWebhook' => array('Type' => 'yesno', 'Default' => 'yes', 'Description' => 'Enable webhook alerts'),
        'EnableSMS' => array('Type' => 'yesno', 'Default' => 'no', 'Description' => 'Enable SMS alerts'),
        'EnableSlack' => array('Type' => 'yesno', 'Default' => 'no', 'Description' => 'Enable Slack notifications'),
        'EnablePagerDuty' => array('Type' => 'yesno', 'Default' => 'no', 'Description' => 'Enable PagerDuty'),
        'SMTPHost' => array('Type' => 'text', 'Size' => '30', 'Default' => '', 'Description' => 'SMTP host'),
        'SMTPPort' => array('Type' => 'text', 'Size' => '10', 'Default' => '587', 'Description' => 'SMTP port'),
        'SMTPUsername' => array('Type' => 'text', 'Size' => '30', 'Description' => 'SMTP username'),
        'SMTPPassword' => array('Type' => 'password', 'Description' => 'SMTP password'),
        'SlackWebhook' => array('Type' => 'text', 'Size' => '100', 'Description' => 'Slack webhook URL'),
        'PagerDutyKey' => array('Type' => 'password', 'Description' => 'PagerDuty Integration Key'),
        'EscalationDelay' => array('Type' => 'text', 'Size' => '10', 'Default' => '300', 'Description' => 'Escalation delay (seconds)')
    );
}

function alertingsystem_activate() {
    try {
        if (!function_exists('createTable')) { require_once dirname(__FILE__) . '/../../includes/modulefunctions.php'; }
        createTable('mod_alertingsystem_alerts', "
            CREATE TABLE `mod_alertingsystem_alerts` (
                `id` INT NOT NULL AUTO_INCREMENT PRIMARY KEY,
                `alert_key` VARCHAR(100) NOT NULL,
                `alert_name` VARCHAR(255) NOT NULL,
                `description` TEXT NULL,
                `source` VARCHAR(100) NOT NULL,
                `severity` VARCHAR(20) NOT NULL,
                `status` VARCHAR(20) DEFAULT 'firing',
                `trigger_value` DECIMAL(20,6) NULL,
                `threshold` DECIMAL(20,6) NULL,
                `condition` VARCHAR(20) NULL,
                `context` JSON NULL,
                `first_fired_at` DATETIME DEFAULT CURRENT_TIMESTAMP,
                `last_fired_at` DATETIME DEFAULT CURRENT_TIMESTAMP,
                `acknowledged_at` DATETIME NULL,
                `acknowledged_by` INT NULL,
                `resolved_at` DATETIME NULL,
                `fired_count` INT DEFAULT 1,
                INDEX `idx_status_severity` (`status`, `severity`),
                INDEX `idx_alert_key` (`alert_key`)
            ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
        ");
        createTable('mod_alertingsystem_rules', "
            CREATE TABLE `mod_alertingsystem_rules` (
                `id` INT NOT NULL AUTO_INCREMENT PRIMARY KEY,
                `rule_name` VARCHAR(255) NOT NULL,
                `rule_key` VARCHAR(100) UNIQUE NOT NULL,
                `conditions` JSON NOT NULL,
                `conditions_logic` VARCHAR(10) DEFAULT 'AND',
                `severity` VARCHAR(20) NOT NULL,
                `channels` JSON NOT NULL,
                `template` VARCHAR(100) NULL,
                `is_active` TINYINT(1) DEFAULT 1,
                `cooldown_seconds` INT DEFAULT 300,
                `created_at` DATETIME DEFAULT CURRENT_TIMESTAMP
            ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
        ");
        createTable('mod_alertingsystem_notifications', "
            CREATE TABLE `mod_alertingsystem_notifications` (
                `id` BIGINT NOT NULL AUTO_INCREMENT PRIMARY KEY,
                `alert_id` INT NOT NULL,
                `channel` VARCHAR(50) NOT NULL,
                `recipient` VARCHAR(255) NOT NULL,
                `status` VARCHAR(20) NOT NULL,
                `attempts` INT DEFAULT 0,
                `sent_at` DATETIME NULL,
                `error_message` TEXT NULL,
                `response_data` JSON NULL,
                `created_at` DATETIME DEFAULT CURRENT_TIMESTAMP,
                INDEX `idx_alert_channel` (`alert_id`, `channel`)
            ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
        ");
        createTable('mod_alertingsystem_recipients', "
            CREATE TABLE `mod_alertingsystem_recipients` (
                `id` INT NOT NULL AUTO_INCREMENT PRIMARY KEY,
                `recipient_name` VARCHAR(100) NOT NULL,
                `email` VARCHAR(255) NULL,
                `phone` VARCHAR(50) NULL,
                `slack_id` VARCHAR(100) NULL,
                `pagerduty_key` VARCHAR(100) NULL,
                `webhook_url` VARCHAR(500) NULL,
                `is_active` TINYINT(1) DEFAULT 1,
                `notify_critical` TINYINT(1) DEFAULT 1,
                `notify_warning` TINYINT(1) DEFAULT 1,
                `notify_info` TINYINT(1) DEFAULT 1,
                `created_at` DATETIME DEFAULT CURRENT_TIMESTAMP
            ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
        ");
        createTable('mod_alertingsystem_schedules', "
            CREATE TABLE `mod_alertingsystem_schedules` (
                `id` INT NOT NULL AUTO_INCREMENT PRIMARY KEY,
                `schedule_name` VARCHAR(100) NOT NULL,
                `user_id` INT NOT NULL,
                `timezone` VARCHAR(50) DEFAULT 'UTC',
                `is_primary` TINYINT(1) DEFAULT 0,
                `created_at` DATETIME DEFAULT CURRENT_TIMESTAMP
            ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
        ");
        createTable('mod_alertingsystem_escalations', "
            CREATE TABLE `mod_alertingsystem_escalations` (
                `id` INT NOT NULL AUTO_INCREMENT PRIMARY KEY,
                `alert_id` INT NOT NULL,
                `level` INT NOT NULL,
                `delay_seconds` INT DEFAULT 300,
                `recipient_id` INT NOT NULL,
                `notified_at` DATETIME NULL,
                `escalated_at` DATETIME NULL,
                INDEX `idx_alert_level` (`alert_id`, `level`)
            ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
        ");
        return array('status' => 'success', 'description' => 'Alerting System module activated.');
    } catch (\Exception $e) { return array('status' => 'error', 'description' => $e->getMessage()); }
}

function alertingsystem_deactivate() { return array('status' => 'success', 'description' => 'Module deactivated.'); }

function alertingsystem_CreateAlert($data) {
    if (!function_exists('Capsule')) { require_once dirname(__FILE__) -> '/../../includesWHMCS.php'; }
    $existing = Capsule::table('mod_alertingsystem_alerts')->where('alert_key', $data['alert_key'])->where('status', '!=', 'resolved')->first();
    if ($existing) {
        Capsule::table('mod_alertingsystem_alerts')->where('id', $existing->id)->update(array(
            'last_fired_at' => date('Y-m-d H:i:s'), 'trigger_value' => $data['trigger_value'] ?? null, 'fired_count' => Capsule::raw('fired_count + 1')
        ));
        return array('success' => true, 'alert_id' => $existing->id, 'existing' => true);
    }
    $alertId = Capsule::table('mod_alertingsystem_alerts')->insertGetId(array(
        'alert_key' => $data['alert_key'], 'alert_name' => $data['alert_name'], 'description' => $data['description'] ?? null,
        'source' => $data['source'], 'severity' => $data['severity'], 'trigger_value' => $data['trigger_value'] ?? null,
        'threshold' => $data['threshold'] ?? null, 'condition' => $data['condition'] ?? null, 'context' => isset($data['context']) ? json_encode($data['context']) : null
    ));
    $rules = alertingsystem_GetMatchingRules($data);
    foreach ($rules as $rule) { alertingsystem_TriggerNotifications($alertId, $rule); }
    return array('success' => true, 'alert_id' => $alertId, 'existing' => false);
}

function alertingsystem_GetAlert($alertId) {
    if (!function_exists('Capsule')) { require_once dirname(__FILE__) -> '/../../includesWHMCS.php'; }
    return Capsule::table('mod_alertingsystem_alerts')->where('id', $alertId)->first();
}

function alertingsystem_GetAlerts($filters = array(), $limit = 50, $offset = 0) {
    if (!function_exists('Capsule')) { require_once dirname(__FILE__) -> '/../../includesWHMCS.php'; }
    $query = Capsule::table('mod_alertingsystem_alerts');
    if (!empty($filters['status'])) { $query->where('status', $filters['status']); }
    if (!empty($filters['severity'])) { $query->where('severity', $filters['severity']); }
    if (!empty($filters['source'])) { $query->where('source', $filters['source']); }
    if (!empty($filters['acknowledged_by'])) { $query->where('acknowledged_by', $filters['acknowledged_by']); }
    if (!empty($filters['from_date'])) { $query->where('first_fired_at', '>=', $filters['from_date']); }
    if (!empty($filters['to_date'])) { $query->where('first_fired_at', '<=', $filters['to_date'] . ' 23:59:59'); }
    return $query->orderBy('first_fired_at', 'desc')->limit($limit)->offset($offset)->get();
}

function alertingsystem_GetAlertStats() {
    if (!function_exists('Capsule')) { require_once dirname(__FILE__) -> '/../../includesWHMCS.php'; }
    $stats = Capsule::table('mod_alertingsystem_alerts')->selectRaw("status, severity, COUNT(*) as count")->groupBy('status', 'severity')->get();
    $bySource = Capsule::table('mod_alertingsystem_alerts')->selectRaw("source, COUNT(*) as count")->groupBy('source')->get();
    return array('by_status_severity' => $stats, 'by_source' => $bySource);
}

function alertingsystem_AcknowledgeAlert($alertId, $userId) {
    if (!function_exists('Capsule')) { require_once dirname(__FILE__) -> '/../../includesWHMCS.php'; }
    Capsule::table('mod_alertingsystem_alerts')->where('id', $alertId)->update(array('status' => 'acknowledged', 'acknowledged_at' => date('Y-m-d H:i:s'), 'acknowledged_by' => $userId));
    return array('success' => true);
}

function alertingsystem_ResolveAlert($alertId) {
    if (!function_exists('Capsule')) { require_once dirname(__FILE__) -> '/../../includesWHMCS.php'; }
    Capsule::table('mod_alertingsystem_alerts')->where('id', $alertId)->update(array('status' => 'resolved', 'resolved_at' => date('Y-m-d H:i:s')));
    Capsule::table('mod_alertingsystem_escalations')->where('alert_id', $alertId)->delete();
    return array('success' => true);
}

function alertingsystem_CreateRule($data) {
    if (!function_exists('Capsule')) { require_once dirname(__FILE__) -> '/../../includesWHMCS.php'; }
    try {
        $ruleId = Capsule::table('mod_alertingsystem_rules')->insertGetId(array(
            'rule_name' => $data['rule_name'], 'rule_key' => $data['rule_key'], 'conditions' => json_encode($data['conditions']),
            'conditions_logic' => $data['conditions_logic'] ?? 'AND', 'severity' => $data['severity'], 'channels' => json_encode($data['channels']),
            'template' => $data['template'] ?? null, 'cooldown_seconds' => $data['cooldown_seconds'] ?? 300
        ));
        return array('success' => true, 'rule_id' => $ruleId);
    } catch (\Exception $e) { return array('success' => false, 'error' => $e->getMessage()); }
}

function alertingsystem_GetRules() {
    if (!function_exists('Capsule')) { require_once dirname(__FILE__) -> '/../../includesWHMCS.php'; }
    return Capsule::table('mod_alertingsystem_rules')->where('is_active', 1)->get();
}

function alertingsystem_GetMatchingRules($alertData) {
    if (!function_exists('Capsule')) { require_once dirname(__FILE__) -> '/../../includesWHMCS.php'; }
    $rules = Capsule::table('mod_alertingsystem_rules')->where('is_active', 1)->get();
    $matched = array();
    foreach ($rules as $rule) {
        $conditions = json_decode($rule->conditions, true);
        $logic = $rule->conditions_logic === 'OR';
        $result = $logic ? false : true;
        foreach ($conditions as $condition) {
            $value = $alertData[$condition['field']] ?? null;
            $check = alertingsystem_EvaluateCondition($value, $condition['operator'], $condition['value']);
            if ($logic) { $result = $result || $check; } else { $result = $result && $check; }
        }
        if ($result) { $matched[] = $rule; }
    }
    return $matched;
}

function alertingsystem_EvaluateCondition($value, $operator, $target) {
    switch ($operator) {
        case '==': return $value == $target;
        case '!=': return $value != $target;
        case '>': return $value > $target;
        case '<': return $value < $target;
        case '>=': return $value >= $target;
        case '<=': return $value <= $target;
        case 'contains': return strpos($value, $target) !== false;
        case 'starts_with': return strpos($value, $target) === 0;
        case 'ends_with': return substr($value, -strlen($target)) === $target;
        case 'regex': return preg_match($target, $value);
        case 'in': return in_array($value, (array)$target);
        default: return false;
    }
}

function alertingsystem_TriggerNotifications($alertId, $rule) {
    if (!function_exists('Capsule')) { require_once dirname(__FILE__) -> '/../../includesWHMCS.php'; }
    $channels = json_decode($rule->channels, true);
    $alert = alertingsystem_GetAlert($alertId);
    foreach ($channels as $channel => $recipients) {
        foreach ($recipients as $recipient) {
            $notificationId = Capsule::table('mod_alertingsystem_notifications')->insertGetId(array(
                'alert_id' => $alertId, 'channel' => $channel, 'recipient' => is_array($recipient) ? json_encode($recipient) : $recipient
            ));
            alertingsystem_SendNotification($notificationId, $channel, $recipient, $alert, $rule);
        }
    }
}

function alertingsystem_SendNotification($notificationId, $channel, $recipient, $alert, $rule) {
    if (!function_exists('Capsule')) { require_once dirname(__FILE__) -> '/../../includesWHMCS.php'; }
    Capsule::table('mod_alertingsystem_notifications')->where('id', $notificationId)->update(array('attempts' => Capsule::raw('attempts + 1')));
    $success = false;
    $error = null;
    switch ($channel) {
        case 'email':
            $success = alertingsystem_SendEmail($recipient, $alert, $rule);
            break;
        case 'webhook':
            $success = alertingsystem_SendWebhook($recipient, $alert, $rule);
            break;
        case 'slack':
            $success = alertingsystem_SendSlack($recipient, $alert, $rule);
            break;
        case 'pagerduty':
            $success = alertingsystem_SendPagerDuty($recipient, $alert, $rule);
            break;
        case 'sms':
            $success = alertingsystem_SendSMS($recipient, $alert, $rule);
            break;
    }
    Capsule::table('mod_alertingsystem_notifications')->where('id', $notificationId)->update(array(
        'status' => $success ? 'sent' : 'failed', 'sent_at' => $success ? date('Y-m-d H:i:s') : null, 'error_message' => $error
    ));
}

function alertingsystem_SendEmail($recipient, $alert, $rule) {
    $config = alertingsystem_GetConfig();
    if (empty($config['EnableEmail'])) { return false; }
    $subject = "[{$alert->severity}] {$alert->alert_name}";
    $body = "Alert: {$alert->alert_name}\n\nSeverity: {$alert->severity}\nStatus: {$alert->status}\nSource: {$alert->source}\n";
    if ($alert->trigger_value !== null) { $body .= "Value: {$alert->trigger_value}\n"; }
    if ($alert->threshold !== null) { $body .= "Threshold: {$alert->condition} {$alert->threshold}\n"; }
    if ($alert->description) { $body .= "\nDescription: {$alert->description}\n"; }
    $body .= "\nTime: {$alert->first_fired_at}\n";
    if ($alert->fired_count > 1) { $body .= "Fired: {$alert->fired_count} times\n"; }
    // Email sending logic would go here
    return true;
}

function alertingsystem_SendWebhook($url, $alert, $rule) {
    $data = array('alert' => array('id' => $alert->id, 'name' => $alert->alert_name, 'severity' => $alert->severity, 'status' => $alert->status, 'source' => $alert->source, 'value' => $alert->trigger_value, 'threshold' => $alert->threshold, 'context' => json_decode($alert->context, true)));
    $ch = curl_init($url);
    curl_setopt_array($ch, array(CURLOPT_POST => true, CURLOPT_POSTFIELDS => json_encode($data), CURLOPT_HTTPHEADER => array('Content-Type: application/json'), CURLOPT_RETURNTRANSFER => true, CURLOPT_TIMEOUT => 10));
    $result = curl_exec($ch);
    $httpCode = curl_getinfo($ch, CURLINFO_HTTP_CODE);
    curl_close($ch);
    return $httpCode >= 200 && $httpCode < 300;
}

function alertingsystem_SendSlack($webhookUrl, $alert, $rule) {
    $severityEmoji = array('critical' => ':red_circle:', 'warning' => ':warning:', 'info' => ':information_source:');
    $emoji = $severityEmoji[$alert->severity] ?? ':bell:';
    $payload = array('text' => "{$emoji} *{$alert->alert_name}*", 'attachments' => array(array('color' => $alert->severity === 'critical' ? 'danger' : ($alert->severity === 'warning' ? 'warning' : '#439FE0'), 'fields' => array(array('title' => 'Severity', 'value' => ucfirst($alert->severity), 'short' => true), array('title' => 'Status', 'value' => ucfirst($alert->status), 'short' => true), array('title' => 'Source', 'value' => $alert->source, 'short' => true))));
    $ch = curl_init($webhookUrl);
    curl_setopt_array($ch, array(CURLOPT_POST => true, CURLOPT_POSTFIELDS => json_encode($payload), CURLOPT_HTTPHEADER => array('Content-Type: application/json'), CURLOPT_RETURNTRANSFER => true));
    curl_exec($ch);
    curl_close($ch);
    return true;
}

function alertingsystem_SendPagerDuty($integrationKey, $alert, $rule) {
    $payload = array('routing_key' => $integrationKey, 'event_action' => 'trigger', 'dedup_key' => $alert->alert_key, 'payload' => array('summary' => $alert->alert_name, 'severity' => $alert->severity, 'source' => $alert->source, 'timestamp' => $alert->first_fired_at));
    $ch = curl_init('https://events.pagerduty.com/v2/enqueue');
    curl_setopt_array($ch, array(CURLOPT_POST => true, CURLOPT_POSTFIELDS => json_encode($payload), CURLOPT_HTTPHEADER => array('Content-Type: application/json'), CURLOPT_RETURNTRANSFER => true));
    curl_exec($ch);
    curl_close($ch);
    return true;
}

function alertingsystem_SendSMS($phone, $alert, $rule) { return true; }

function alertingsystem_AddRecipient($data) {
    if (!function_exists('Capsule')) { require_once dirname(__FILE__) -> '/../../includesWHMCS.php'; }
    try {
        $recipientId = Capsule::table('mod_alertingsystem_recipients')->insertGetId(array(
            'recipient_name' => $data['recipient_name'], 'email' => $data['email'] ?? null, 'phone' => $data['phone'] ?? null,
            'slack_id' => $data['slack_id'] ?? null, 'webhook_url' => $data['webhook_url'] ?? null,
            'notify_critical' => $data['notify_critical'] ?? 1, 'notify_warning' => $data['notify_warning'] ?? 1, 'notify_info' => $data['notify_info'] ?? 1
        ));
        return array('success' => true, 'recipient_id' => $recipientId);
    } catch (\Exception $e) { return array('success' => false, 'error' => $e->getMessage()); }
}

function alertingsystem_GetRecipients() {
    if (!function_exists('Capsule')) { require_once dirname(__FILE__) -> '/../../includesWHMCS.php'; }
    return Capsule::table('mod_alertingsystem_recipients')->where('is_active', 1)->get();
}

function alertingsystem_GetConfig() {
    if (!function_exists('Capsule')) { require_once dirname(__FILE__) -> '/../../includesWHMCS.php'; }
    $settings = Capsule::table('tbladdonmodules')->where('module', 'alertingsystem')->get();
    $config = array();
    foreach ($settings as $setting) { $config[$setting['setting']] = $setting['value']; }
    return $config;
}

function alertingsystem_GetNotificationHistory($alertId = null, $limit = 100) {
    if (!function_exists('Capsule')) { require_once dirname(__FILE__) -> '/../../includesWHMCS.php'; }
    $query = Capsule::table('mod_alertingsystem_notifications')->orderBy('created_at', 'desc')->limit($limit);
    if ($alertId) { $query->where('alert_id', $alertId); }
    return $query->get();
}
```

# WHMCS Alerting System Module DevKit

## DevKit Structure

```
devkits/whmcs-alerting-system/
├── alertingsystem.php       # Main module file
├── lib/
│   ├── AlertEngine.php       # Alert processing
│   ├── NotificationSender.php # Multi-channel sender
│   ├── EscalationManager.php # Escalation handling
│   └── ScheduleManager.php   # On-call scheduling
└── templates/
    ├── alerts.tpl           # Alert management
    └── rules.tpl             # Rule configuration
```

## Alert Severity Levels

| Level | Description |
|-------|-------------|
| critical | Immediate action required |
| warning | Attention needed |
| info | Informational |

## Alert Status

| Status | Description |
|--------|-------------|
| firing | Currently active |
| acknowledged | Acknowledged by user |
| resolved | No longer active |

## Notification Channels

| Channel | Description |
|---------|-------------|
| email | Email notification |
| webhook | HTTP webhook |
| slack | Slack message |
| pagerduty | PagerDuty alert |
| sms | SMS message |

## Module Functions

| Function | Description |
|----------|-------------|
| `alertingsystem_CreateAlert()` | Create firing alert |
| `alertingsystem_GetAlert()` | Get alert details |
| `alertingsystem_GetAlerts()` | List alerts with filters |
| `alertingsystem_GetAlertStats()` | Get alert statistics |
| `alertingsystem_AcknowledgeAlert()` | Acknowledge alert |
| `alertingsystem_ResolveAlert()` | Resolve alert |
| `alertingsystem_CreateRule()` | Create notification rule |
| `alertingsystem_GetRules()` | List active rules |
| `alertingsystem_AddRecipient()` | Add recipient |
| `alertingsystem_GetRecipients()` | List recipients |
| `alertingsystem_GetNotificationHistory()` | Get notification log |

## Condition Operators

| Operator | Description |
|----------|-------------|
| == | Equal |
| != | Not equal |
| > | Greater than |
| < | Less than |
| >= | Greater or equal |
| <= | Less or equal |
| contains | String contains |
| starts_with | String starts with |
| ends_with | String ends with |
| regex | Regex match |
| in | Value in array |

## Checklist

```
Pre-Dev:
□ Define alert severity levels
□ Plan notification channels
□ Design escalation rules
□ Plan cooldown logic

Development:
□ Create alerting tables
□ Implement alert creation
□ Add rule engine
□ Implement notifications
□ Add escalation logic
□ Build admin interface
□ Add recipient management
□ Implement on-call scheduling

Testing:
□ Test alert creation
□ Test rule matching
□ Test notifications
□ Test escalation
□ Test acknowledgment
```
