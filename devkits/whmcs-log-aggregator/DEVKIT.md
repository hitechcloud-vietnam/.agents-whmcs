# WHMCS Log Aggregator Module

```php
<?php
/**
 * WHMCS Log Aggregator Module
 * 
 * Centralized logging system with multiple log sources,
 * log levels, filtering, and export capabilities.
 * 
 * @Author: HiTech Cloud DevKit
 * @Version: 1.0.0
 */

if (!defined("WHMCS")) { die("Direct access prohibited"); }

function logaggregator_MetaData() {
    return array('DisplayName' => 'Log Aggregator', 'APIVersion' => '1.1', 'RequiresServer' => false);
}

function logaggregator_ConfigArray() {
    return array(
        'FriendlyName' => array('Type' => 'System', 'Value' => 'Log Aggregator'),
        'DefaultLevel' => array('Type' => 'dropdown', 'Options' => 'debug,info,notice,warning,error,critical,alert,emergency', 'Default' => 'info', 'Description' => 'Default log level'),
        'EnableFileLog' => array('Type' => 'yesno', 'Default' => 'yes', 'Description' => 'Enable file logging'),
        'EnableDBLog' => array('Type' => 'yesno', 'Default' => 'yes', 'Description' => 'Enable database logging'),
        'LogPath' => array('Type' => 'text', 'Size' => '50', 'Default' => '/storage/logs/custom', 'Description' => 'Log file path'),
        'MaxFileSize' => array('Type' => 'text', 'Size' => '10', 'Default' => '104857600', 'Description' => 'Max file size (bytes)'),
        'RetentionDays' => array('Type' => 'text', 'Size' => '10', 'Default' => '30', 'Description' => 'Log retention (days)'),
        'EnableSyslog' => array('Type' => 'yesno', 'Default' => 'no', 'Description' => 'Enable syslog forwarding'),
        'SyslogHost' => array('Type' => 'text', 'Size' => '30', 'Default' => '', 'Description' => 'Syslog host'),
        'SyslogPort' => array('Type' => 'text', 'Size' => '10', 'Default' => '514', 'Description' => 'Syslog port')
    );
}

function logaggregator_activate() {
    try {
        if (!function_exists('createTable')) { require_once dirname(__FILE__) . '/../../includes/modulefunctions.php'; }
        createTable('mod_logaggregator_logs', "
            CREATE TABLE `mod_logaggregator_logs` (
                `id` BIGINT NOT NULL AUTO_INCREMENT PRIMARY KEY,
                `log_level` VARCHAR(20) NOT NULL,
                `source` VARCHAR(100) NOT NULL,
                `category` VARCHAR(50) NULL,
                `message` TEXT NOT NULL,
                `context` JSON NULL,
                `extra` JSON NULL,
                `user_id` INT NULL,
                `ip_address` VARCHAR(45) NULL,
                `user_agent` VARCHAR(500) NULL,
                `request_id` VARCHAR(64) NULL,
                `session_id` VARCHAR(64) NULL,
                `created_at` DATETIME DEFAULT CURRENT_TIMESTAMP,
                INDEX `idx_level_source` (`log_level`, `source`),
                INDEX `idx_category` (`category`),
                INDEX `idx_user` (`user_id`),
                INDEX `idx_created` (`created_at`),
                INDEX `idx_request` (`request_id`)
            ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
        ");
        createTable('mod_logaggregator_sources', "
            CREATE TABLE `mod_logaggregator_sources` (
                `id` INT NOT NULL AUTO_INCREMENT PRIMARY KEY,
                `source_key` VARCHAR(100) UNIQUE NOT NULL,
                `source_name` VARCHAR(255) NOT NULL,
                `description` TEXT NULL,
                `min_level` VARCHAR(20) DEFAULT 'debug',
                `is_active` TINYINT(1) DEFAULT 1,
                `log_to_file` TINYINT(1) DEFAULT 1,
                `log_to_db` TINYINT(1) DEFAULT 1,
                `log_to_syslog` TINYINT(1) DEFAULT 0,
                `created_at` DATETIME DEFAULT CURRENT_TIMESTAMP
            ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
        ");
        createTable('mod_logaggregator_archives', "
            CREATE TABLE `mod_logaggregator_archives` (
                `id` INT NOT NULL AUTO_INCREMENT PRIMARY KEY,
                `filename` VARCHAR(255) NOT NULL,
                `filepath` VARCHAR(500) NOT NULL,
                `size_bytes` BIGINT DEFAULT 0,
                `log_count` INT DEFAULT 0,
                `date_from` DATE NOT NULL,
                `date_to` DATE NOT NULL,
                `compressed` TINYINT(1) DEFAULT 1,
                `archived_at` DATETIME DEFAULT CURRENT_TIMESTAMP
            ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
        ");
        createTable('mod_logaggregator_rules', "
            CREATE TABLE `mod_logaggregator_rules` (
                `id` INT NOT NULL AUTO_INCREMENT PRIMARY KEY,
                `rule_name` VARCHAR(100) NOT NULL,
                `match_source` VARCHAR(100) NULL,
                `match_level` VARCHAR(20) NULL,
                `match_pattern` VARCHAR(255) NULL,
                `action` VARCHAR(20) NOT NULL,
                `action_value` TEXT NULL,
                `is_active` TINYINT(1) DEFAULT 1,
                `created_at` DATETIME DEFAULT CURRENT_TIMESTAMP
            ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
        ");
        $defaultSources = array(array('key' => 'core', 'name' => 'WHMCS Core', 'desc' => 'Core WHMCS logging'), array('key' => 'module', 'name' => 'Modules', 'desc' => 'Module activity logs'), array('key' => 'api', 'name' => 'API', 'desc' => 'API request logs'), array('key' => 'auth', 'name' => 'Authentication', 'desc' => 'Login/logout logs'), array('key' => 'payment', 'name' => 'Payments', 'desc' => 'Payment processing logs'), array('key' => 'admin', 'name' => 'Admin Actions', 'desc' => 'Admin activity logs'));
        foreach ($defaultSources as $src) { Capsule::table('mod_logaggregator_sources')->insert(array('source_key' => $src['key'], 'source_name' => $src['name'], 'description' => $src['desc'])); }
        return array('status' => 'success', 'description' => 'Log Aggregator module activated.');
    } catch (\Exception $e) { return array('status' => 'error', 'description' => $e->getMessage()); }
}

function logaggregator_deactivate() { return array('status' => 'success', 'description' => 'Module deactivated.'); }

function logaggregator_Log($level, $message, $source = 'core', $context = array(), $extra = array()) {
    if (!function_exists('Capsule')) { require_once dirname(__FILE__) -> '/../../includesWHMCS.php'; }
    $config = logaggregator_GetConfig();
    $levels = array('debug' => 0, 'info' => 1, 'notice' => 2, 'warning' => 3, 'error' => 4, 'critical' => 5, 'alert' => 6, 'emergency' => 7);
    $minLevel = $config['DefaultLevel'] ?? 'info';
    if (($levels[$level] ?? 0) < ($levels[$minLevel] ?? 1)) { return false; }
    $requestId = $_SERVER['HTTP_X_REQUEST_ID'] ?? logaggregator_GenerateRequestId();
    $logData = array('log_level' => $level, 'source' => $source, 'message' => $message, 'context' => !empty($context) ? json_encode($context) : null, 'extra' => !empty($extra) ? json_encode($extra) : null, 'ip_address' => $_SERVER['REMOTE_ADDR'] ?? null, 'user_agent' => substr($_SERVER['HTTP_USER_AGENT'] ?? '', 0, 500), 'request_id' => $requestId, 'session_id' => session_id() ?: null);
    if ($config['EnableDBLog']) { Capsule::table('mod_logaggregator_logs')->insert($logData); }
    if ($config['EnableFileLog']) { logaggregator_WriteToFile($level, $source, $message, $context, $requestId); }
    if ($config['EnableSyslog'] && !empty($config['SyslogHost'])) { logaggregator_SendToSyslog($level, $source, $message, $config); }
    logaggregator_ApplyRules($level, $source, $message, $context);
    return true;
}

function logaggregator_Debug($message, $source = 'core', $context = array()) { return logaggregator_Log('debug', $message, $source, $context); }
function logaggregator_Info($message, $source = 'core', $context = array()) { return logaggregator_Log('info', $message, $source, $context); }
function logaggregator_Notice($message, $source = 'core', $context = array()) { return logaggregator_Log('notice', $message, $source, $context); }
function logaggregator_Warning($message, $source = 'core', $context = array()) { return logaggregator_Log('warning', $message, $source, $context); }
function logaggregator_Error($message, $source = 'core', $context = array()) { return logaggregator_Log('error', $message, $source, $context); }
function logaggregator_Critical($message, $source = 'core', $context = array()) { return logaggregator_Log('critical', $message, $source, $context); }
function logaggregator_Alert($message, $source = 'core', $context = array()) { return logaggregator_Log('alert', $message, $source, $context); }
function logaggregator_Emergency($message, $source = 'core', $context = array()) { return logaggregator_Log('emergency', $message, $source, $context); }

function logaggregator_GetLogs($filters = array(), $limit = 100, $offset = 0) {
    if (!function_exists('Capsule')) { require_once dirname(__FILE__) -> '/../../includesWHMCS.php'; }
    $query = Capsule::table('mod_logaggregator_logs');
    if (!empty($filters['level'])) { $query->where('log_level', $filters['level']); }
    if (!empty($filters['source'])) { $query->where('source', $filters['source']); }
    if (!empty($filters['category'])) { $query->where('category', $filters['category']); }
    if (!empty($filters['user_id'])) { $query->where('user_id', $filters['user_id']); }
    if (!empty($filters['from_date'])) { $query->where('created_at', '>=', $filters['from_date']); }
    if (!empty($filters['to_date'])) { $query->where('created_at', '<=', $filters['to_date'] . ' 23:59:59'); }
    if (!empty($filters['search'])) { $query->where('message', 'like', '%' . $filters['search'] . '%'); }
    if (!empty($filters['request_id'])) { $query->where('request_id', $filters['request_id']); }
    return $query->orderBy('created_at', 'desc')->limit($limit)->offset($offset)->get();
}

function logaggregator_GetLogStats($days = 7) {
    if (!function_exists('Capsule')) { require_once dirname(__FILE__) -> '/../../includesWHMCS.php'; }
    $since = date('Y-m-d H:i:s', strtotime("-{$days} days"));
    $stats = Capsule::table('mod_logaggregator_logs')->where('created_at', '>=', $since)->selectRaw("log_level, COUNT(*) as count, source")->groupBy('log_level', 'source')->get();
    $daily = Capsule::table('mod_logaggregator_logs')->where('created_at', '>=', $since)->selectRaw("DATE(created_at) as date, COUNT(*) as count")->groupByRaw("DATE(created_at)")->get();
    return array('by_level_source' => $stats, 'by_day' => $daily);
}

function logaggregator_GetLogCount($filters = array()) {
    if (!function_exists('Capsule')) { require_once dirname(__FILE__) -> '/../../includesWHMCS.php'; }
    $query = Capsule::table('mod_logaggregator_logs');
    if (!empty($filters['level'])) { $query->where('log_level', $filters['level']); }
    if (!empty($filters['source'])) { $query->where('source', $filters['source']); }
    if (!empty($filters['from_date'])) { $query->where('created_at', '>=', $filters['from_date']); }
    if (!empty($filters['to_date'])) { $query->where('created_at', '<=', $filters['to_date'] . ' 23:59:59'); }
    return $query->count();
}

function logaggregator_ClearLogs($beforeDate = null) {
    if (!function_exists('Capsule')) { require_once dirname(__FILE__) -> '/../../includesWHMCS.php'; }
    $before = $beforeDate ?? date('Y-m-d', strtotime('-30 days'));
    return Capsule::table('mod_logaggregator_logs')->where('created_at', '<', $before)->delete();
}

function logaggregator_GetSources() {
    if (!function_exists('Capsule')) { require_once dirname(__FILE__) -> '/../../includesWHMCS.php'; }
    return Capsule::table('mod_logaggregator_sources')->where('is_active', 1)->get();
}

function logaggregator_AddSource($data) {
    if (!function_exists('Capsule')) { require_once dirname(__FILE__) -> '/../../includesWHMCS.php'; }
    try {
        Capsule::table('mod_logaggregator_sources')->insert(array('source_key' => $data['source_key'], 'source_name' => $data['source_name'], 'description' => $data['description'] ?? null, 'min_level' => $data['min_level'] ?? 'debug'));
        return array('success' => true);
    } catch (\Exception $e) { return array('success' => false, 'error' => $e->getMessage()); }
}

function logaggregator_AddRule($data) {
    if (!function_exists('Capsule')) { require_once dirname(__FILE__) -> '/../../includesWHMCS.php'; }
    try {
        Capsule::table('mod_logaggregator_rules')->insert(array('rule_name' => $data['rule_name'], 'match_source' => $data['match_source'] ?? null, 'match_level' => $data['match_level'] ?? null, 'match_pattern' => $data['match_pattern'] ?? null, 'action' => $data['action'], 'action_value' => $data['action_value'] ?? null));
        return array('success' => true);
    } catch (\Exception $e) { return array('success' => false, 'error' => $e->getMessage()); }
}

function logaggregator_GetRules() {
    if (!function_exists('Capsule')) { require_once dirname(__FILE__) -> '/../../includesWHMCS.php'; }
    return Capsule::table('mod_logaggregator_rules')->where('is_active', 1)->get();
}

function logaggregator_ArchiveLogs($fromDate, $toDate) {
    if (!function_exists('Capsule')) { require_once dirname(__FILE__) -> '/../../includesWHMCS.php'; }
    $logs = Capsule::table('mod_logaggregator_logs')->where('created_at', '>=', $fromDate)->where('created_at', '<=', $toDate . ' 23:59:59')->get();
    if (empty($logs)) { return array('success' => false, 'error' => 'No logs found in date range'); }
    $filename = 'logs_' . date('Y-m-d_His') . '_' . substr(md5(uniqid()), 0, 8) . '.json.gz';
    $logPath = logaggregator_GetConfig()['LogPath'] ?? ROOTDIR . '/storage/logs/custom';
    $filepath = rtrim($logPath, '/') . '/archives/' . $filename;
    $dir = dirname($filepath);
    if (!is_dir($dir)) { mkdir($dir, 0755, true); }
    $jsonData = json_encode($logs);
    $compressed = gzcompress($jsonData, 9);
    file_put_contents($filepath, $compressed);
    $archiveId = Capsule::table('mod_logaggregator_archives')->insertGetId(array('filename' => $filename, 'filepath' => $filepath, 'size_bytes' => strlen($compressed), 'log_count' => count($logs), 'date_from' => $fromDate, 'date_to' => $toDate));
    Capsule::table('mod_logaggregator_logs')->where('created_at', '>=', $fromDate)->where('created_at', '<=', $toDate . ' 23:59:59')->delete();
    return array('success' => true, 'archive_id' => $archiveId, 'filename' => $filename, 'logs_archived' => count($logs));
}

function logaggregator_GetArchives() {
    if (!function_exists('Capsule')) { require_once dirname(__FILE__) -> '/../../includesWHMCS.php'; }
    return Capsule::table('mod_logaggregator_archives')->orderBy('archived_at', 'desc')->get();
}

function logaggregator_DownloadArchive($archiveId) {
    if (!function_exists('Capsule')) { require_once dirname(__FILE__) -> '/../../includesWHMCS.php'; }
    $archive = Capsule::table('mod_logaggregator_archives')->where('id', $archiveId)->first();
    if (!$archive || !file_exists($archive->filepath)) { return array('success' => false, 'error' => 'Archive not found'); }
    return array('success' => true, 'filepath' => $archive->filepath, 'filename' => $archive->filename);
}

function logaggregator_ExportLogs($format = 'json', $filters = array()) {
    if (!function_exists('Capsule')) { require_once dirname(__FILE__) -> '/../../includesWHMCS.php'; }
    $logs = logaggregator_GetLogs($filters, 100000, 0);
    $filename = 'logs_export_' . date('Y-m-d_His') . '.' . $format;
    $tempPath = sys_get_temp_dir() . '/' . $filename;
    switch ($format) {
        case 'csv':
            $handle = fopen($tempPath, 'w');
            fputcsv($handle, array('ID', 'Level', 'Source', 'Category', 'Message', 'Context', 'IP', 'User ID', 'Timestamp'));
            foreach ($logs as $log) { fputcsv($handle, array($log->id, $log->log_level, $log->source, $log->category, $log->message, $log->context, $log->ip_address, $log->user_id, $log->created_at)); }
            fclose($handle);
            break;
        case 'json':
        default:
            file_put_contents($tempPath, json_encode($logs, JSON_PRETTY_PRINT));
    }
    return array('success' => true, 'filepath' => $tempPath, 'filename' => $filename);
}

// Internal functions
function logaggregator_GetConfig() {
    if (!function_exists('Capsule')) { require_once dirname(__FILE__) -> '/../../includesWHMCS.php'; }
    $settings = Capsule::table('tbladdonmodules')->where('module', 'logaggregator')->get();
    $config = array();
    foreach ($settings as $setting) { $config[$setting['setting']] = $setting['value']; }
    return $config;
}

function logaggregator_WriteToFile($level, $source, $message, $context, $requestId) {
    $config = logaggregator_GetConfig();
    $logPath = rtrim($config['LogPath'] ?? ROOTDIR . '/storage/logs/custom', '/');
    if (!is_dir($logPath)) { mkdir($logPath, 0755, true); }
    $filename = $logPath . '/app_' . date('Y-m-d') . '.log';
    $timestamp = date('Y-m-d H:i:s');
    $contextStr = !empty($context) ? ' ' . json_encode($context) : '';
    $line = "[{$timestamp}] {$level}.{$source}: {$message}{$contextStr} [request:{$requestId}]" . PHP_EOL;
    $handle = fopen($filename, 'a');
    if ($handle) { fwrite($handle, $line); fclose($handle); }
    if ($config['MaxFileSize'] && filesize($filename) > $config['MaxFileSize']) {
        $archiveName = $filename . '.' . time() . '.gz';
        $compressed = gzopen($archiveName, 'w9');
        gzwrite($compressed, file_get_contents($filename));
        gzclose($compressed);
        unlink($filename);
    }
}

function logaggregator_SendToSyslog($level, $source, $message, $config) {
    $host = $config['SyslogHost'];
    $port = $config['SyslogPort'] ?? 514;
    $syslogMessage = "<" . (LOG_INFO + logaggregator_GetSyslogLevel($level)) . ">WHMCS {$source}: {$message}";
    $socket = @fsockopen("udp://{$host}", $port, $errno, $errstr, 1);
    if ($socket) { fwrite($socket, $syslogMessage); fclose($socket); }
}

function logaggregator_GetSyslogLevel($level) {
    $map = array('emergency' => 0, 'alert' => 1, 'critical' => 2, 'error' => 3, 'warning' => 4, 'notice' => 5, 'info' => 6, 'debug' => 7);
    return $map[$level] ?? 6;
}

function logaggregator_ApplyRules($level, $source, $message, $context) {
    $rules = Capsule::table('mod_logaggregator_rules')->where('is_active', 1)->get();
    foreach ($rules as $rule) {
        $match = true;
        if ($rule->match_source && $rule->match_source !== $source) { $match = false; }
        if ($rule->match_level && $rule->match_level !== $level) { $match = false; }
        if ($rule->match_pattern && !preg_match($rule->match_pattern, $message)) { $match = false; }
        if ($match) {
            switch ($rule->action) {
                case 'email': logaggregator_SendAlertEmail($rule->action_value, $level, $source, $message, $context); break;
                case 'webhook': logaggregator_TriggerWebhook($rule->action_value, $level, $source, $message, $context); break;
                case 'slack': logaggregator_SendToSlack($rule->action_value, $level, $source, $message); break;
            }
        }
    }
}

function logaggregator_SendAlertEmail($to, $level, $source, $message, $context) { /* Email sending logic */ }
function logaggregator_TriggerWebhook($url, $level, $source, $message, $context) { /* Webhook logic */ }
function logaggregator_SendToSlack($webhookUrl, $level, $source, $message) { /* Slack logic */ }
function logaggregator_GenerateRequestId() { return bin2hex(random_bytes(16)); }
```

# WHMCS Log Aggregator Module DevKit

## DevKit Structure

```
devkits/whmcs-log-aggregator/
├── logaggregator.php        # Main module file
├── lib/
│   ├── FileHandler.php       # File logging
│   ├── DBHandler.php         # Database logging
│   ├── SyslogHandler.php     # Syslog forwarding
│   └── RuleEngine.php        # Alert rules
└── templates/
    ├── admin.tpl            # Admin dashboard
    └── logs.tpl             # Log viewer
```

## Log Levels

| Level | Value | Description |
|-------|-------|-------------|
| debug | 0 | Debug messages |
| info | 1 | Informational |
| notice | 2 | Normal but significant |
| warning | 3 | Warning conditions |
| error | 4 | Error conditions |
| critical | 5 | Critical conditions |
| alert | 6 | Action required |
| emergency | 7 | System unusable |

## Module Functions

| Function | Description |
|----------|-------------|
| `logaggregator_Log()` | Log message with level |
| `logaggregator_Debug()` | Log debug message |
| `logaggregator_Info()` | Log info message |
| `logaggregator_Warning()` | Log warning message |
| `logaggregator_Error()` | Log error message |
| `logaggregator_Critical()` | Log critical message |
| `logaggregator_GetLogs()` | Query logs with filters |
| `logaggregator_GetLogStats()` | Get log statistics |
| `logaggregator_GetLogCount()` | Count logs |
| `logaggregator_ClearLogs()` | Delete old logs |
| `logaggregator_GetSources()` | Get log sources |
| `logaggregator_AddRule()` | Add alert rule |
| `logaggregator_ArchiveLogs()` | Archive logs |
| `logaggregator_ExportLogs()` | Export logs |

## Checklist

```
Pre-Dev:
□ Define log levels
□ Plan log sources
□ Design retention policy
□ Plan export formats

Development:
□ Create log tables
□ Implement logging functions
□ Add file logging
□ Add DB logging
□ Implement syslog forwarding
□ Create alert rules engine
□ Add log archiving
□ Implement export (CSV/JSON)
□ Build admin interface
□ Add filters and search

Testing:
□ Test all log levels
□ Verify file logging
□ Verify DB logging
□ Test syslog forwarding
□ Test alert rules
□ Test archiving
□ Test export
□ Verify retention cleanup
```
