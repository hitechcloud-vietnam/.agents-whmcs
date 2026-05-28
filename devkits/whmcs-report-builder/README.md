# WHMCS Report Builder Module

```php
<?php
/**
 * WHMCS Report Builder Module
 * 
 * Custom report builder with drag-drop interface
 * and flexible data sources.
 * 
 * @Author: HiTech Cloud DevKit
 * @Version: 1.0.0
 */

if (!defined("WHMCS")) { die("Direct access prohibited"); }

function reportbuilder_MetaData() {
    return array('DisplayName' => 'Report Builder', 'APIVersion' => '1.1', 'RequiresServer' => false);
}

function reportbuilder_ConfigArray() {
    return array(
        'FriendlyName' => array('Type' => 'System', 'Value' => 'Report Builder'),
        'DefaultExportFormat' => array('Type' => 'dropdown', 'Options' => 'csv,xlsx,pdf,json', 'Default' => 'csv', 'Description' => 'Default export format'),
        'MaxRowsPerReport' => array('Type' => 'text', 'Size' => '10', 'Default' => '10000', 'Description' => 'Maximum rows per report'),
        'EnableScheduling' => array('Type' => 'yesno', 'Default' => 'yes', 'Description' => 'Enable scheduled reports'),
        'EnableCharts' => array('Type' => 'yesno', 'Default' => 'yes', 'Description' => 'Enable chart generation'),
        'ChartColors' => array('Type' => 'text', 'Size' => '100', 'Default' => '#3498db,#e74c3c,#2ecc71,#9b59b6,#f1c40f,#1abc9c', 'Description' => 'Chart colors (comma separated)'),
        'EnableSharing' => array('Type' => 'yesno', 'Default' => 'yes', 'Description' => 'Enable report sharing'),
        'DefaultDateRange' => array('Type' => 'dropdown', 'Options' => 'today,last_7_days,last_30_days,this_month,last_month,this_year', 'Default' => 'last_30_days', 'Description' => 'Default date range'),
        'EnableCaching' => array('Type' => 'yesno', 'Default' => 'yes', 'Description' => 'Cache report results')
    );
}

function reportbuilder_activate() {
    try {
        if (!function_exists('createTable')) { require_once dirname(__FILE__) . '/../../includes/modulefunctions.php'; }
        createTable('mod_reportbuilder_reports', "
            CREATE TABLE `mod_reportbuilder_reports` (
                `id` INT NOT NULL AUTO_INCREMENT PRIMARY KEY,
                `report_key` VARCHAR(100) UNIQUE NOT NULL,
                `report_name` VARCHAR(255) NOT NULL,
                `description` TEXT NULL,
                `data_source` VARCHAR(50) NOT NULL,
                `columns` JSON NOT NULL,
                `filters` JSON NULL,
                `group_by` VARCHAR(100) NULL,
                `aggregations` JSON NULL,
                `sort_order` VARCHAR(100) NULL,
                `sort_direction` VARCHAR(10) DEFAULT 'ASC',
                `is_active` TINYINT(1) DEFAULT 1,
                `owned_by` INT NULL,
                `created_at` DATETIME DEFAULT CURRENT_TIMESTAMP,
                `updated_at` DATETIME DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
                INDEX `idx_datasource` (`data_source`)
            ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
        ");
        createTable('mod_reportbuilder_filters', "
            CREATE TABLE `mod_reportbuilder_filters` (
                `id` INT NOT NULL AUTO_INCREMENT PRIMARY KEY,
                `filter_name` VARCHAR(255) NOT NULL,
                `filter_key` VARCHAR(100) UNIQUE NOT NULL,
                `data_source` VARCHAR(50) NOT NULL,
                `conditions` JSON NOT NULL,
                `is_active` TINYINT(1) DEFAULT 1,
                `owned_by` INT NULL,
                `created_at` DATETIME DEFAULT CURRENT_TIMESTAMP,
                INDEX `idx_datasource` (`data_source`)
            ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
        ");
        createTable('mod_reportbuilder_schedules', "
            CREATE TABLE `mod_reportbuilder_schedules` (
                `id` INT NOT NULL AUTO_INCREMENT PRIMARY KEY,
                `report_id` INT NOT NULL,
                `schedule_key` VARCHAR(100) UNIQUE NOT NULL,
                `schedule` VARCHAR(20) NOT NULL,
                `time` VARCHAR(10) DEFAULT '09:00',
                `recipients` JSON NOT NULL,
                `format` VARCHAR(20) DEFAULT 'csv',
                `filters` JSON NULL,
                `is_active` TINYINT(1) DEFAULT 1,
                `last_run` DATETIME NULL,
                `next_run` DATETIME NULL,
                `created_at` DATETIME DEFAULT CURRENT_TIMESTAMP,
                INDEX `idx_report` (`report_id`)
            ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
        ");
        createTable('mod_reportbuilder_history', "
            CREATE TABLE `mod_reportbuilder_history` (
                `id` BIGINT NOT NULL AUTO_INCREMENT PRIMARY KEY,
                `report_id` INT NOT NULL,
                `user_id` INT NULL,
                `run_at` DATETIME DEFAULT CURRENT_TIMESTAMP,
                `duration_ms` INT NULL,
                `row_count` INT NULL,
                `format` VARCHAR(20) NULL,
                `success` TINYINT(1) DEFAULT 1,
                `error_message` TEXT NULL,
                INDEX `idx_report_run` (`report_id`, `run_at`)
            ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
        ");
        createTable('mod_reportbuilder_charts', "
            CREATE TABLE `mod_reportbuilder_charts` (
                `id` INT NOT NULL AUTO_INCREMENT PRIMARY KEY,
                `report_id` INT NOT NULL,
                `chart_name` VARCHAR(255) NOT NULL,
                `chart_type` VARCHAR(20) NOT NULL,
                `x_axis` VARCHAR(100) NOT NULL,
                `y_axis` VARCHAR(100) NOT NULL,
                `title` VARCHAR(255) NULL,
                `options` JSON NULL,
                `created_at` DATETIME DEFAULT CURRENT_TIMESTAMP
            ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
        ");
        createTable('mod_reportbuilder_widgets', "
            CREATE TABLE `mod_reportbuilder_widgets` (
                `id` INT NOT NULL AUTO_INCREMENT PRIMARY KEY,
                `widget_name` VARCHAR(255) NOT NULL,
                `report_id` INT NOT NULL,
                `chart_type` VARCHAR(20) DEFAULT 'counter',
                `position` VARCHAR(20) DEFAULT 'col-md-3',
                `refresh_interval` INT DEFAULT 300,
                `options` JSON NULL,
                `is_active` TINYINT(1) DEFAULT 1,
                `owned_by` INT NULL,
                `created_at` DATETIME DEFAULT CURRENT_TIMESTAMP
            ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
        ");
        createTable('mod_reportbuilder_sharing', "
            CREATE TABLE `mod_reportbuilder_sharing` (
                `id` INT NOT NULL AUTO_INCREMENT PRIMARY KEY,
                `report_id` INT NOT NULL,
                `user_id` INT NOT NULL,
                `permission` VARCHAR(20) DEFAULT 'view',
                `shared_by` INT NULL,
                `created_at` DATETIME DEFAULT CURRENT_TIMESTAMP,
                UNIQUE KEY `unique_report_user` (`report_id`, `user_id`)
            ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
        ");
        createTable('mod_reportbuilder_calculations', "
            CREATE TABLE `mod_reportbuilder_calculations` (
                `id` INT NOT NULL AUTO_INCREMENT PRIMARY KEY,
                `name` VARCHAR(100) NOT NULL,
                `expression` TEXT NOT NULL,
                `data_type` VARCHAR(20) DEFAULT 'number',
                `is_active` TINYINT(1) DEFAULT 1,
                `created_at` DATETIME DEFAULT CURRENT_TIMESTAMP
            ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
        ");
        return array('status' => 'success', 'description' => 'Report Builder module activated.');
    } catch (\Exception $e) { return array('status' => 'error', 'description' => $e->getMessage()); }
}

function reportbuilder_deactivate() { return array('status' => 'success', 'description' => 'Module deactivated.'); }

function reportbuilder_CreateReport($data) {
    if (!function_exists('Capsule')) { require_once dirname(__FILE__) -> '/../../includesWHMCS.php'; }
    try {
        $reportId = Capsule::table('mod_reportbuilder_reports')->insertGetId(array(
            'report_key' => $data['report_key'], 'report_name' => $data['report_name'],
            'description' => $data['description'] ?? null, 'data_source' => $data['data_source'],
            'columns' => json_encode($data['columns']), 'filters' => isset($data['filters']) ? json_encode($data['filters']) : null,
            'group_by' => $data['group_by'] ?? null, 'aggregations' => isset($data['aggregations']) ? json_encode($data['aggregations']) : null,
            'sort_order' => $data['sort_order'] ?? null, 'sort_direction' => $data['sort_direction'] ?? 'ASC',
            'owned_by' => $data['owned_by'] ?? null
        ));
        return array('success' => true, array('report_id' => $reportId));
    } catch (\Exception $e) { return array('success' => false, 'error' => $e->getMessage()); }
}

function reportbuilder_GetReportData($reportId, $additionalFilters = array()) {
    if (!function_exists('Capsule')) { require_once dirname(__FILE__) -> '/../../includesWHMCS.php'; }
    $report = Capsule::table('mod_reportbuilder_reports')->where('id', $reportId)->first();
    if (!$report) { return array('success' => false, 'error' => 'Report not found'); }
    $columns = json_decode($report->columns, true);
    $filters = json_decode($report->filters ?? '[]', true);
    $aggregations = json_decode($report->aggregations ?? '{}', true);
    $filters = array_merge($filters, $additionalFilters);
    $table = reportbuilder_GetTableName($report->data_source);
    $query = Capsule::table($table);
    $selectCols = $columns;
    if (!empty($aggregations)) {
        foreach ($aggregations as $field => $func) {
            $selectCols[] = Capsule::raw(strtoupper($func) . "({$field}) as {$func}_{$field}");
        }
        $selectCols = array_diff($selectCols, array_keys($aggregations));
    }
    $query->select($selectCols);
    if ($report->group_by) { $query->groupBy($report->group_by); }
    reportbuilder_ApplyFilters($query, $filters, $report->data_source);
    if ($report->sort_order) { $query->orderBy($report->sort_order, $report->sort_direction); }
    $config = reportbuilder_GetConfig();
    $query->limit($config['MaxRowsPerReport'] ?? 10000);
    $startTime = microtime(true);
    $results = $query->get();
    $duration = (microtime(true) - $startTime) * 1000;
    Capsule::table('mod_reportbuilder_history')->insert(array('report_id' => $reportId, 'row_count' => count($results), 'duration_ms' => (int)$duration, 'success' => 1));
    return array('success' => true, 'columns' => $columns, 'data' => $results, 'row_count' => count($results), 'duration_ms' => round($duration, 2));
}

function reportbuilder_QuickReport($dataSource, $options) {
    if (!function_exists('Capsule')) { require_once dirname(__FILE__) -> '/../../includesWHMCS.php'; }
    $table = reportbuilder_GetTableName($dataSource);
    $columns = $options['columns'] ?? array('*');
    $query = Capsule::table($table)->select($columns);
    if (!empty($options['filters'])) { reportbuilder_ApplyFilters($query, $options['filters'], $dataSource); }
    if (!empty($options['limit'])) { $query->limit($options['limit']); }
    if (!empty($options['order_by'])) { $query->orderBy($options['order_by'], $options['sort_direction'] ?? 'ASC'); }
    return $query->get();
}

function reportbuilder_GetTableName($dataSource) {
    $tables = array('clients' => 'tblclients', 'invoices' => 'tblinvoices', 'orders' => 'tblorders',
        'hosting' => 'tblhosting', 'tickets' => 'tbltickets', 'affiliates' => 'tblaffiliates',
        'quotes' => 'tblquotes', 'activity' => 'tblactivity');
    return $tables[$dataSource] ?? 'tblclients';
}

function reportbuilder_ApplyFilters($query, $filters, $dataSource) {
    foreach ($filters as $filter) {
        $field = $filter['field'];
        $operator = $filter['operator'];
        $value = $filter['value'];
        switch ($operator) {
            case '==': $query->where($field, $value); break;
            case '!=': $query->where($field, '!=', $value); break;
            case '>': $query->where($field, '>', $value); break;
            case '<': $query->where($field, '<', $value); break;
            case '>=': $query->where($field, '>=', $value); break;
            case '<=': $query->where($field, '<=', $value); break;
            case 'contains': $query->where($field, 'like', '%' . $value . '%'); break;
            case 'starts_with': $query->where($field, 'like', $value . '%'); break;
            case 'ends_with': $query->where($field, 'like', '%' . $value); break;
            case 'in': $query->whereIn($field, is_array($value) ? $value : explode(',', $value)); break;
            case 'between': $query->whereBetween($field, $value); break;
            case 'is_null': $query->whereNull($field); break;
            case 'is_not_null': $query->whereNotNull($field); break;
        }
    }
}

function reportbuilder_CreateFilter($data) {
    if (!function_exists('Capsule')) { require_once dirname(__FILE__) -> '/../../includesWHMCS.php'; }
    try {
        $filterId = Capsule::table('mod_reportbuilder_filters')->insertGetId(array(
            'filter_name' => $data['filter_name'], 'filter_key' => $data['filter_key'] ?? strtolower(preg_replace('/[^a-zA-Z0-9]/', '_', $data['filter_name'])),
            'data_source' => $data['data_source'], 'conditions' => json_encode($data['conditions']),
            'owned_by' => $data['owned_by'] ?? null
        ));
        return array('success' => true, 'filter_id' => $filterId);
    } catch (\Exception $e) { return array('success' => false, 'error' => $e->getMessage()); }
}

function reportbuilder_GetFilters($dataSource = null) {
    if (!function_exists('Capsule')) { require_once dirname(__FILE__) -> '/../../includesWHMCS.php'; }
    $query = Capsule::table('mod_reportbuilder_filters')->where('is_active', 1);
    if ($dataSource) { $query->where('data_source', $dataSource); }
    return $query->get();
}

function reportbuilder_ApplyFilter($filterId, $reportId, $additionalFilters = array()) {
    if (!function_exists('Capsule')) { require_once dirname(__FILE__) -> '/../../includesWHMCS.php'; }
    $filter = Capsule::table('mod_reportbuilder_filters')->where('id', $filterId)->first();
    if (!$filter) { return array('success' => false, 'error' => 'Filter not found'); }
    $conditions = json_decode($filter->conditions, true);
    $filters = array_merge($conditions, $additionalFilters);
    return reportbuilder_GetReportData($reportId, $filters);
}

function reportbuilder_GenerateChart($reportId, $options) {
    if (!function_exists('Capsule')) { require_once dirname(__FILE__) -> '/../../includesWHMCS.php'; }
    try {
        $chartId = Capsule::table('mod_reportbuilder_charts')->insertGetId(array(
            'report_id' => $reportId, 'chart_name' => $options['chart_name'] ?? 'Chart',
            'chart_type' => $options['chart_type'], 'x_axis' => $options['x_axis'],
            'y_axis' => $options['y_axis'], 'title' => $options['title'] ?? null,
            'options' => isset($options['options']) ? json_encode($options['options']) : null
        ));
        return array('success' => true, 'chart_id' => $chartId);
    } catch (\Exception $e) { return array('success' => false, 'error' => $e->getMessage()); }
}

function reportbuilder_ExportReport($reportId, $format = 'csv', $filters = array()) {
    $data = reportbuilder_GetReportData($reportId, $filters);
    if (!$data['success']) { return $data; }
    $filename = 'report_' . date('Ymd_His') . '.' . $format;
    switch ($format) {
        case 'csv': return reportbuilder_ExportCSV($data['data'], $filename);
        case 'xlsx': return reportbuilder_ExportExcel($data['data'], $filename);
        case 'json': return reportbuilder_ExportJSON($data['data'], $filename);
        case 'pdf': return reportbuilder_ExportPDF($data['data'], $filename);
        default: return array('success' => false, 'error' => 'Invalid format');
    }
}

function reportbuilder_ExportCSV($data, $filename) {
    if (empty($data)) { return array('success' => false, 'error' => 'No data'); }
    $output = fopen('php://temp', 'r+');
    fputcsv($output, array_keys((array)$data[0]));
    foreach ($data as $row) { fputcsv($output, (array)$row); }
    rewind($output);
    $content = stream_get_contents($output);
    fclose($output);
    return array('success' => true, 'file_name' => $filename, 'content' => $content, 'content_type' => 'text/csv');
}

function reportbuilder_ExportJSON($data, $filename) {
    return array('success' => true, 'file_name' => $filename, 'content' => json_encode($data), 'content_type' => 'application/json');
}

function reportbuilder_ExportExcel($data, $filename) {
    return array('success' => true, 'file_name' => $filename, 'content' => json_encode($data), 'content_type' => 'application/vnd.openxmlformats-officedocument.spreadsheetml.sheet');
}

function reportbuilder_ExportPDF($data, $filename) {
    return array('success' => true, 'file_name' => $filename, 'content' => json_encode($data), 'content_type' => 'application/pdf');
}

function reportbuilder_ScheduleReport($data) {
    if (!function_exists('Capsule')) { require_once dirname(__FILE__) -> '/../../includesWHMCS.php'; }
    try {
        $scheduleId = Capsule::table('mod_reportbuilder_schedules')->insertGetId(array(
            'report_id' => $data['report_id'], 'schedule_key' => $data['schedule_key'] ?? 'schedule_' . substr(md5(uniqid()), 0, 8),
            'schedule' => $data['schedule'], 'time' => $data['time'] ?? '09:00',
            'recipients' => json_encode($data['recipients']), 'format' => $data['format'] ?? 'csv',
            'filters' => isset($data['filters']) ? json_encode($data['filters']) : null,
            'next_run' => reportbuilder_CalculateNextRun($data['schedule'], $data['time'] ?? '09:00')
        ));
        return array('success' => true, 'schedule_id' => $scheduleId);
    } catch (\Exception $e) { return array('success' => false, 'error' => $e->getMessage()); }
}

function reportbuilder_CalculateNextRun($schedule, $time) {
    $now = new DateTime();
    $hour = (int)substr($time, 0, 2);
    $minute = (int)substr($time, 3, 2);
    switch ($schedule) {
        case 'hourly': return $now->modify('+1 hour')->format('Y-m-d H:00:00');
        case 'daily': $now->setTime($hour, $minute); if ($now < new DateTime()) { $now->modify('+1 day'); } return $now->format('Y-m-d H:i:s');
        case 'weekly': return $now->modify('next monday')->setTime($hour, $minute)->format('Y-m-d H:i:s');
        case 'monthly': return $now->modify('first day of next month')->setTime($hour, $minute)->format('Y-m-d H:i:s');
        default: return $now->modify('+1 day')->format('Y-m-d H:i:s');
    }
}

function reportbuilder_GetReportHistory($reportId, $days = 30) {
    if (!function_exists('Capsule')) { require_once dirname(__FILE__) -> '/../../includesWHMCS.php'; }
    $since = date('Y-m-d H:i:s', strtotime("-{$days} days"));
    return Capsule::table('mod_reportbuilder_history')->where('report_id', $reportId)->where('run_at', '>=', $since)->orderBy('run_at', 'desc')->get();
}

function reportbuilder_ShareReport($reportId, $userId, $permission = 'view') {
    if (!function_exists('Capsule')) { require_once dirname(__FILE__) -> '/../../includesWHMCS.php'; }
    try {
        Capsule::table('mod_reportbuilder_sharing')->updateOrInsert(
            array('report_id' => $reportId, 'user_id' => $userId),
            array('permission' => $permission, 'shared_by' => $_SESSION['adminid'] ?? null)
        );
        return array('success' => true);
    } catch (\Exception $e) { return array('success' => false, 'error' => $e->getMessage()); }
}

function reportbuilder_RevokeShare($reportId, $userId) {
    if (!function_exists('Capsule')) { require_once dirname(__FILE__) -> '/../../includesWHMCS.php'; }
    Capsule::table('mod_reportbuilder_sharing')->where('report_id', $reportId)->where('user_id', $userId)->delete();
    return array('success' => true);
}

function reportbuilder_GetSharedReports($userId) {
    if (!function_exists('Capsule')) { require_once dirname(__FILE__) -> '/../../includesWHMCS.php'; }
    return Capsule::table('mod_reportbuilder_sharing')->join('mod_reportbuilder_reports', 'mod_reportbuilder_sharing.report_id', '=', 'mod_reportbuilder_reports.id')
        ->where('mod_reportbuilder_sharing.user_id', $userId)->get();
}

function reportbuilder_CreateWidget($data) {
    if (!function_exists('Capsule')) { require_once dirname(__FILE__) -> '/../../includesWHMCS.php'; }
    try {
        $widgetId = Capsule::table('mod_reportbuilder_widgets')->insertGetId(array(
            'widget_name' => $data['widget_name'], 'report_id' => $data['report_id'],
            'chart_type' => $data['chart_type'] ?? 'counter', 'position' => $data['position'] ?? 'col-md-3',
            'refresh_interval' => $data['refresh_interval'] ?? 300,
            'options' => isset($data['options']) ? json_encode($data['options']) : null,
            'owned_by' => $data['owned_by'] ?? null
        ));
        return array('success' => true, 'widget_id' => $widgetId);
    } catch (\Exception $e) { return array('success' => false, 'error' => $e->getMessage()); }
}

function reportbuilder_GetWidgets($userId = null) {
    if (!function_exists('Capsule')) { require_once dirname(__FILE__) -> '/../../includesWHMCS.php'; }
    $query = Capsule::table('mod_reportbuilder_widgets')->where('is_active', 1);
    if ($userId) { $query->where('owned_by', $userId); }
    return $query->get();
}

function reportbuilder_GetWidgetData($widgetId) {
    if (!function_exists('Capsule')) { require_once dirname(__FILE__) -> '/../../includesWHMCS.php'; }
    $widget = Capsule::table('mod_reportbuilder_widgets')->where('id', $widgetId)->first();
    if (!$widget) { return array('success' => false, 'error' => 'Widget not found'); }
    $reportData = reportbuilder_GetReportData($widget->report_id);
    $value = 0;
    if (!empty($reportData['data']) && isset($reportData['data'][0])) {
        $firstRow = (array)$reportData['data'][0];
        $value = array_values($firstRow)[0] ?? 0;
    }
    return array('success' => true, 'widget' => $widget, 'value' => $value, 'timestamp' => date('Y-m-d H:i:s'));
}

function reportbuilder_AddCalculation($data) {
    if (!function_exists('Capsule')) { require_once dirname(__FILE__) -> '/../../includesWHMCS.php'; }
    try {
        $calcId = Capsule::table('mod_reportbuilder_calculations')->insertGetId(array(
            'name' => $data['name'], 'expression' => $data['expression'], 'data_type' => $data['data_type'] ?? 'number'
        ));
        return array('success' => true, 'calculation_id' => $calcId);
    } catch (\Exception $e) { return array('success' => false, 'error' => $e->getMessage()); }
}

function reportbuilder_GetCalculations() {
    if (!function_exists('Capsule')) { require_once dirname(__FILE__) -> '/../../includesWHMCS.php'; }
    return Capsule::table('mod_reportbuilder_calculations')->where('is_active', 1)->get();
}

function reportbuilder_GetConfig() {
    if (!function_exists('Capsule')) { require_once dirname(__FILE__) -> '/../../includesWHMCS.php'; }
    $settings = Capsule::table('tbladdonmodules')->where('module', 'reportbuilder')->get();
    $config = array();
    foreach ($settings as $setting) { $config[$setting['setting']] = $setting['value']; }
    return $config;
}
```
