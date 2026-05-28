# WHMCS API Documentation Module

```php
<?php
/**
 * WHMCS API Documentation Module
 * 
 * API documentation generator with Swagger/OpenAPI
 * support.
 * 
 * @Author: HiTech Cloud DevKit
 * @Version: 1.0.0
 */

if (!defined("WHMCS")) { die("Direct access prohibited"); }

function apidocumentation_MetaData() {
    return array('DisplayName' => 'API Documentation', 'APIVersion' => '1.1', 'RequiresServer' => false);
}

function apidocumentation_ConfigArray() {
    return array(
        'FriendlyName' => array('Type' => 'System', 'Value' => 'API Documentation'),
        'DocTitle' => array('Type' => 'text', 'Size' => '50', 'Default' => 'WHMCS API Documentation', 'Description' => 'Documentation title'),
        'DocVersion' => array('Type' => 'text', 'Size' => '20', 'Default' => '1.0.0', 'Description' => 'API version'),
        'DocDescription' => array('Type' => 'textarea', 'Rows' => '3', 'Description' => 'API description'),
        'IncludeInternal' => array('Type' => 'yesno', 'Default' => 'no', 'Description' => 'Include internal endpoints'),
        'EnableTryIt' => array('Type' => 'yesno', 'Default' => 'yes', 'Description' => 'Enable sandbox testing'),
        'DefaultLanguage' => array('Type' => 'dropdown', 'Options' => 'en,es,fr,de,vi', 'Default' => 'en', 'Description' => 'Default language'),
        'Theme' => array('Type' => 'dropdown', 'Options' => 'default,slate,mono,highlight', 'Default' => 'default', 'Description' => 'Documentation theme'),
        'ShowExamples' => array('Type' => 'yesno', 'Default' => 'yes', 'Description' => 'Show request examples'),
        'Authentication' => array('Type' => 'dropdown', 'Options' => 'bearer,basic,apikey,oauth2', 'Default' => 'bearer', 'Description' => 'Default authentication type'),
        'ContactEmail' => array('Type' => 'text', 'Size' => '50', 'Description' => 'Contact email'),
        'LicenseName' => array('Type' => 'text', 'Size' => '50', 'Default' => 'MIT', 'Description' => 'License name')
    );
}

function apidocumentation_activate() {
    try {
        if (!function_exists('createTable')) { require_once dirname(__FILE__) . '/../../includes/modulefunctions.php'; }
        createTable('mod_apidocumentation_endpoints', "
            CREATE TABLE `mod_apidocumentation_endpoints` (
                `id` INT NOT NULL AUTO_INCREMENT PRIMARY KEY,
                `group_id` INT NULL,
                `path` VARCHAR(255) NOT NULL,
                `method` VARCHAR(10) NOT NULL,
                `operation_id` VARCHAR(100) UNIQUE NOT NULL,
                `summary` VARCHAR(255) NOT NULL,
                `description` TEXT NULL,
                `parameters` JSON NULL,
                `request_body` JSON NULL,
                `responses` JSON NULL,
                `tags` JSON NULL,
                `security` JSON NULL,
                `deprecated` TINYINT(1) DEFAULT 0,
                `is_internal` TINYINT(1) DEFAULT 0,
                `sort_order` INT DEFAULT 0,
                `created_at` DATETIME DEFAULT CURRENT_TIMESTAMP,
                `updated_at` DATETIME DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
                INDEX `idx_path` (`path`),
                INDEX `idx_group` (`group_id`)
            ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
        ");
        createTable('mod_apidocumentation_groups', "
            CREATE TABLE `mod_apidocumentation_groups` (
                `id` INT NOT NULL AUTO_INCREMENT PRIMARY KEY,
                `name` VARCHAR(100) NOT NULL,
                `slug` VARCHAR(100) UNIQUE NOT NULL,
                `description` TEXT NULL,
                `icon` VARCHAR(50) NULL,
                `sort_order` INT DEFAULT 0,
                `is_active` TINYINT(1) DEFAULT 1,
                `created_at` DATETIME DEFAULT CURRENT_TIMESTAMP
            ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
        ");
        createTable('mod_apidocumentation_examples', "
            CREATE TABLE `mod_apidocumentation_examples` (
                `id` INT NOT NULL AUTO_INCREMENT PRIMARY KEY,
                `endpoint_id` INT NOT NULL,
                `example_name` VARCHAR(100) NOT NULL,
                `example_type` VARCHAR(20) DEFAULT 'request',
                `request_data` JSON NULL,
                `response_data` JSON NULL,
                `language` VARCHAR(10) DEFAULT 'php',
                `explanation` TEXT NULL,
                `created_at` DATETIME DEFAULT CURRENT_TIMESTAMP,
                INDEX `idx_endpoint` (`endpoint_id`)
            ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
        ");
        createTable('mod_apidocumentation_changelog', "
            CREATE TABLE `mod_apidocumentation_changelog` (
                `id` INT NOT NULL AUTO_INCREMENT PRIMARY KEY,
                `version` VARCHAR(50) NOT NULL,
                `changes` JSON NOT NULL,
                `release_date` DATE NOT NULL,
                `is_current` TINYINT(1) DEFAULT 0,
                `created_at` DATETIME DEFAULT CURRENT_TIMESTAMP
            ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
        ");
        createTable('mod_apidocumentation_schemas', "
            CREATE TABLE `mod_apidocumentation_schemas` (
                `id` INT NOT NULL AUTO_INCREMENT PRIMARY KEY,
                `name` VARCHAR(100) UNIQUE NOT NULL,
                `type` VARCHAR(20) DEFAULT 'object',
                `properties` JSON NOT NULL,
                `required` JSON NULL,
                `description` TEXT NULL,
                `example` JSON NULL,
                `created_at` DATETIME DEFAULT CURRENT_TIMESTAMP
            ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
        ");
        // Insert default groups
        $defaultGroups = array(
            array('name' => 'Clients', 'slug' => 'clients', 'description' => 'Client management endpoints', 'icon' => 'users'),
            array('name' => 'Services', 'slug' => 'services', 'description' => 'Service/product endpoints', 'icon' => 'server'),
            array('name' => 'Billing', 'slug' => 'billing', 'description' => 'Billing and invoice endpoints', 'icon' => 'credit-card'),
            array('name' => 'Support', 'slug' => 'support', 'description' => 'Support ticket endpoints', 'icon' => 'headset'),
            array('name' => 'Domains', 'slug' => 'domains', 'description' => 'Domain management endpoints', 'icon' => 'globe'),
            array('name' => 'Authentication', 'slug' => 'auth', 'description' => 'Authentication endpoints', 'icon' => 'lock')
        );
        foreach ($defaultGroups as $group) {
            Capsule::table('mod_apidocumentation_groups')->insert($group);
        }
        return array('status' => 'success', 'description' => 'API Documentation module activated.');
    } catch (\Exception $e) { return array('status' => 'error', 'description' => $e->getMessage()); }
}

function apidocumentation_deactivate() { return array('status' => 'success', 'description' => 'Module deactivated.'); }

function apidocumentation_RegisterEndpoint($data) {
    if (!function_exists('Capsule')) { require_once dirname(__FILE__) -> '/../../includesWHMCS.php'; }
    try {
        $operationId = $data['operation_id'] ?? $data['method'] . '_' . str_replace('/', '_', trim($data['path'], '/'));
        $endpointId = Capsule::table('mod_apidocumentation_endpoints')->insertGetId(array(
            'group_id' => $data['group_id'] ?? null, 'path' => $data['path'], 'method' => strtoupper($data['method']),
            'operation_id' => $operationId, 'summary' => $data['summary'], 'description' => $data['description'] ?? null,
            'parameters' => isset($data['parameters']) ? json_encode($data['parameters']) : null,
            'request_body' => isset($data['request_body']) ? json_encode($data['request_body']) : null,
            'responses' => isset($data['responses']) ? json_encode($data['responses']) : null,
            'tags' => isset($data['tags']) ? json_encode($data['tags']) : null,
            'security' => isset($data['security']) ? json_encode($data['security']) : null,
            'deprecated' => $data['deprecated'] ?? 0, 'is_internal' => $data['is_internal'] ?? 0,
            'sort_order' => $data['sort_order'] ?? 0
        ));
        return array('success' => true, 'endpoint_id' => $endpointId, 'operation_id' => $operationId);
    } catch (\Exception $e) { return array('success' => false, 'error' => $e->getMessage()); }
}

function apidocumentation_GetEndpoint($endpointId) {
    if (!function_exists('Capsule')) { require_once dirname(__FILE__) -> '/../../includesWHMCS.php'; }
    $endpoint = Capsule::table('mod_apidocumentation_endpoints')->where('id', $endpointId)->first();
    if ($endpoint) { apidocumentation_decodeEndpointJson($endpoint); }
    return $endpoint;
}

function apidocumentation_GetEndpoints($filters = array()) {
    if (!function_exists('Capsule')) { require_once dirname(__FILE__) -> '/../../includesWHMCS.php'; }
    $query = Capsule::table('mod_apidocumentation_endpoints');
    if (!empty($filters['group_id'])) { $query->where('group_id', $filters['group_id']); }
    if (!empty($filters['method'])) { $query->where('method', strtoupper($filters['method'])); }
    if (!empty($filters['tag'])) { $query->whereRaw("JSON_CONTAINS(tags, '\"' . ? . '\"')", array($filters['tag'])); }
    if (isset($filters['deprecated'])) { $query->where('deprecated', $filters['deprecated']); }
    if (empty($filters['include_internal'])) { $query->where('is_internal', 0); }
    $query->orderBy('group_id')->orderBy('sort_order');
    $endpoints = $query->get();
    foreach ($endpoints as &$endpoint) { apidocumentation_decodeEndpointJson($endpoint); }
    return $endpoints;
}

function apidocumentation_UpdateEndpoint($endpointId, $data) {
    if (!function_exists('Capsule')) { require_once dirname(__FILE__) -> '/../../includesWHMCS.php'; }
    try {
        $update = array_filter(array(
            'group_id' => $data['group_id'] ?? null,
            'path' => $data['path'] ?? null,
            'method' => isset($data['method']) ? strtoupper($data['method']) : null,
            'summary' => $data['summary'] ?? null,
            'description' => $data['description'] ?? null,
            'parameters' => isset($data['parameters']) ? json_encode($data['parameters']) : null,
            'request_body' => isset($data['request_body']) ? json_encode($data['request_body']) : null,
            'responses' => isset($data['responses']) ? json_encode($data['responses']) : null,
            'tags' => isset($data['tags']) ? json_encode($data['tags']) : null,
            'security' => isset($data['security']) ? json_encode($data['security']) : null,
            'deprecated' => isset($data['deprecated']) ? (int)$data['deprecated'] : null,
            'sort_order' => $data['sort_order'] ?? null
        ), function($v) { return $v !== null; });
        Capsule::table('mod_apidocumentation_endpoints')->where('id', $endpointId)->update($update);
        return array('success' => true);
    } catch (\Exception $e) { return array('success' => false, 'error' => $e->getMessage()); }
}

function apidocumentation_DeleteEndpoint($endpointId) {
    if (!function_exists('Capsule')) { require_once dirname(__FILE__) -> '/../../includesWHMCS.php'; }
    Capsule::table('mod_apidocumentation_examples')->where('endpoint_id', $endpointId)->delete();
    Capsule::table('mod_apidocumentation_endpoints')->where('id', $endpointId)->delete();
    return array('success' => true);
}

function apidocumentation_AddExample($endpointId, $data) {
    if (!function_exists('Capsule')) { require_once dirname(__FILE__) -> '/../../includesWHMCS.php'; }
    try {
        $exampleId = Capsule::table('mod_apidocumentation_examples')->insertGetId(array(
            'endpoint_id' => $endpointId, 'example_name' => $data['example_name'] ?? 'Default',
            'example_type' => $data['example_type'] ?? 'request',
            'request_data' => isset($data['request']) ? json_encode($data['request']) : null,
            'response_data' => isset($data['response']) ? json_encode($data['response']) : null,
            'language' => $data['language'] ?? 'php', 'explanation' => $data['explanation'] ?? null
        ));
        return array('success' => true, 'example_id' => $exampleId);
    } catch (\Exception $e) { return array('success' => false, 'error' => $e->getMessage()); }
}

function apidocumentation_GenerateDocumentation($options = array()) {
    if (!function_exists('Capsule')) { require_once dirname(__FILE__) -> '/../../includesWHMCS.php'; }
    $config = apidocumentation_GetConfig();
    $spec = array('openapi' => '3.0.0', 'info' => array('title' => $options['title'] ?? ($config['DocTitle'] ?? 'WHMCS API'),
        'description' => $config['DocDescription'] ?? 'WHMCS API Documentation', 'version' => $options['version'] ?? ($config['DocVersion'] ?? '1.0.0'),
        'contact' => isset($config['ContactEmail']) ? array('email' => $config['ContactEmail']) : null,
        'license' => isset($config['LicenseName']) ? array('name' => $config['LicenseName']) : null),
        'servers' => array(array('url' => rtrim($options['base_url'] ?? 'https://example.com/whmcs', '/'), 'description' => 'Production Server')),
        'paths' => new stdClass(), 'components' => array('schemas' => new stdClass(), 'securitySchemes' => array('bearerAuth' => array('type' => 'http', 'scheme' => 'bearer', 'bearerFormat' => 'JWT'))));
    $groups = Capsule::table('mod_apidocumentation_groups')->where('is_active', 1)->orderBy('sort_order')->get();
    $tags = array();
    foreach ($groups as $group) { $tags[] = array('name' => $group->slug, 'description' => $group->description); }
    $spec['tags'] = $tags;
    $endpoints = apidocumentation_GetEndpoints(array('include_internal' => !empty($options['include_internal'])));
    $schemas = array();
    foreach ($endpoints as $endpoint) {
        $path = $endpoint->path;
        $method = strtolower($endpoint->method);
        if (!isset($spec['paths']->$path)) { $spec['paths']->$path = new stdClass(); }
        $operation = array('summary' => $endpoint->summary, 'description' => $endpoint->description, 'operationId' => $endpoint->operation_id,
            'tags' => $endpoint->tags ? json_decode($endpoint->tags, true) : array(), 'responses' => $endpoint->responses ? json_decode($endpoint->responses, true) : new stdClass(),
            'deprecated' => (bool)$endpoint->is_internal, 'security' => array(array('bearerAuth' => array())));
        if ($endpoint->parameters) { $operation['parameters'] = json_decode($endpoint->parameters, true); }
        if ($endpoint->request_body) { $operation['requestBody'] = json_decode($endpoint->request_body, true); }
        $spec['paths']->$path->$method = $operation;
    }
    $schemaObjects = Capsule::table('mod_apidocumentation_schemas')->get();
    foreach ($schemaObjects as $schema) { $spec['components']['schemas'][$schema->name] = array('type' => $schema->type, 'properties' => json_decode($schema->properties, true), 'required' => json_decode($schema->required ?? '[]', true), 'description' => $schema->description); }
    return array('success' => true, 'spec' => $spec);
}

function apidocumentation_ExportDocumentation($format = 'json') {
    if (!function_exists('Capsule')) { require_once dirname(__FILE__) -> '/../../includesWHMCS.php'; }
    $doc = apidocumentation_GenerateDocumentation();
    if (!$doc['success']) { return $doc; }
    $spec = $doc['spec'];
    switch ($format) {
        case 'json': return array('success' => true, 'content' => json_encode($spec, JSON_PRETTY_PRINT), 'content_type' => 'application/json');
        case 'yaml': return array('success' => true, 'content' => apidocumentation_ToYaml($spec), 'content_type' => 'text/yaml');
        case 'html': return array('success' => true, 'content' => apidocumentation_GenerateHtml($spec), 'content_type' => 'text/html');
        default: return array('success' => false, 'error' => 'Invalid format');
    }
}

function apidocumentation_ToYaml($spec, $indent = 0) {
    $yaml = '';
    $spacing = str_repeat('  ', $indent);
    foreach ($spec as $key => $value) {
        if (is_array($value) || is_object($value)) {
            $yaml .= $spacing . $key . ":\n" . apidocumentation_ToYaml($value, $indent + 1);
        } elseif (is_bool($value)) {
            $yaml .= $spacing . $key . ": " . ($value ? 'true' : 'false') . "\n";
        } else {
            $yaml .= $spacing . $key . ": " . $value . "\n";
        }
    }
    return $yaml;
}

function apidocumentation_GenerateHtml($spec) {
    return '<!DOCTYPE html><html><head><title>' . htmlspecialchars($spec['info']['title']) . '</title>' .
        '<style>body{font-family:Arial,sans-serif;margin:0;padding:20px}.endpoint{margin:20px 0;padding:15px;border:1px solid #ddd}' .
        '.method{font-weight:bold;padding:3px 8px;color:#fff}.get,.head{background:#61affe}.post{background:#49cc90}' .
        '.put{background:#fca130}.delete{background:#f93e3e}.patch{background:#50e3c2}' .
        '</style></head><body><h1>' . htmlspecialchars($spec['info']['title']) . ' v' . htmlspecialchars($spec['info']['version']) . '</h1>' .
        '<p>' . htmlspecialchars($spec['info']['description'] ?? '') . '</p>' .
        '<pre>' . htmlspecialchars(json_encode($spec, JSON_PRETTY_PRINT)) . '</pre></body></html>';
}

function apidocumentation_GenerateCodeSamples($endpointId) {
    $endpoint = apidocumentation_GetEndpoint($endpointId);
    if (!$endpoint) { return array('success' => false, 'error' => 'Endpoint not found'); }
    $samples = array();
    $params = $endpoint->parameters ? json_decode($endpoint->parameters, true) : array();
    $phpExample = '<?php' . "\n";
    $phpExample .= '$response = $client->request(\'' . $endpoint->method . '\', \'' . $endpoint->path . '\');' . "\n";
    $jsExample = 'const response = await fetch(\'' . $endpoint->path . '\', {' . "\n";
    $jsExample .= '  method: \'' . $endpoint->method . '\',' . "\n";
    $jsExample .= '  headers: { \'Authorization\': \'Bearer \' + token }' . "\n";
    $jsExample .= '});' . "\n";
    $pyExample = 'import requests' . "\n";
    $pyExample .= 'response = requests.' . strtolower($endpoint->method) . '(\'' . $endpoint->path . '\', headers=headers)' . "\n";
    return array('success' => true, 'php' => $phpExample, 'javascript' => $jsExample, 'python' => $pyExample);
}

function apidocumentation_GetGroups() {
    if (!function_exists('Capsule')) { require_once dirname(__FILE__) -> '/../../includesWHMCS.php'; }
    return Capsule::table('mod_apidocumentation_groups')->where('is_active', 1)->orderBy('sort_order')->get();
}

function apidocumentation_AddChangelog($data) {
    if (!function_exists('Capsule')) { require_once dirname(__FILE__) -> '/../../includesWHMCS.php'; }
    try {
        if (!empty($data['is_current'])) { Capsule::table('mod_apidocumentation_changelog')->update(array('is_current' => 0)); }
        $changeId = Capsule::table('mod_apidocumentation_changelog')->insertGetId(array(
            'version' => $data['version'], 'changes' => json_encode($data['changes']),
            'release_date' => $data['date'] ?? date('Y-m-d'), 'is_current' => !empty($data['is_current'])
        ));
        return array('success' => true, 'change_id' => $changeId);
    } catch (\Exception $e) { return array('success' => false, 'error' => $e->getMessage()); }
}

function apidocumentation_GetChangelog() {
    if (!function_exists('Capsule')) { require_once dirname(__FILE__) -> '/../../includesWHMCS.php'; }
    $entries = Capsule::table('mod_apidocumentation_changelog')->orderBy('release_date', 'desc')->get();
    foreach ($entries as &$entry) { $entry->changes = json_decode($entry->changes, true); }
    return $entries;
}

function apidocumentation_SearchEndpoints($query) {
    if (!function_exists('Capsule')) { require_once dirname(__FILE__) -> '/../../includesWHMCS.php'; }
    $search = '%' . $query . '%';
    $endpoints = Capsule::table('mod_apidocumentation_endpoints')
        ->whereRaw("(summary LIKE ? OR description LIKE ? OR path LIKE ? OR operation_id LIKE ?)", array($search, $search, $search, $search))
        ->get();
    foreach ($endpoints as &$endpoint) { apidocumentation_decodeEndpointJson($endpoint); }
    return $endpoints;
}

function apidocumentation_AddSchema($data) {
    if (!function_exists('Capsule')) { require_once dirname(__FILE__) -> '/../../includesWHMCS.php'; }
    try {
        $schemaId = Capsule::table('mod_apidocumentation_schemas')->insertGetId(array(
            'name' => $data['name'], 'type' => $data['type'] ?? 'object',
            'properties' => json_encode($data['properties']), 'required' => isset($data['required']) ? json_encode($data['required']) : null,
            'description' => $data['description'] ?? null, 'example' => isset($data['example']) ? json_encode($data['example']) : null
        ));
        return array('success' => true, 'schema_id' => $schemaId);
    } catch (\Exception $e) { return array('success' => false, 'error' => $e->getMessage()); }
}

function apidocumentation_decodeEndpointJson(&$endpoint) {
    $endpoint->parameters = json_decode($endpoint->parameters ?? null, true);
    $endpoint->request_body = json_decode($endpoint->request_body ?? null, true);
    $endpoint->responses = json_decode($endpoint->responses ?? null, true);
    $endpoint->tags = json_decode($endpoint->tags ?? null, true);
    $endpoint->security = json_decode($endpoint->security ?? null, true);
}

function apidocumentation_GetConfig() {
    if (!function_exists('Capsule')) { require_once dirname(__FILE__) -> '/../../includesWHMCS.php'; }
    $settings = Capsule::table('tbladdonmodules')->where('module', 'apidocumentation')->get();
    $config = array();
    foreach ($settings as $setting) { $config[$setting['setting']] = $setting['value']; }
    return $config;
}
```
