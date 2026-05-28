# WHMCS Client Migration Module

```php
<?php
/**
 * WHMCS Client Migration Module
 * 
 * Handles client data migration between WHMCS installations,
 * including services, domains, invoices, and tickets.
 * 
 * @Author: HiTech Cloud DevKit
 * @Version: 1.0.0
 */

if (!defined("WHMCS")) { die("Direct access prohibited"); }

function clientmigration_MetaData() {
    return array('DisplayName' => 'Client Migration', 'APIVersion' => '1.1', 'RequiresServer' => false);
}

function clientmigration_ConfigArray() {
    return array('FriendlyName' => array('Type' => 'System', 'Value' => 'Client Migration'),
        'EnableImport' => array('Type' => 'yesno', 'Default' => 'yes', 'Description' => 'Enable import functionality'),
        'EnableExport' => array('Type' => 'yesno', 'Default' => 'yes', 'Description' => 'Enable export functionality'),
        'MergeMode' => array('Type' => 'yesno', 'Default' => 'yes', 'Description' => 'Allow merging existing clients'),
        'EncryptData' => array('Type' => 'yesno', 'Default' => 'yes', 'Description' => 'Encrypt exported data'));
}

function clientmigration_activate() {
    try {
        if (!function_exists('createTable')) { require_once dirname(__FILE__) . '/../../includes/modulefunctions.php'; }
        createTable('mod_clientmigration_imports', "
            CREATE TABLE `mod_clientmigration_imports` (
                `id` INT NOT NULL AUTO_INCREMENT PRIMARY KEY,
                `import_key` VARCHAR(100) UNIQUE NOT NULL,
                `file_data` LONGTEXT NOT NULL,
                `status` ENUM('pending', 'processing', 'completed', 'failed') DEFAULT 'pending',
                `total_clients` INT DEFAULT 0,
                `processed` INT DEFAULT 0,
                `imported` INT DEFAULT 0,
                `merged` INT DEFAULT 0,
                `failed` INT DEFAULT 0,
                `errors` JSON NULL,
                `started_at` DATETIME NULL,
                `completed_at` DATETIME NULL,
                `created_at` DATETIME DEFAULT CURRENT_TIMESTAMP
            ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
        ");
        createTable('mod_clientmigration_logs', "
            CREATE TABLE `mod_clientmigration_logs` (
                `id` INT NOT NULL AUTO_INCREMENT PRIMARY KEY,
                `import_id` INT NULL,
                `action` VARCHAR(100) NOT NULL,
                `client_id` INT NULL,
                `details` TEXT NULL,
                `logged_at` DATETIME DEFAULT CURRENT_TIMESTAMP
            ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
        ");
        return array('status' => 'success', 'description' => 'Client Migration module activated.');
    } catch (\Exception $e) { return array('status' => 'error', 'description' => $e->getMessage()); }
}

function clientmigration_deactivate() { return array('status' => 'success', 'description' => 'Module deactivated.'); }

function clientmigration_ExportClient($clientId, $options = array()) {
    if (!function_exists('Capsule')) { require_once dirname(__FILE__) -> '/../../includesWHMCS.php'; }
    $client = Capsule::table('tblclients')->where('id', $clientId)->first();
    if (!$client) return array('success' => false, 'error' => 'Client not found');
    $export = array('version' => '1.0', 'exported_at' => date('Y-m-d H:i:s'), 'client' => (array)$client);
    if ($options['include_services'] ?? true) { $export['services'] = Capsule::table('tblhosting')->where('userid', $clientId)->get(); }
    if ($options['include_domains'] ?? true) { $export['domains'] = Capsule::table('tbldomains')->where('userid', $clientId)->get(); }
    if ($options['include_invoices'] ?? true) { $export['invoices'] = Capsule::table('tblinvoices')->where('userid', $clientId)->get(); }
    if ($options['include_tickets'] ?? true) { $export['tickets'] = Capsule::table('tbltickets')->where('userid', $clientId)->get(); }
    if ($options['include_contacts'] ?? true) { $export['contacts'] = Capsule::table('tblcontacts')->where('userid', $clientId)->get(); }
    if ($options['include_notes'] ?? true) { $export['notes'] = Capsule::table('tblnotes')->where('userid', $clientId)->get(); }
    return array('success' => true, 'data' => $export);
}

function clientmigration_ExportMultiple($clientIds, $options = array()) {
    $exports = array();
    foreach ($clientIds as $clientId) {
        $exports[] = clientmigration_ExportClient($clientId, $options);
    }
    return array('success' => true, 'exports' => $exports, 'count' => count($exports));
}

function clientmigration_PrepareImport($data) {
    if (!function_exists('Capsule')) { require_once dirname(__FILE__) -> '/../../includesWHMCS.php'; }
    try {
        $key = 'imp-' . substr(md5(uniqid()), 0, 12);
        $jsonData = json_encode($data);
        if (strlen($jsonData) > 1024 * 1024 * 10) { return array('success' => false, 'error' => 'Data exceeds 10MB limit'); }
        Capsule::table('mod_clientmigration_imports')->insert(array('import_key' => $key, 'file_data' => $jsonData, 'total_clients' => count($data)));
        return array('success' => true, 'import_key' => $key);
    } catch (\Exception $e) { return array('success' => false, 'error' => $e->getMessage()); }
}

function clientmigration_ProcessImport($importKey, $options = array()) {
    if (!function_exists('Capsule')) { require_once dirname(__FILE__) -> '/../../includesWHMCS.php'; }
    $import = Capsule::table('mod_clientmigration_imports')->where('import_key', $importKey)->first();
    if (!$import) return array('success' => false, 'error' => 'Import not found');
    $data = json_decode($import->file_data, true);
    Capsule::table('mod_clientmigration_imports')->where('id', $import->id)->update(array('status' => 'processing', 'started_at' => date('Y-m-d H:i:s')));
    $results = array('imported' => 0, 'merged' => 0, 'failed' => 0, 'errors' => array());
    foreach ($data as $clientData) {
        $result = clientmigration_ImportClient($clientData, $options);
        if ($result['success']) {
            if ($result['merged']) { $results['merged']++; } else { $results['imported']++; }
            clientmigration_Log($import->id, 'imported', $result['client_id'], 'Client imported/merged');
        } else {
            $results['failed']++;
            $results['errors'][] = $result['error'];
            clientmigration_Log($import->id, 'failed', null, $result['error']);
        }
        Capsule::table('mod_clientmigration_imports')->where('id', $import->id)->update(array('processed' => $results['imported'] + $results['merged'] + $results['failed'], 'errors' => json_encode($results['errors'])));
    }
    Capsule::table('mod_clientmigration_imports')->where('id', $import->id)->update(array('status' => 'completed', 'completed_at' => date('Y-m-d H:i:s'), 'imported' => $results['imported'], 'merged' => $results['merged'], 'failed' => $results['failed']));
    return array('success' => true, 'results' => $results);
}

function clientmigration_ImportClient($data, $options = array()) {
    if (!function_exists('Capsule')) { require_once dirname(__FILE__) -> '/../../includesWHMCS.php'; }
    try {
        $existingEmail = Capsule::table('tblclients')->where('email', $data['client']['email'])->first();
        if ($existingEmail) {
            if ($options['merge_mode'] ?? false) {
                $clientId = $existingEmail->id;
                clientmigration_MergeClient($clientId, $data);
                return array('success' => true, 'client_id' => $clientId, 'merged' => true);
            }
            return array('success' => false, 'error' => 'Client with email ' . $data['client']['email'] . ' already exists');
        }
        $clientData = $data['client'];
        unset($clientData['id']);
        $clientData['datecreated'] = date('Y-m-d H:i:s');
        Capsule::table('tblclients')->insert($clientData);
        $clientId = Capsule::connection()->getPdo()->lastInsertId();
        if (!empty($data['services'])) { foreach ($data['services'] as $service) { $s = (array)$service; unset($s['id'], $s['userid']); $s['userid'] = $clientId; Capsule::table('tblhosting')->insert($s); } }
        if (!empty($data['domains'])) { foreach ($data['domains'] as $domain) { $d = (array)$domain; unset($d['id'], $d['userid']); $d['userid'] = $clientId; Capsule::table('tbldomains')->insert($d); } }
        return array('success' => true, 'client_id' => $clientId, 'merged' => false);
    } catch (\Exception $e) { return array('success' => false, 'error' => $e->getMessage()); }
}

function clientmigration_MergeClient($clientId, $data) {
    if (!empty($data['services'])) { foreach ($data['services'] as $service) { $s = (array)$service; unset($s['id']); Capsule::table('tblhosting')->where('id', $s['id'])->update($s); } }
    if (!empty($data['domains'])) { foreach ($data['domains'] as $domain) { $d = (array)$domain; unset($d['id']); Capsule::table('tbldomains')->where('id', $d['id'])->update($d); } }
}

function clientmigration_Log($importId, $action, $clientId, $details) {
    if (!function_exists('Capsule')) { require_once dirname(__FILE__) -> '/../../includesWHMCS.php'; }
    try { Capsule::table('mod_clientmigration_logs')->insert(array('import_id' => $importId, 'action' => $action, 'client_id' => $clientId, 'details' => $details)); } catch (\Exception $e) {}
}

function clientmigration_GetImportStatus($importKey) {
    if (!function_exists('Capsule')) { require_once dirname(__FILE__) -> '/../../includesWHMCS.php'; }
    return Capsule::table('mod_clientmigration_imports')->where('import_key', $importKey)->first();
}

function clientmigration_GetLogs($importId) {
    if (!function_exists('Capsule')) { require_once dirname(__FILE__) -> '/../../includesWHMCS.php'; }
    return Capsule::table('mod_clientmigration_logs')->where('import_id', $importId)->get();
}
