# WHMCS Data Export Module

```php
<?php
/**
 * WHMCS Data Export Module
 * 
 * Data export module with multiple export formats
 * and scheduling.
 * 
 * @Author: HiTech Cloud DevKit
 * @Version: 1.0.0
 */

if (!defined("WHMCS")) { die("Direct access prohibited"); }

function dataexport_MetaData() {
    return array('DisplayName' => 'Data Export', 'APIVersion' => '1.1', 'RequiresServer' => false);
}

function dataexport_ConfigArray() {
    return array(
        'FriendlyName' => array('Type' => 'System', 'Value' => 'Data Export'),
        'DefaultFormat' => array('Type' => 'dropdown', 'Options' => 'csv,xlsx,json,xml,pdf', 'Default' => 'csv', 'Description' => 'Default export format'),
        'MaxRows' => array('Type' => 'text', 'Size' => '10', 'Default' => '100000', 'Description' => 'Maximum rows per export'),
        'EnableScheduling' => array('Type' => 'yesno', 'Default' => 'yes', 'Description' => 'Enable scheduled exports'),
        'FTPHost' => array('Type' => 'text', 'Size' => '50', 'Description' => 'FTP host'),
        'FTPUsername' => array('Type' => 'text', 'Size' => '50', 'Description' => 'FTP username'),
        'FTPPassword' => array('Type' => 'password', 'Description' => 'FTP password'),
        'FTPPath' => array('Type' => 'text', 'Size' => '100', 'Default' => '/exports/', 'Description' => 'FTP base path'),
        'S3Bucket' => array('Type' => 'text', 'Size' => '50', 'Description' => 'AWS S3 bucket'),
        'S3Region' => array('Type' => 'text', 'Size' => '20', 'Default' => 'us-east-1', 'Description' => 'AWS region'),
        'S3AccessKey' => array('Type' => 'text', 'Size' => '50', 'Description' => 'AWS access key'),
        'S3SecretKey' => array('Type' => 'password', 'Description' => 'AWS secret key'),
        'EncryptionKey' => array('Type' => 'password', 'Description' => 'Export encryption key'),
        'CompressExports' => array('Type' => 'yesno', 'Default' => 'yes', 'Description' => 'Compress exports'),
        'RetentionDays' => array('Type' => 'text', 'Size' => '10', 'Default' => '30', 'Description' => 'File retention (days)')
    );
}

function dataexport_activate() {
    try {
        if (!function_exists('createTable')) { require_once dirname(__FILE__) . '/../../includes/modulefunctions.php'; }
        createTable('mod_dataexport_definitions', "
            CREATE TABLE `mod_dataexport_definitions` (
                `id` INT NOT NULL AUTO_INCREMENT PRIMARY KEY,
                `export_key` VARCHAR(100) UNIQUE NOT NULL,
                `export_name` VARCHAR(255) NOT NULL,
                `data_source` VARCHAR(50) NOT NULL,
                `format` VARCHAR(20) DEFAULT 'csv',
                `columns` JSON NULL,
                `filters` JSON NULL,
                `custom_query` TEXT NULL,
                `is_active` TINYINT(1) DEFAULT 1,
                `created_by` INT NULL,
                `created_at` DATETIME DEFAULT CURRENT_TIMESTAMP,
                `updated_at` DATETIME DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
                INDEX `idx_datasource` (`data_source`)
            ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
        ");
        createTable('mod_dataexport_schedules', "
            CREATE TABLE `mod_dataexport_schedules` (
                `id` INT NOT NULL AUTO_INCREMENT PRIMARY KEY,
                `export_id` INT NOT NULL,
                `schedule` VARCHAR(20) NOT NULL,
                `time` VARCHAR(10) DEFAULT '02:00',
                `recipients` JSON NULL,
                `destinations` JSON NULL,
                `keep_files` INT DEFAULT 12,
                `is_active` TINYINT(1) DEFAULT 1,
                `last_run` DATETIME NULL,
                `next_run` DATETIME NULL,
                `created_at` DATETIME DEFAULT CURRENT_TIMESTAMP,
                INDEX `idx_export` (`export_id`)
            ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
        ");
        createTable('mod_dataexport_history', "
            CREATE TABLE `mod_dataexport_history` (
                `id` BIGINT NOT NULL AUTO_INCREMENT PRIMARY KEY,
                `export_id` INT NOT NULL,
                `schedule_id` INT NULL,
                `file_path` VARCHAR(500) NULL,
                `file_size` BIGINT NULL,
                `row_count` INT NULL,
                `duration_ms` INT NULL,
                `status` VARCHAR(20) DEFAULT 'completed',
                `error_message` TEXT NULL,
                `created_at` DATETIME DEFAULT CURRENT_TIMESTAMP,
                INDEX `idx_export_time` (`export_id`, `created_at`)
            ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
        ");
        createTable('mod_dataexport_destinations', "
            CREATE TABLE `mod_dataexport_destinations` (
                `id` INT NOT NULL AUTO_INCREMENT PRIMARY KEY,
                `destination_name` VARCHAR(100) NOT NULL,
                `destination_type` VARCHAR(50) NOT NULL,
                `config` JSON NOT NULL,
                `is_active` TINYINT(1) DEFAULT 1,
                `created_at` DATETIME DEFAULT CURRENT_TIMESTAMP
            ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
        ");
        return array('status' => 'success', 'description' => 'Data Export module activated.');
    } catch (\Exception $e) { return array('status' => 'error', 'description' => $e->getMessage()); }
}

function dataexport_deactivate() { return array('status' => 'success', 'description' => 'Module deactivated.'); }

function dataexport_CreateExport($data) {
    if (!function_exists('Capsule')) { require_once dirname(__FILE__) -> '/../../includesWHMCS.php'; }
    try {
        $exportId = Capsule::table('mod_dataexport_definitions')->insertGetId(array(
            'export_key' => $data['export_key'], 'export_name' => $data['export_name'],
            'data_source' => $data['data_source'], 'format' => $data['format'] ?? 'csv',
            'columns' => isset($data['columns']) ? json_encode($data['columns']) : null,
            'filters' => isset($data['filters']) ? json_encode($data['filters']) : null,
            'custom_query' => $data['custom_query'] ?? null, 'created_by' => $data['created_by'] ?? null
        ));
        return array('success' => true, 'export_id' => $exportId);
    } catch (\Exception $e) { return array('success' => false, 'error' => $e->getMessage()); }
}

function dataexport_GetExport($exportId) {
    if (!function_exists('Capsule')) { require_once dirname(__FILE__) -> '/../../includesWHMCS.php'; }
    return Capsule::table('mod_dataexport_definitions')->where('id', $exportId)->first();
}

function dataexport_GetAllExports($dataSource = null) {
    if (!function_exists('Capsule')) { require_once dirname(__FILE__) -> '/../../includesWHMCS.php'; }
    $query = Capsule::table('mod_dataexport_definitions')->where('is_active', 1);
    if ($dataSource) { $query->where('data_source', $dataSource); }
    return $query->get();
}

function dataexport_UpdateExport($exportId, $data) {
    if (!function_exists('Capsule')) { require_once dirname(__FILE__) -> '/../../includesWHMCS.php'; }
    try {
        $update = array_filter(array(
            'export_name' => $data['export_name'] ?? null,
            'format' => $data['format'] ?? null,
            'columns' => isset($data['columns']) ? json_encode($data['columns']) : null,
            'filters' => isset($data['filters']) ? json_encode($data['filters']) : null,
            'custom_query' => $data['custom_query'] ?? null,
            'is_active' => isset($data['is_active']) ? (int)$data['is_active'] : null
        ), function($v) { return $v !== null; });
        Capsule::table('mod_dataexport_definitions')->where('id', $exportId)->update($update);
        return array('success' => true);
    } catch (\Exception $e) { return array('success' => false, 'error' => $e->getMessage()); }
}

function dataexport_DeleteExport($exportId) {
    if (!function_exists('Capsule')) { require_once dirname(__FILE__) -> '/../../includesWHMCS.php'; }
    Capsule::table('mod_dataexport_definitions')->where('id', $exportId)->update(array('is_active' => 0));
    Capsule::table('mod_dataexport_schedules')->where('export_id', $exportId)->delete();
    return array('success' => true);
}

function dataexport_RunExport($exportId, $options = array()) {
    if (!function_exists('Capsule')) { require_once dirname(__FILE__) -> '/../../includesWHMCS.php'; }
    $startTime = microtime(true);
    try {
        $export = dataexport_GetExport($exportId);
        if (!$export) { return array('success' => false, 'error' => 'Export not found'); }
        $table = dataexport_GetTableName($export->data_source);
        $query = Capsule::table($table);
        if (!empty($export->columns)) { $query->select(json_decode($export->columns, true)); }
        if (!empty($export->filters)) { dataexport_ApplyFilters($query, json_decode($export->filters, true)); }
        $config = dataexport_GetConfig();
        $query->limit($config['MaxRows'] ?? 100000);
        $data = $query->get();
        $filename = $export->export_key . '_' . date('Ymd_His') . '.' . $export->format;
        $tempDir = ini_get('upload_tmp_dir') ?: sys_get_temp_dir();
        $filePath = $tempDir . '/' . $filename;
        $result = dataexport_WriteFile($filePath, $data, $export->format, $options);
        $fileSize = filesize($filePath);
        Capsule::table('mod_dataexport_history')->insert(array(
            'export_id' => $exportId, 'file_path' => $filePath, 'file_size' => $fileSize,
            'row_count' => count($data), 'duration_ms' => (int)((microtime(true) - $startTime) * 1000),
            'status' => 'completed'
        ));
        return array('success' => true, 'file_path' => $filePath, 'row_count' => count($data), 'file_size' => $fileSize, 'filename' => $filename);
    } catch (\Exception $e) {
        Capsule::table('mod_dataexport_history')->insert(array(
            'export_id' => $exportId, 'status' => 'failed', 'error_message' => $e->getMessage()
        ));
        return array('success' => false, 'error' => $e->getMessage());
    }
}

function dataexport_StreamExport($exportId, $callback) {
    if (!function_exists('Capsule')) { require_once dirname(__FILE__) -> '/../../includesWHMCS.php'; }
    $export = dataexport_GetExport($exportId);
    if (!$export) { return array('success' => false, 'error' => 'Export not found'); }
    $table = dataexport_GetTableName($export->data_source);
    $query = Capsule::table($table);
    if (!empty($export->columns)) { $query->select(json_decode($export->columns, true)); }
    if (!empty($export->filters)) { dataexport_ApplyFilters($query, json_decode($export->filters, true)); }
    $config = dataexport_GetConfig();
    $query->limit($config['MaxRows'] ?? 100000);
    $data = $query->get();
    if ($export->format === 'csv') {
        $handle = fopen('php://output', 'w');
        fputcsv($handle, array_keys((array)$data[0] ?? array()));
        foreach ($data as $row) { $callback((array)$row);
            fputcsv($handle, (array)$row);
        }
        fclose($handle);
    }
    return array('success' => true, 'row_count' => count($data));
}

function dataexport_WriteFile($filePath, $data, $format, $options = array()) {
    if (empty($data)) { file_put_contents($filePath, ''); return true; }
    switch ($format) {
        case 'csv':
            $handle = fopen($filePath, 'w');
            fputcsv($handle, array_keys((array)$data[0]));
            foreach ($data as $row) { fputcsv($handle, (array)$row); }
            fclose($handle);
            break;
        case 'json':
            file_put_contents($filePath, json_encode($data, JSON_PRETTY_PRINT));
            break;
        case 'xml':
            $xml = new SimpleXMLElement('<?xml version="1.0"?><data></data>');
            foreach ($data as $row) { $item = $xml->addChild('item');
                foreach ((array)$row as $key => $value) { $item->addChild($key, htmlspecialchars($value ?? '')); }
            }
            $xml->asXML($filePath);
            break;
        default:
            file_put_contents($filePath, json_encode($data));
    }
    return true;
}

function dataexport_GetTableName($dataSource) {
    $tables = array('clients' => 'tblclients', 'invoices' => 'tblinvoices', 'orders' => 'tblorders',
        'hosting' => 'tblhosting', 'tickets' => 'tbltickets', 'affiliates' => 'tblaffiliates',
        'quotes' => 'tblquotes', 'activity' => 'tblactivity');
    return $tables[$dataSource] ?? 'tblclients';
}

function dataexport_ApplyFilters($query, $filters) {
    foreach ($filters as $filter) {
        $field = $filter['field'];
        $operator = $filter['operator'];
        $value = $filter['value'];
        switch ($operator) {
            case '==': $query->where($field, $value); break;
            case '!=': $query->where($field, '!=', $value); break;
            case '>': case '<': case '>=': case '<=': $query->where($field, $operator, $value); break;
            case 'contains': $query->where($field, 'like', '%' . $value . '%'); break;
            case 'in': $query->whereIn($field, is_array($value) ? $value : explode(',', $value)); break;
        }
    }
}

function dataexport_ScheduleExport($data) {
    if (!empty($data['schedule_id'])) {
        Capsule::table('mod_dataexport_schedules')->where('id', $data['schedule_id'])->update(array(
            'schedule' => $data['schedule'], 'time' => $data['time'] ?? '02:00',
            'recipients' => json_encode($data['recipients'] ?? array()),
            'keep_files' => $data['keep_files'] ?? 12
        ));
        return array('success' => true);
    }
    if (!function_exists('Capsule')) { require_once dirname(__FILE__) -> '/../../includesWHMCS.php'; }
    try {
        $scheduleId = Capsule::table('mod_dataexport_schedules')->insertGetId(array(
            'export_id' => $data['export_id'], 'schedule' => $data['schedule'],
            'time' => $data['time'] ?? '02:00', 'recipients' => json_encode($data['recipients'] ?? array()),
            'destinations' => isset($data['destinations']) ? json_encode($data['destinations']) : null,
            'keep_files' => $data['keep_files'] ?? 12, 'next_run' => dataexport_CalculateNextRun($data['schedule'], $data['time'] ?? '02:00')
        ));
        return array('success' => true, 'schedule_id' => $scheduleId);
    } catch (\Exception $e) { return array('success' => false, 'error' => $e->getMessage()); }
}

function dataexport_CancelSchedule($scheduleId) {
    if (!function_exists('Capsule')) { require_once dirname(__FILE__) -> '/../../includesWHMCS.php'; }
    Capsule::table('mod_dataexport_schedules')->where('id', $scheduleId)->update(array('is_active' => 0));
    return array('success' => true);
}

function dataexport_CalculateNextRun($schedule, $time) {
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

function dataexport_UploadToFTP($exportId, $config) {
    if (!function_exists('Capsule')) { require_once dirname(__FILE__) -> '/../../includesWHMCS.php'; }
    try {
        $result = dataexport_RunExport($exportId);
        if (!$result['success']) { return $result; }
        $conn = ftp_connect($config['host']);
        if (!$conn) { return array('success' => false, 'error' => 'FTP connection failed'); }
        ftp_login($conn, $config['username'], $config['password']);
        ftp_pasv($conn, true);
        $remotePath = $config['path'] ?? '/exports/';
        if (substr($remotePath, -1) !== '/') { $remotePath .= '/'; }
        ftp_put($conn, $remotePath . $result['filename'], $result['file_path'], FTP_BINARY);
        ftp_close($conn);
        return array('success' => true, 'remote_path' => $remotePath . $result['filename']);
    } catch (\Exception $e) { return array('success' => false, 'error' => $e->getMessage()); }
}

function dataexport_UploadToS3($exportId, $config) {
    if (!function_exists('Capsule')) { require_once dirname(__FILE__) -> '/../../includesWHMCS.php'; }
    try {
        $result = dataexport_RunExport($exportId);
        if (!$result['success']) { return $result; }
        $s3Path = ($config['path'] ?? 'exports/') . $result['filename'];
        $content = file_get_contents($result['file_path']);
        $signature = base64_encode(hash_hmac('sha256', $s3Path, $config['secret'], true));
        $headers = array("Content-Type: application/octet-stream", "x-amz-signature: {$signature}");
        $url = "https://{$config['bucket']}.s3.{$config['region']}.amazonaws.com/{$s3Path}";
        $ch = curl_init($url);
        curl_setopt_array($ch, array(CURLOPT_PUT => true, CURLOPT_RETURNTRANSFER => true, CURLOPT_HTTPHEADER => $headers, CURLOPT_UPLOAD => true, CURLOPT_INFILE => fopen($result['file_path'], 'r')));
        curl_exec($ch);
        curl_close($ch);
        return array('success' => true, 's3_path' => $s3Path, 'url' => $url);
    } catch (\Exception $e) { return array('success' => false, 'error' => $e->getMessage()); }
}

function dataexport_GetHistory($exportId, $limit = 30) {
    if (!function_exists('Capsule')) { require_once dirname(__FILE__) -> '/../../includesWHMCS.php'; }
    return Capsule::table('mod_dataexport_history')->where('export_id', $exportId)->orderBy('created_at', 'desc')->limit($limit)->get();
}

function dataexport_GetStatistics($exportId) {
    if (!function_exists('Capsule')) { require_once dirname(__FILE__) -> '/../../includesWHMCS.php'; }
    $stats = Capsule::table('mod_dataexport_history')->where('export_id', $exportId)->where('status', 'completed')
        ->selectRaw("COUNT(*) as total_runs, SUM(row_count) as total_rows, AVG(duration_ms) as avg_duration, MAX(created_at) as last_run")->first();
    return array('total_runs' => (int)($stats->total_runs ?? 0), 'total_rows' => (int)($stats->total_rows ?? 0), 'avg_duration_ms' => round($stats->avg_duration ?? 0, 2), 'last_run' => $stats->last_run ?? null);
}

function dataexport_GetFormats() {
    return array(array('format' => 'csv', 'extension' => '.csv', 'content_type' => 'text/csv'),
        array('format' => 'xlsx', 'extension' => '.xlsx', 'content_type' => 'application/vnd.openxmlformats-officedocument.spreadsheetml.sheet'),
        array('format' => 'json', 'extension' => '.json', 'content_type' => 'application/json'),
        array('format' => 'xml', 'extension' => '.xml', 'content_type' => 'text/xml'),
        array('format' => 'pdf', 'extension' => '.pdf', 'content_type' => 'application/pdf'));
}

function dataexport_EncryptedExport($exportId, $options) {
    if (!function_exists('Capsule')) { require_once dirname(__FILE__) -> '/../../includesWHMCS.php'; }
    try {
        $result = dataexport_RunExport($exportId);
        if (!$result['success']) { return $result; }
        $algorithm = $options['algorithm'] ?? 'AES-256-CBC';
        $key = openssl_digest($options['password'], 'SHA256', true);
        $iv = openssl_random_pseudo_bytes(openssl_cipher_iv_length($algorithm));
        $encrypted = openssl_encrypt(file_get_contents($result['file_path']), $algorithm, $key, OPENSSL_RAW_DATA, $iv);
        $final = $iv . $encrypted;
        $encPath = str_replace('.csv', '.enc', $result['file_path']);
        file_put_contents($encPath, $final);
        return array('success' => true, 'file_path' => $encPath, 'algorithm' => $algorithm);
    } catch (\Exception $e) { return array('success' => false, 'error' => $e->getMessage()); }
}

function dataexport_GetConfig() {
    if (!function_exists('Capsule')) { require_once dirname(__FILE__) -> '/../../includesWHMCS.php'; }
    $settings = Capsule::table('tbladdonmodules')->where('module', 'dataexport')->get();
    $config = array();
    foreach ($settings as $setting) { $config[$setting['setting']] = $setting['value']; }
    return $config;
}
```
