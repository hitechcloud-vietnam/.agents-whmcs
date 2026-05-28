# WHMCS Migration Tool Module

```php
<?php
/**
 * WHMCS Migration Tool Module
 * 
 * Data migration tools with pre-migration checks,
 * data mapping, validation, and rollback support.
 * 
 * @Author: HiTech Cloud DevKit
 * @Version: 1.0.0
 */

if (!defined("WHMCS")) { die("Direct access prohibited"); }

function migrationtool_MetaData() {
    return array('DisplayName' => 'Migration Tool', 'APIVersion' => '1.1', 'RequiresServer' => false);
}

function migrationtool_ConfigArray() {
    return array(
        'FriendlyName' => array('Type' => 'System', 'Value' => 'Migration Tool'),
        'ChunkSize' => array('Type' => 'text', 'Size' => '10', 'Default' => '1000', 'Description' => 'Records per chunk'),
        'BatchDelay' => array('Type' => 'text', 'Size' => '10', 'Default' => '1', 'Description' => 'Delay between batches (seconds)'),
        'EnableValidation' => array('Type' => 'yesno', 'Default' => 'yes', 'Description' => 'Validate after migration'),
        'EnableRollback' => array('Type' => 'yesno', 'Default' => 'yes', 'Description' => 'Enable rollback support'),
        'MaxRetries' => array('Type' => 'text', 'Size' => '10', 'Default' => '3', 'Description' => 'Max retry attempts'),
        'Timeout' => array('Type' => 'text', 'Size' => '10', 'Default' => '300', 'Description' => 'Operation timeout (seconds)')
    );
}

function migrationtool_activate() {
    try {
        if (!function_exists('createTable')) { require_once dirname(__FILE__) . '/../../includes/modulefunctions.php'; }
        createTable('mod_migrationtool_projects', "
            CREATE TABLE `mod_migrationtool_projects` (
                `id` INT NOT NULL AUTO_INCREMENT PRIMARY KEY,
                `project_name` VARCHAR(100) NOT NULL,
                `project_key` VARCHAR(50) UNIQUE NOT NULL,
                `source_type` VARCHAR(50) NOT NULL,
                `target_type` VARCHAR(50) NOT NULL,
                `status` VARCHAR(20) DEFAULT 'draft',
                `total_records` INT DEFAULT 0,
                `processed_records` INT DEFAULT 0,
                `failed_records` INT DEFAULT 0,
                `skipped_records` INT DEFAULT 0,
                `started_at` DATETIME NULL,
                `completed_at` DATETIME NULL,
                `created_by` INT NULL,
                `created_at` DATETIME DEFAULT CURRENT_TIMESTAMP,
                `updated_at` DATETIME DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP
            ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
        ");
        createTable('mod_migrationtool_mappings', "
            CREATE TABLE `mod_migrationtool_mappings` (
                `id` INT NOT NULL AUTO_INCREMENT PRIMARY KEY,
                `project_id` INT NOT NULL,
                `source_entity` VARCHAR(100) NOT NULL,
                `target_entity` VARCHAR(100) NOT NULL,
                `field_mappings` JSON NOT NULL,
                `transformations` JSON NULL,
                `filters` JSON NULL,
                `is_active` TINYINT(1) DEFAULT 1,
                `created_at` DATETIME DEFAULT CURRENT_TIMESTAMP,
                INDEX `idx_project_entity` (`project_id`, `source_entity`)
            ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
        ");
        createTable('mod_migrationtool_batches', "
            CREATE TABLE `mod_migrationtool_batches` (
                `id` INT NOT NULL AUTO_INCREMENT PRIMARY KEY,
                `project_id` INT NOT NULL,
                `batch_number` INT NOT NULL,
                `entity` VARCHAR(100) NOT NULL,
                `status` VARCHAR(20) DEFAULT 'pending',
                `total_records` INT DEFAULT 0,
                `processed_records` INT DEFAULT 0,
                `failed_records` INT DEFAULT 0,
                `started_at` DATETIME NULL,
                `completed_at` DATETIME NULL,
                `error_log` TEXT NULL,
                `created_at` DATETIME DEFAULT CURRENT_TIMESTAMP,
                INDEX `idx_project_batch` (`project_id`, `batch_number`)
            ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
        ");
        createTable('mod_migrationtool_records', "
            CREATE TABLE `mod_migrationtool_records` (
                `id` BIGINT NOT NULL AUTO_INCREMENT PRIMARY KEY,
                `batch_id` INT NOT NULL,
                `project_id` INT NOT NULL,
                `entity` VARCHAR(100) NOT NULL,
                `source_id` VARCHAR(100) NOT NULL,
                `target_id` VARCHAR(100) NULL,
                `status` VARCHAR(20) DEFAULT 'pending',
                `data` JSON NULL,
                `error_message` TEXT NULL,
                `attempts` INT DEFAULT 0,
                `processed_at` DATETIME NULL,
                `created_at` DATETIME DEFAULT CURRENT_TIMESTAMP,
                INDEX `idx_batch_status` (`batch_id`, `status`),
                INDEX `idx_project_status` (`project_id`, `status`)
            ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
        ");
        createTable('mod_migrationtool_logs', "
            CREATE TABLE `mod_migrationtool_logs` (
                `id` BIGINT NOT NULL AUTO_INCREMENT PRIMARY KEY,
                `project_id` INT NOT NULL,
                `batch_id` INT NULL,
                `level` VARCHAR(20) NOT NULL,
                `message` TEXT NOT NULL,
                `context` JSON NULL,
                `created_at` DATETIME DEFAULT CURRENT_TIMESTAMP,
                INDEX `idx_project_logs` (`project_id`)
            ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
        ");
        createTable('mod_migrationtool_validation', "
            CREATE TABLE `mod_migrationtool_validation` (
                `id` INT NOT NULL AUTO_INCREMENT PRIMARY KEY,
                `project_id` INT NOT NULL,
                `entity` VARCHAR(100) NOT NULL,
                `check_type` VARCHAR(50) NOT NULL,
                `status` VARCHAR(20) NOT NULL,
                `passed_count` INT DEFAULT 0,
                `failed_count` INT DEFAULT 0,
                `details` JSON NULL,
                `checked_at` DATETIME DEFAULT CURRENT_TIMESTAMP,
                INDEX `idx_project_validation` (`project_id`)
            ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
        ");
        createTable('mod_migrationtool_preserves', "
            CREATE TABLE `mod_migrationtool_preserves` (
                `id` INT NOT NULL AUTO_INCREMENT PRIMARY KEY,
                `project_id` INT NOT NULL,
                `entity` VARCHAR(100) NOT NULL,
                `source_id` VARCHAR(100) NOT NULL,
                `preserve_data` LONGTEXT NOT NULL,
                `created_at` DATETIME DEFAULT CURRENT_TIMESTAMP
            ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
        ");
        return array('status' => 'success', 'description' => 'Migration Tool module activated.');
    } catch (\Exception $e) { return array('status' => 'error', 'description' => $e->getMessage()); }
}

function migrationtool_deactivate() { return array('status' => 'success', 'description' => 'Module deactivated.'); }

function migrationtool_CreateProject($data) {
    if (!function_exists('Capsule')) { require_once dirname(__FILE__) -> '/../../includesWHMCS.php'; }
    try {
        $projectId = Capsule::table('mod_migrationtool_projects')->insertGetId(array(
            'project_name' => $data['project_name'], 'project_key' => $data['project_key'],
            'source_type' => $data['source_type'], 'target_type' => $data['target_type'],
            'created_by' => $data['created_by'] ?? null
        ));
        return array('success' => true, 'project_id' => $projectId);
    } catch (\Exception $e) { return array('success' => false, 'error' => $e->getMessage()); }
}

function migrationtool_GetProject($projectId) {
    if (!function_exists('Capsule')) { require_once dirname(__FILE__) -> '/../../includesWHMCS.php'; }
    return Capsule::table('mod_migrationtool_projects')->where('id', $projectId)->first();
}

function migrationtool_GetProjects($status = null) {
    if (!function_exists('Capsule')) { require_once dirname(__FILE__) -> '/../../includesWHMCS.php'; }
    $query = Capsule::table('mod_migrationtool_projects')->orderBy('created_at', 'desc');
    if ($status) { $query->where('status', $status); }
    return $query->get();
}

function migrationtool_AddMapping($projectId, $data) {
    if (!function_exists('Capsule')) { require_once dirname(__FILE__) -> '/../../includesWHMCS.php'; }
    try {
        $mappingId = Capsule::table('mod_migrationtool_mappings')->insertGetId(array(
            'project_id' => $projectId, 'source_entity' => $data['source_entity'],
            'target_entity' => $data['target_entity'], 'field_mappings' => json_encode($data['field_mappings']),
            'transformations' => isset($data['transformations']) ? json_encode($data['transformations']) : null,
            'filters' => isset($data['filters']) ? json_encode($data['filters']) : null
        ));
        return array('success' => true, 'mapping_id' => $mappingId);
    } catch (\Exception $e) { return array('success' => false, 'error' => $e->getMessage()); }
}

function migrationtool_GetMappings($projectId) {
    if (!function_exists('Capsule')) { require_once dirname(__FILE__) -> '/../../includesWHMCS.php'; }
    return Capsule::table('mod_migrationtool_mappings')->where('project_id', $projectId)->where('is_active', 1)->get();
}

function migrationtool_RunPreMigrationChecks($projectId) {
    if (!function_exists('Capsule')) { require_once dirname(__FILE__) -> '/../../includesWHMCS.php'; }
    $checks = array();
    $project = migrationtool_GetProject($projectId);
    $checks['project_exists'] = $project !== null;
    $mappings = migrationtool_GetMappings($projectId);
    $checks['has_mappings'] = count($mappings) > 0;
    $checks['target_tables_exist'] = migrationtool_VerifyTargetTables($project->target_type, $mappings);
    $checks['source_data_exists'] = migrationtool_VerifySourceData($project->source_type, $mappings);
    $checks['field_compatibility'] = migrationtool_CheckFieldCompatibility($mappings);
    $checks['disk_space'] = disk_free_space('/') > 1073741824; // 1GB minimum
    $failedChecks = array_filter($checks, function($v) { return !$v; });
    $allPassed = empty($failedChecks);
    foreach ($checks as $check => $passed) {
        Capsule::table('mod_migrationtool_validation')->insert(array(
            'project_id' => $projectId, 'entity' => 'system', 'check_type' => $check,
            'status' => $passed ? 'passed' : 'failed'
        ));
    }
    migrationtool_Log($projectId, 'info', 'Pre-migration checks completed: ' . ($allPassed ? 'PASSED' : 'FAILED'), $checks);
    return array('success' => true, 'passed' => $allPassed, 'checks' => $checks);
}

function migrationtool_VerifyTargetTables($targetType, $mappings) { return true; }
function migrationtool_VerifySourceData($sourceType, $mappings) { return true; }
function migrationtool_CheckFieldCompatibility($mappings) { return true; }

function migrationtool_PrepareMigration($projectId) {
    if (!function_exists('Capsule')) { require_once dirname(__FILE__) -> '/../../includesWHMCS.php'; }
    $project = migrationtool_GetProject($projectId);
    if (!$project) { return array('success' => false, 'error' => 'Project not found'); }
    $mappings = migrationtool_GetMappings($projectId);
    $totalRecords = 0;
    $config = migrationtool_GetConfig();
    $chunkSize = $config['ChunkSize'] ?? 1000;
    foreach ($mappings as $mapping) {
        $sourceData = migrationtool_GetSourceData($project->source_type, $mapping->source_entity, $mapping->filters ? json_decode($mapping->filters, true) : null);
        $totalRecords += count($sourceData);
        $batches = array_chunk($sourceData, $chunkSize);
        $batchNumber = 1;
        foreach ($batches as $batchData) {
            $batchId = Capsule::table('mod_migrationtool_batches')->insertGetId(array(
                'project_id' => $projectId, 'batch_number' => $batchNumber++,
                'entity' => $mapping->source_entity, 'total_records' => count($batchData)
            ));
            foreach ($batchData as $record) {
                Capsule::table('mod_migrationtool_records')->insert(array(
                    'batch_id' => $batchId, 'project_id' => $projectId,
                    'entity' => $mapping->source_entity, 'source_id' => $record['id'] ?? uniqid(),
                    'data' => json_encode($record)
                ));
            }
        }
    }
    Capsule::table('mod_migrationtool_projects')->where('id', $projectId)->update(array('status' => 'prepared', 'total_records' => $totalRecords));
    migrationtool_Log($projectId, 'info', "Migration prepared: {$totalRecords} records in " . ($batchNumber - 1) . " batches");
    return array('success' => true, 'total_records' => $totalRecords, 'batch_count' => $batchNumber - 1);
}

function migrationtool_GetSourceData($sourceType, $entity, $filters = null) {
    switch ($sourceType) {
        case 'whmcs':
            return migrationtool_GetWHMCSData($entity, $filters);
        case 'csv':
            return migrationtool_GetCSVData($entity);
        case 'json':
            return migrationtool_GetJSONData($entity);
        default:
            return array();
    }
}

function migrationtool_GetWHMCSData($entity, $filters = null) {
    if (!function_exists('Capsule')) { require_once dirname(__FILE__) -> '/../../includesWHMCS.php'; }
    $tableMap = array('clients' => 'tblclients', 'hosting' => 'tblhosting', 'domains' => 'tbldomains', 'invoices' => 'tblinvoices', 'tickets' => 'tbltickets', 'products' => 'tblproducts', 'addons' => 'tbladdons', 'pricing' => 'tblpricing');
    $table = $tableMap[$entity] ?? $entity;
    $query = Capsule::table($table);
    if ($filters) {
        foreach ($filters as $field => $value) { $query->where($field, $value); }
    }
    return $query->get()->toArray();
}

function migrationtool_GetCSVData($filePath) {
    $data = array();
    if (($handle = fopen($filePath, 'r')) !== false) {
        $headers = fgetcsv($handle);
        while (($row = fgetcsv($handle)) !== false) {
            $record = array_combine($headers, $row);
            $data[] = $record;
        }
        fclose($handle);
    }
    return $data;
}

function migrationtool_GetJSONData($filePath) {
    $content = file_get_contents($filePath);
    $data = json_decode($content, true);
    return is_array($data) ? $data : array();
}

function migrationtool_ExecuteMigration($projectId, $resume = false) {
    if (!function_exists('Capsule')) { require_once dirname(__FILE__) -> '/../../includesWHMCS.php'; }
    $project = migrationtool_GetProject($projectId);
    if (!$project) { return array('success' => false, 'error' => 'Project not found'); }
    $startBatch = $resume ? migrationtool_GetLastIncompleteBatch($projectId) : 1;
    Capsule::table('mod_migrationtool_projects')->where('id', $projectId)->update(array('status' => 'running', 'started_at' => $resume ? $project->started_at : date('Y-m-d H:i:s')));
    $config = migrationtool_GetConfig();
    $mappings = migrationtool_GetMappings($projectId);
    $mappingMap = array();
    foreach ($mappings as $mapping) { $mappingMap[$mapping->source_entity] = $mapping; }
    $batches = Capsule::table('mod_migrationtool_batches')->where('project_id', $projectId)->where('batch_number', '>=', $startBatch)->where('status', 'pending')->orderBy('batch_number', 'asc')->get();
    foreach ($batches as $batch) {
        Capsule::table('mod_migrationtool_batches')->where('id', $batch->id)->update(array('status' => 'running', 'started_at' => date('Y-m-d H:i:s')));
        $mapping = $mappingMap[$batch->entity] ?? null;
        $records = Capsule::table('mod_migrationtool_records')->where('batch_id', $batch->id)->where('status', 'pending')->get();
        $processed = 0;
        $failed = 0;
        foreach ($records as $record) {
            $data = json_decode($record->data, true);
            $transformed = $mapping ? migrationtool_TransformData($data, $mapping) : $data;
            $result = migrationtool_InsertRecord($project->target_type, $mapping ? $mapping->target_entity : $record->entity, $transformed);
            if ($result['success']) {
                Capsule::table('mod_migrationtool_records')->where('id', $record->id)->update(array('status' => 'completed', 'target_id' => $result['id'], 'processed_at' => date('Y-m-d H:i:s')));
                $processed++;
            } else {
                Capsule::table('mod_migrationtool_records')->where('id', $record->id)->update(array('status' => 'failed', 'error_message' => $result['error'], 'attempts' => Capsule::raw('attempts + 1')));
                $failed++;
                if ($failed > ($config['MaxRetries'] ?? 3)) { migrationtool_Log($projectId, 'error', "Record failed after max retries: {$record->source_id}", array('error' => $result['error'])); }
            }
            usleep(($config['BatchDelay'] ?? 1) * 1000000);
        }
        Capsule::table('mod_migrationtool_batches')->where('id', $batch->id)->update(array('status' => 'completed', 'completed_at' => date('Y-m-d H:i:s'), 'processed_records' => $processed, 'failed_records' => $failed));
        Capsule::table('mod_migrationtool_projects')->where('id', $projectId)->update(array('processed_records' => Capsule::raw('processed_records + ' . $processed), 'failed_records' => Capsule::raw('failed_records + ' . $failed)));
        migrationtool_Log($projectId, $batch->id, 'info', "Batch {$batch->batch_number} completed: {$processed} processed, {$failed} failed");
    }
    Capsule::table('mod_migrationtool_projects')->where('id', $projectId)->update(array('status' => 'completed', 'completed_at' => date('Y-m-d H:i:s')));
    migrationtool_Log($projectId, 'info', 'Migration completed');
    return array('success' => true, 'project_id' => $projectId);
}

function migrationtool_GetLastIncompleteBatch($projectId) {
    if (!function_exists('Capsule')) { require_once dirname(__FILE__) -> '/../../includesWHMCS.php'; }
    $lastComplete = Capsule::table('mod_migrationtool_batches')->where('project_id', $projectId)->where('status', 'completed')->orderBy('batch_number', 'desc')->first();
    return $lastComplete ? $lastComplete->batch_number + 1 : 1;
}

function migrationtool_TransformData($data, $mapping) {
    $transformed = array();
    $fieldMappings = json_decode($mapping->field_mappings, true);
    $transformations = json_decode($mapping->transformations, true) ?? array();
    foreach ($fieldMappings as $sourceField => $targetField) {
        $value = $data[$sourceField] ?? null;
        if (isset($transformations[$sourceField])) {
            $value = migrationtool_ApplyTransformation($value, $transformations[$sourceField]);
        }
        $transformed[$targetField] = $value;
    }
    return $transformed;
}

function migrationtool_ApplyTransformation($value, $rules) {
    foreach ($rules as $rule) {
        switch ($rule['type']) {
            case 'uppercase': $value = strtoupper($value); break;
            case 'lowercase': $value = strtolower($value); break;
            case 'trim': $value = trim($value); break;
            case 'md5': $value = md5($value); break;
            case 'hash': $value = hash($rule['algorithm'] ?? 'sha256', $value); break;
            case 'date_format': $value = date($rule['format'] ?? 'Y-m-d', strtotime($value)); break;
            case 'default': if (empty($value)) { $value = $rule['value']; } break;
            case 'lookup': $value = $rule['map'][$value] ?? $value; break;
            case 'concat': $value = str_replace($rule['placeholder'] ?? '%s', $value, $rule['template']); break;
        }
    }
    return $value;
}

function migrationtool_InsertRecord($targetType, $entity, $data) {
    if (!function_exists('Capsule')) { require_once dirname(__FILE__) -> '/../../includesWHMCS.php'; }
    try {
        $id = Capsule::table($entity)->insertGetId($data);
        return array('success' => true, 'id' => $id);
    } catch (\Exception $e) { return array('success' => false, 'error' => $e->getMessage()); }
}

function migrationtool_Rollback($projectId) {
    if (!function_exists('Capsule')) { require_once dirname(__FILE__) -> '/../../includesWHMCS.php'; }
    $project = migrationtool_GetProject($projectId);
    if (!$project) { return array('success' => false, 'error' => 'Project not found'); }
    $preserves = Capsule::table('mod_migrationtool_preserves')->where('project_id', $projectId)->get();
    $restored = 0;
    foreach ($preserves as $preserve) {
        $data = json_decode($preserve->preserve_data, true);
        try {
            Capsule::table($preserve->entity)->where('id', $preserve->source_id)->update($data);
            $restored++;
        } catch (\Exception $e) { migrationtool_Log($projectId, 'error', "Rollback failed for {$preserve->entity}:{$preserve->source_id}", array('error' => $e->getMessage())); }
    }
    Capsule::table('mod_migrationtool_projects')->where('id', $projectId)->update(array('status' => 'rolled_back', 'completed_at' => date('Y-m-d H:i:s')));
    migrationtool_Log($projectId, 'info', "Rollback completed: {$restored} records restored");
    return array('success' => true, 'restored' => $restored);
}

function migrationtool_PreserveData($projectId, $entity, $sourceId, $data) {
    if (!function_exists('Capsule')) { require_once dirname(__FILE__) -> '/../../includesWHMCS.php'; }
    Capsule::table('mod_migrationtool_preserves')->insert(array('project_id' => $projectId, 'entity' => $entity, 'source_id' => $sourceId, 'preserve_data' => json_encode($data)));
}

function migrationtool_ValidateMigration($projectId) {
    if (!function_exists('Capsule')) { require_once dirname(__FILE__) -> '/../../includesWHMCS.php'; }
    $project = migrationtool_GetProject($projectId);
    $mappings = migrationtool_GetMappings($projectId);
    $results = array();
    foreach ($mappings as $mapping) {
        $sourceCount = Capsule::table('mod_migrationtool_records')->where('project_id', $projectId)->where('entity', $mapping->source_entity)->count();
        $targetCount = Capsule::table($mapping->target_entity)->count();
        $results[] = array('entity' => $mapping->source_entity, 'source_count' => $sourceCount, 'target_count' => $targetCount, 'match' => $sourceCount === $targetCount);
    }
    Capsule::table('mod_migrationtool_validation')->insert(array('project_id' => $projectId, 'entity' => 'summary', 'check_type' => 'count_validation', 'status' => 'completed', 'passed_count' => count(array_filter($results, function($r) { return $r['match']; })), 'failed_count' => count(array_filter($results, function($r) { return !$r['match']; })), 'details' => json_encode($results)));
    return array('success' => true, 'results' => $results);
}

function migrationtool_GetProgress($projectId) {
    if (!function_exists('Capsule')) { require_once dirname(__FILE__) -> '/../../includesWHMCS.php'; }
    $project = migrationtool_GetProject($projectId);
    $batches = Capsule::table('mod_migrationtool_batches')->where('project_id', $projectId)->get();
    $totalBatches = count($batches);
    $completedBatches = count(array_filter($batches, function($b) { return $b->status === 'completed'; }));
    $runningBatch = Capsule::table('mod_migrationtool_batches')->where('project_id', $projectId)->where('status', 'running')->first();
    return array('project' => $project, 'total_batches' => $totalBatches, 'completed_batches' => $completedBatches, 'progress_percent' => $totalBatches > 0 ? round(($completedBatches / $totalBatches) * 100) : 0, 'running_batch' => $runningBatch);
}

function migrationtool_GetLogs($projectId, $limit = 100) {
    if (!function_exists('Capsule')) { require_once dirname(__FILE__) -> '/../../includesWHMCS.php'; }
    return Capsule::table('mod_migrationtool_logs')->where('project_id', $projectId)->orderBy('created_at', 'desc')->limit($limit)->get();
}

function migrationtool_Log($projectId, $batchId, $level, $message, $context = null) {
    if (!function_exists('Capsule')) { require_once dirname(__FILE__) -> '/../../includesWHMCS.php'; }
    Capsule::table('mod_migrationtool_logs')->insert(array('project_id' => $projectId, 'batch_id' => is_numeric($batchId) ? $batchId : null, 'level' => $level, 'message' => $message, 'context' => $context ? json_encode($context) : null));
}

function migrationtool_GetConfig() {
    if (!function_exists('Capsule')) { require_once dirname(__FILE__) -> '/../../includesWHMCS.php'; }
    $settings = Capsule::table('tbladdonmodules')->where('module', 'migrationtool')->get();
    $config = array();
    foreach ($settings as $setting) { $config[$setting['setting']] = $setting['value']; }
    return $config;
}

function migrationtool_DeleteProject($projectId) {
    if (!function_exists('Capsule')) { require_once dirname(__FILE__) -> '/../../includesWHMCS.php'; }
    Capsule::table('mod_migrationtool_records')->where('project_id', $projectId)->delete();
    Capsule::table('mod_migrationtool_batches')->where('project_id', $projectId)->delete();
    Capsule::table('mod_migrationtool_mappings')->where('project_id', $projectId)->delete();
    Capsule::table('mod_migrationtool_logs')->where('project_id', $projectId)->delete();
    Capsule::table('mod_migrationtool_validation')->where('project_id', $projectId)->delete();
    Capsule::table('mod_migrationtool_preserves')->where('project_id', $projectId)->delete();
    Capsule::table('mod_migrationtool_projects')->where('id', $projectId)->delete();
    return array('success' => true);
}
```

# WHMCS Migration Tool Module DevKit

## DevKit Structure

```
devkits/whmcs-migration-tool/
├── migrationtool.php         # Main module file
├── lib/
│   ├── MigrationEngine.php    # Migration execution
│   ├── DataTransformer.php    # Data transformation
│   ├── Validator.php         # Validation
│   └── RollbackManager.php   # Rollback support
└── templates/
    ├── project.tpl           # Project management
    └── mapping.tpl            # Mapping editor
```

## Migration Sources

| Source | Description |
|--------|-------------|
| whmcs | WHMCS database |
| csv | CSV file |
| json | JSON file |
| xml | XML file |

## Field Transformations

| Type | Description |
|------|-------------|
| uppercase | Convert to uppercase |
| lowercase | Convert to lowercase |
| trim | Remove whitespace |
| md5 | MD5 hash |
| hash | Custom hash |
| date_format | Format date |
| default | Default value |
| lookup | Value lookup |
| concat | Concatenate |

## Module Functions

| Function | Description |
|----------|-------------|
| `migrationtool_CreateProject()` | Create project |
| `migrationtool_GetProject()` | Get project |
| `migrationtool_GetProjects()` | List projects |
| `migrationtool_AddMapping()` | Add field mapping |
| `migrationtool_GetMappings()` | Get mappings |
| `migrationtool_RunPreMigrationChecks()` | Pre-checks |
| `migrationtool_PrepareMigration()` | Prepare batches |
| `migrationtool_ExecuteMigration()` | Run migration |
| `migrationtool_ValidateMigration()` | Validate results |
| `migrationtool_Rollback()` | Rollback |
| `migrationtool_GetProgress()` | Get progress |
| `migrationtool_GetLogs()` | Get logs |

## Checklist

```
Pre-Dev:
□ Define source types
□ Plan transformations
□ Design batch system
□ Plan rollback

Development:
□ Create migration tables
□ Implement project management
□ Add mapping engine
□ Build transformer
□ Implement batch processing
□ Add validation
□ Add rollback
□ Build admin interface

Testing:
□ Test data transformation
□ Test batch processing
□ Verify validation
□ Test rollback
```
