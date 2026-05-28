# WHMCS Metrics Collector Module

```php
<?php
/**
 * WHMCS Metrics Collector Module
 * 
 * Metrics collection and dashboards with real-time monitoring,
 * performance tracking, and alerting capabilities.
 * 
 * @Author: HiTech Cloud DevKit
 * @Version: 1.0.0
 */

if (!defined("WHMCS")) { die("Direct access prohibited"); }

function metricscollector_MetaData() {
    return array('DisplayName' => 'Metrics Collector', 'APIVersion' => '1.1', 'RequiresServer' => false);
}

function metricscollector_ConfigArray() {
    return array(
        'FriendlyName' => array('Type' => 'System', 'Value' => 'Metrics Collector'),
        'CollectionInterval' => array('Type' => 'dropdown', 'Options' => '60,300,600,3600', 'Default' => '300', 'Description' => 'Collection interval (seconds)'),
        'RetentionDays' => array('Type' => 'text', 'Size' => '10', 'Default' => '90', 'Description' => 'Data retention (days)'),
        'EnableRealTime' => array('Type' => 'yesno', 'Default' => 'yes', 'Description' => 'Enable real-time metrics'),
        'EnablePerformance' => array('Type' => 'yesno', 'Default' => 'yes', 'Description' => 'Track performance metrics'),
        'EnableBusiness' => array('Type' => 'yesno', 'Default' => 'yes', 'Description' => 'Track business metrics'),
        'DashboardRefresh' => array('Type' => 'text', 'Size' => '10', 'Default' => '30', 'Description' => 'Dashboard refresh (seconds)')
    );
}

function metricscollector_activate() {
    try {
        if (!function_exists('createTable')) { require_once dirname(__FILE__) . '/../../includes/modulefunctions.php'; }
        createTable('mod_metricscollector_data', "
            CREATE TABLE `mod_metricscollector_data` (
                `id` BIGINT NOT NULL AUTO_INCREMENT PRIMARY KEY,
                `metric_name` VARCHAR(100) NOT NULL,
                `metric_type` VARCHAR(20) NOT NULL,
                `value` DECIMAL(20,6) NOT NULL,
                `unit` VARCHAR(20) NULL,
                `tags` JSON NULL,
                `timestamp` DATETIME DEFAULT CURRENT_TIMESTAMP,
                INDEX `idx_metric_time` (`metric_name`, `timestamp`),
                INDEX `idx_timestamp` (`timestamp`)
            ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
        ");
        createTable('mod_metricscollector_aggregates', "
            CREATE TABLE `mod_metricscollector_aggregates` (
                `id` BIGINT NOT NULL AUTO_INCREMENT PRIMARY KEY,
                `metric_name` VARCHAR(100) NOT NULL,
                `period` VARCHAR(20) NOT NULL,
                `period_start` DATETIME NOT NULL,
                `period_end` DATETIME NOT NULL,
                `count` BIGINT DEFAULT 0,
                `sum` DECIMAL(20,6) DEFAULT 0,
                `min` DECIMAL(20,6) DEFAULT 0,
                `max` DECIMAL(20,6) DEFAULT 0,
                `avg` DECIMAL(20,6) DEFAULT 0,
                `p50` DECIMAL(20,6) DEFAULT 0,
                `p90` DECIMAL(20,6) DEFAULT 0,
                `p95` DECIMAL(20,6) DEFAULT 0,
                `p99` DECIMAL(20,6) DEFAULT 0,
                INDEX `idx_metric_period` (`metric_name`, `period`, `period_start`)
            ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
        ");
        createTable('mod_metricscollector_dashboards', "
            CREATE TABLE `mod_metricscollector_dashboards` (
                `id` INT NOT NULL AUTO_INCREMENT PRIMARY KEY,
                `dashboard_name` VARCHAR(100) NOT NULL,
                `dashboard_key` VARCHAR(50) UNIQUE NOT NULL,
                `widgets` JSON NOT NULL,
                `layout` JSON NULL,
                `is_default` TINYINT(1) DEFAULT 0,
                `is_public` TINYINT(1) DEFAULT 0,
                `user_id` INT NULL,
                `created_at` DATETIME DEFAULT CURRENT_TIMESTAMP,
                `updated_at` DATETIME DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP
            ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
        ");
        createTable('mod_metricscollector_alerts', "
            CREATE TABLE `mod_metricscollector_alerts` (
                `id` INT NOT NULL AUTO_INCREMENT PRIMARY KEY,
                `alert_name` VARCHAR(100) NOT NULL,
                `metric_name` VARCHAR(100) NOT NULL,
                `condition` VARCHAR(20) NOT NULL,
                `threshold` DECIMAL(20,6) NOT NULL,
                `duration_seconds` INT DEFAULT 0,
                `severity` VARCHAR(20) DEFAULT 'warning',
                `action` VARCHAR(50) NOT NULL,
                `action_config` JSON NULL,
                `is_active` TINYINT(1) DEFAULT 1,
                `last_triggered` DATETIME NULL,
                `trigger_count` INT DEFAULT 0,
                `created_at` DATETIME DEFAULT CURRENT_TIMESTAMP
            ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
        ");
        createTable('mod_metricscollector_health', "
            CREATE TABLE `mod_metricscollector_health` (
                `id` INT NOT NULL AUTO_INCREMENT PRIMARY KEY,
                `check_name` VARCHAR(100) NOT NULL,
                `status` VARCHAR(20) NOT NULL,
                `message` TEXT NULL,
                `details` JSON NULL,
                `checked_at` DATETIME DEFAULT CURRENT_TIMESTAMP,
                INDEX `idx_check_status` (`check_name`, `status`)
            ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
        ");
        return array('status' => 'success', 'description' => 'Metrics Collector module activated.');
    } catch (\Exception $e) { return array('status' => 'error', 'description' => $e->getMessage()); }
}

function metricscollector_deactivate() { return array('status' => 'success', 'description' => 'Module deactivated.'); }

function metricscollector_RecordMetric($name, $value, $type = 'gauge', $unit = null, $tags = array()) {
    if (!function_exists('Capsule')) { require_once dirname(__FILE__) -> '/../../includesWHMCS.php'; }
    Capsule::table('mod_metricscollector_data')->insert(array('metric_name' => $name, 'metric_type' => $type, 'value' => $value, 'unit' => $unit, 'tags' => !empty($tags) ? json_encode($tags) : null));
    return true;
}

function metricscollector_Counter($name, $value = 1, $tags = array()) { return metricscollector_RecordMetric($name, $value, 'counter', 'count', $tags); }
function metricscollector_Gauge($name, $value, $unit = null, $tags = array()) { return metricscollector_RecordMetric($name, $value, 'gauge', $unit, $tags); }
function metricscollector_Histogram($name, $value, $unit = null, $tags = array()) { return metricscollector_RecordMetric($name, $value, 'histogram', $unit, $tags); }
function metricscollector_Timing($name, $valueMs, $tags = array()) { return metricscollector_RecordMetric($name, $valueMs, 'histogram', 'ms', $tags); }

function metricscollector_GetMetric($name, $from = null, $to = null, $interval = null) {
    if (!function_exists('Capsule')) { require_once dirname(__FILE__) -> '/../../includesWHMCS.php'; }
    $from = $from ?? date('Y-m-d H:i:s', strtotime('-24 hours'));
    $to = $to ?? date('Y-m-d H:i:s');
    $query = Capsule::table('mod_metricscollector_data')->where('metric_name', $name)->where('timestamp', '>=', $from)->where('timestamp', '<=', $to)->orderBy('timestamp', 'asc');
    return $query->get();
}

function metricscollector_GetMetricStats($name, $from = null, $to = null) {
    if (!function_exists('Capsule')) { require_once dirname(__FILE__) -> '/../../includesWHMCS.php'; }
    $from = $from ?? date('Y-m-d H:i:s', strtotime('-24 hours'));
    $to = $to ?? date('Y-m-d H:i:s');
    $stats = Capsule::table('mod_metricscollector_data')->where('metric_name', $name)->where('timestamp', '>=', $from)->where('timestamp', '<=', $to)->selectRaw("COUNT(*) as count, SUM(value) as sum, MIN(value) as min, MAX(value) as max, AVG(value) as avg")->first();
    return array('count' => (int)$stats->count, 'sum' => (float)$stats->sum, 'min' => (float)$stats->min, 'max' => (float)$stats->max, 'avg' => (float)$stats->avg);
}

function metricscollector_GetLatestValue($name) {
    if (!function_exists('Capsule')) { require_once dirname(__FILE__) -> '/../../includesWHMCS.php'; }
    $latest = Capsule::table('mod_metricscollector_data')->where('metric_name', $name)->orderBy('timestamp', 'desc')->first();
    return $latest ? array('value' => (float)$latest->value, 'timestamp' => $latest->timestamp, 'unit' => $latest->unit) : null;
}

function metricscollector_GetMetricsList() {
    if (!function_exists('Capsule')) { require_once dirname(__FILE__) -> '/../../includesWHMCS.php'; }
    return Capsule::table('mod_metricscollector_data')->select('metric_name', 'metric_type', 'unit')->groupBy('metric_name', 'metric_type', 'unit')->get();
}

function metricscollector_GetTimeSeries($name, $from = null, $to = null, $bucketCount = 100) {
    if (!function_exists('Capsule')) { require_once dirname(__FILE__) -> '/../../includesWHMCS.php'; }
    $from = $from ?? date('Y-m-d H:i:s', strtotime('-24 hours'));
    $to = $to ?? date('Y-m-d H:i:s');
    $interval = floor((strtotime($to) - strtotime($from)) / $bucketCount);
    $data = Capsule::table('mod_metricscollector_data')->where('metric_name', $name)->where('timestamp', '>=', $from)->where('timestamp', '<=', $to)->selectRaw("FROM_UNIXTIME(FLOOR(UNIX_TIMESTAMP(timestamp)/{$interval})*{$interval}) as bucket, AVG(value) as value")->groupByRaw("bucket")->orderBy('bucket', 'asc')->get();
    return $data;
}

function metricscollector_GetDashboard($key) {
    if (!function_exists('Capsule')) { require_once dirname(__FILE__) -> '/../../includesWHMCS.php'; }
    return Capsule::table('mod_metricscollector_dashboards')->where('dashboard_key', $key)->first();
}

function metricscollector_GetDashboards($userId = null) {
    if (!function_exists('Capsule')) { require_once dirname(__FILE__) -> '/../../includesWHMCS.php'; }
    $query = Capsule::table('mod_metricscollector_dashboards');
    if ($userId) { $query->where(function($q) use ($userId) { $q->where('user_id', $userId)->orWhere('is_public', 1); }); }
    else { $query->where('is_public', 1); }
    return $query->get();
}

function metricscollector_SaveDashboard($data) {
    if (!function_exists('Capsule')) { require_once dirname(__FILE__) -> '/../../includesWHMCS.php'; }
    try {
        $id = Capsule::table('mod_metricscollector_dashboards')->updateOrInsert(
            array('dashboard_key' => $data['dashboard_key']),
            array('dashboard_name' => $data['dashboard_name'], 'widgets' => json_encode($data['widgets']), 'layout' => isset($data['layout']) ? json_encode($data['layout']) : null, 'is_public' => $data['is_public'] ?? 0, 'user_id' => $data['user_id'] ?? null)
        );
        return array('success' => true, 'dashboard_key' => $data['dashboard_key']);
    } catch (\Exception $e) { return array('success' => false, 'error' => $e->getMessage()); }
}

function metricscollector_CreateAlert($data) {
    if (!function_exists('Capsule')) { require_once dirname(__FILE__) -> '/../../includesWHMCS.php'; }
    try {
        $alertId = Capsule::table('mod_metricscollector_alerts')->insertGetId(array(
            'alert_name' => $data['alert_name'], 'metric_name' => $data['metric_name'], 'condition' => $data['condition'],
            'threshold' => $data['threshold'], 'duration_seconds' => $data['duration_seconds'] ?? 0,
            'severity' => $data['severity'] ?? 'warning', 'action' => $data['action'], 'action_config' => json_encode($data['action_config'] ?? array())
        ));
        return array('success' => true, 'alert_id' => $alertId);
    } catch (\Exception $e) { return array('success' => false, 'error' => $e->getMessage()); }
}

function metricscollector_GetAlerts($activeOnly = true) {
    if (!function_exists('Capsule')) { require_once dirname(__FILE__) -> '/../../includesWHMCS.php'; }
    $query = Capsule::table('mod_metricscollector_alerts');
    if ($activeOnly) { $query->where('is_active', 1); }
    return $query->get();
}

function metricscollector_CheckAlerts() {
    if (!function_exists('Capsule')) { require_once dirname(__FILE__) -> '/../../includesWHMCS.php'; }
    $alerts = metricscollector_GetAlerts(true);
    $triggered = array();
    foreach ($alerts as $alert) {
        $latest = metricscollector_GetLatestValue($alert->metric_name);
        if (!$latest) { continue; }
        $conditionMet = false;
        switch ($alert->condition) {
            case '>': $conditionMet = $latest['value'] > $alert->threshold; break;
            case '<': $conditionMet = $latest['value'] < $alert->threshold; break;
            case '>=': $conditionMet = $latest['value'] >= $alert->threshold; break;
            case '<=': $conditionMet = $latest['value'] <= $alert->threshold; break;
            case '==': $conditionMet = $latest['value'] == $alert->threshold; break;
            case '!=': $conditionMet = $latest['value'] != $alert->threshold; break;
        }
        if ($conditionMet) {
            Capsule::table('mod_metricscollector_alerts')->where('id', $alert->id)->update(array('last_triggered' => date('Y-m-d H:i:s'), 'trigger_count' => Capsule::raw('trigger_count + 1')));
            metricscollector_ExecuteAlertAction($alert, $latest);
            $triggered[] = $alert->id;
        }
    }
    return $triggered;
}

function metricscollector_ExecuteAlertAction($alert, $metricValue) {
    $config = json_decode($alert->action_config, true) ?? array();
    switch ($alert->action) {
        case 'email': metricscollector_SendAlertEmail($config['to'] ?? '', $alert->alert_name, $metricValue); break;
        case 'webhook': metricscollector_TriggerWebhook($config['url'] ?? '', $alert, $metricValue); break;
        case 'log': logActivity("Metric Alert: {$alert->alert_name} - {$metricValue['value']}"); break;
    }
}

function metricscollector_SendAlertEmail($to, $alertName, $metricValue) { /* Email logic */ }
function metricscollector_TriggerWebhook($url, $alert, $metricValue) {
    $data = array('alert' => $alert->alert_name, 'metric' => $alert->metric_name, 'value' => $metricValue['value'], 'threshold' => $alert->threshold, 'time' => date('Y-m-d H:i:s'));
    $ch = curl_init($url);
    curl_setopt_array($ch, array(CURLOPT_POST => true, CURLOPT_POSTFIELDS => json_encode($data), CURLOPT_HTTPHEADER => array('Content-Type: application/json'), CURLOPT_RETURNTRANSFER => true));
    curl_exec($ch);
    curl_close($ch);
}

function metricscollector_RecordHealthCheck($checkName, $status, $message = null, $details = null) {
    if (!function_exists('Capsule')) { require_once dirname(__FILE__) -> '/../../includesWHMCS.php'; }
    Capsule::table('mod_metricscollector_health')->insert(array('check_name' => $checkName, 'status' => $status, 'message' => $message, 'details' => $details ? json_encode($details) : null));
}

function metricscollector_GetHealthStatus() {
    if (!function_exists('Capsule')) { require_once dirname(__FILE__) -> '/../../includesWHMCS.php'; }
    $checks = Capsule::table('mod_metricscollector_health')->selectRaw("check_name, status, MAX(checked_at) as last_check")->groupBy('check_name', 'status')->get();
    $issues = Capsule::table('mod_metricscollector_health')->where('status', '!=', 'healthy')->where('checked_at', '>=', date('Y-m-d H:i:s', strtotime('-5 minutes')))->get();
    return array('checks' => $checks, 'issues' => $issues, 'overall_status' => empty($issues) ? 'healthy' : 'degraded');
}

function metricscollector_CollectSystemMetrics() {
    $cpu = sys_getloadavg();
    $mem = memory_get_usage(true);
    $memTotal = memory_get_usage(true);
    metricscollector_Gauge('system.load.1min', $cpu[0], 'load');
    metricscollector_Gauge('system.load.5min', $cpu[1], 'load');
    metricscollector_Gauge('system.load.15min', $cpu[2], 'load');
    metricscollector_Gauge('system.memory.used', $mem, 'bytes');
    metricscollector_Gauge('system.memory.total', $memTotal, 'bytes');
    metricscollector_Gauge('system.disk.usage', disk_free_space('/') / disk_total_space('/') * 100, 'percent');
}

function metricscollector_CollectWHMCSMetrics() {
    if (!function_exists('Capsule')) { require_once dirname(__FILE__) -> '/../../includesWHMCS.php'; }
    metricscollector_Gauge('whmcs.clients.total', Capsule::table('tblclients')->count(), 'count');
    metricscollector_Gauge('whmcs.services.active', Capsule::table('tblhosting')->where('domainstatus', 'Active')->count(), 'count');
    metricscollector_Gauge('whmcs.invoices.unpaid', Capsule::table('tblinvoices')->where('status', 'Unpaid')->count(), 'count');
    metricscollector_Gauge('whmcs.tickets.open', Capsule::table('tbltickets')->whereIn('status', array('Open', 'Answered'))->count(), 'count');
}

function metricscollector_CleanupOldData($retentionDays = 90) {
    if (!function_exists('Capsule')) { require_once dirname(__FILE__) -> '/../../includesWHMCS.php'; }
    $cutoff = date('Y-m-d H:i:s', strtotime("-{$retentionDays} days"));
    return Capsule::table('mod_metricscollector_data')->where('timestamp', '<', $cutoff)->delete();
}

function metricscollector_GetSummary($hours = 24) {
    if (!function_exists('Capsule')) { require_once dirname(__FILE__) -> '/../../includesWHMCS.php'; }
    $since = date('Y-m-d H:i:s', strtotime("-{$hours} hours"));
    $metrics = Capsule::table('mod_metricscollector_data')->where('timestamp', '>=', $since)->selectRaw("metric_name, COUNT(*) as points, MIN(value) as min, MAX(value) as max, AVG(value) as avg")->groupBy('metric_name')->get();
    return $metrics;
}
```

# WHMCS Metrics Collector Module DevKit

## DevKit Structure

```
devkits/whmcs-metrics-collector/
├── metricscollector.php     # Main module file
├── lib/
│   ├── MetricsCollector.php  # Collection engine
│   ├── Aggregator.php        # Aggregation engine
│   ├── DashboardBuilder.php  # Dashboard builder
│   └── AlertEngine.php       # Alert processing
└── templates/
    ├── dashboard.tpl        # Dashboard view
    └── alerts.tpl            # Alerts view
```

## Metric Types

| Type | Description |
|------|-------------|
| gauge | Current value |
| counter | Incrementing value |
| histogram | Distribution |

## Module Functions

| Function | Description |
|----------|-------------|
| `metricscollector_RecordMetric()` | Record raw metric |
| `metricscollector_Counter()` | Increment counter |
| `metricscollector_Gauge()` | Set gauge value |
| `metricscollector_Histogram()` | Record histogram |
| `metricscollector_Timing()` | Record timing |
| `metricscollector_GetMetric()` | Get metric data |
| `metricscollector_GetMetricStats()` | Get statistics |
| `metricscollector_GetLatestValue()` | Get latest value |
| `metricscollector_GetMetricsList()` | List all metrics |
| `metricscollector_GetTimeSeries()` | Get time series |
| `metricscollector_GetDashboard()` | Get dashboard |
| `metricscollector_SaveDashboard()` | Save dashboard |
| `metricscollector_CreateAlert()` | Create alert |
| `metricscollector_CheckAlerts()` | Check alerts |
| `metricscollector_GetHealthStatus()` | Get health status |
| `metricscollector_CollectSystemMetrics()` | Collect system |
| `metricscollector_CollectWHMCSMetrics()` | Collect WHMCS |

## Checklist

```
Pre-Dev:
□ Define metric types
□ Plan retention policy
□ Design dashboard widgets
□ Plan alert conditions

Development:
□ Create metrics tables
□ Implement collection API
□ Add aggregation engine
□ Create dashboards
□ Implement alert engine
□ Add health checks
□ Build admin interface
□ Add chart visualizations

Testing:
□ Test metric recording
□ Verify aggregation
□ Test dashboards
□ Test alerts
□ Verify health checks
```
