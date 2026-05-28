# WHMCS Remote Import Module Skill
# Version: 1.0 | Updated: 2026-05-28

## Purpose

Guide for building modules that import services from external platforms.

## When to Use

- Creating migration tools
- Building sync utilities for external platforms
- Managing bulk imports

## Remote Import Patterns

```php
<?php
// modules/addons/{importmodule}/{importmodule}.php
if (!defined("WHMCS")) { die("Direct access denied"); }

function {importmodule}_config(): array {
    return [
        'name' => 'Remote Import Manager',
        'description' => 'Import services from external platforms',
        'version' => '1.0',
        'author' => 'Author',
        'api_endpoint' => ['FriendlyName' => 'API Endpoint', 'Type' => 'text'],
        'api_key' => ['FriendlyName' => 'API Key', 'Type' => 'password'],
        'auto_import' => ['FriendlyName' => 'Enable Auto Import', 'Type' => 'yesno'],
    ];
}

function {importmodule}_activate(): array {
    Capsule::schema()->create('mod_import_logs', function($t) {
        $t->increments('id');
        $t->string('source_id', 255);
        $t->string('source_type', 50);
        $t->integer('whmcs_service_id')->unsigned()->nullable();
        $t->string('status', 50);
        $t->text('data')->nullable();
        $t->text('error')->nullable();
        $t->timestamp('imported_at');
    });

    Capsule::schema()->create('mod_import_mappings', function($t) {
        $t->increments('id');
        $t->string('source_field', 100);
        $t->string('target_field', 100);
        $t->string('transform', 100)->nullable();
        $t->text('default_value')->nullable();
    });

    Capsule::schema()->create('mod_import_batch', function($t) {
        $t->increments('id');
        $t->string('batch_id', 50)->unique();
        $t->string('status', 50);
        $t->integer('total_items')->unsigned()->default(0);
        $t->integer('processed')->unsigned()->default(0);
        $t->integer('success')->unsigned()->default(0);
        $t->integer('failed')->unsigned()->default(0);
        $t->timestamp('started_at');
        $t->timestamp('completed_at')->nullable();
    });

    return ['status' => 'success', 'description' => 'Import module activated'];
}

function {importmodule}_deactivate(): array {
    Capsule::schema()->dropIfExists('mod_import_logs');
    Capsule::schema()->dropIfExists('mod_import_mappings');
    Capsule::schema()->dropIfExists('mod_import_batch');
    return ['status' => 'success', 'description' => 'Import module deactivated'];
}

function {importmodule}_output(array $vars): void {
    $action = $_REQUEST['action'] ?? 'dashboard';
    include __DIR__ . '/templates/admin/' . $action . '.tpl';
}
```

### Import Engine

```php
function startImportBatch(array $items): string {
    $batchId = 'IMP_' . date('Ymd_His') . '_' . uniqid();

    Capsule::table('mod_import_batch')->insert([
        'batch_id' => $batchId,
        'status' => 'running',
        'total_items' => count($items),
        'started_at' => date('Y-m-d H:i:s'),
    ]);

    return $batchId;
}

function importItem(array $itemData, string $batchId): array {
    try {
        // Apply field mappings
        $mappedData = applyMappings($itemData);

        // Check for existing service
        $existingMapping = getExistingMapping($mappedData);
        if ($existingMapping) {
            // Update existing service
            $result = updateExistingService($existingMapping['whmcs_service_id'], $mappedData);
            $status = 'updated';
        } else {
            // Create new service
            $result = createNewService($mappedData);
            $status = 'created';
        }

        // Log successful import
        Capsule::table('mod_import_logs')->insert([
            'source_id' => $mappedData['source_id'],
            'source_type' => $mappedData['source_type'],
            'whmcs_service_id' => $result['service_id'],
            'status' => $status,
            'data' => json_encode($mappedData),
            'imported_at' => date('Y-m-d H:i:s'),
        ]);

        return [
            'success' => true,
            'service_id' => $result['service_id'],
            'status' => $status,
        ];

    } catch (\Exception $e) {
        Capsule::table('mod_import_logs')->insert([
            'source_id' => $itemData['id'] ?? 'unknown',
            'source_type' => $itemData['type'] ?? 'unknown',
            'status' => 'failed',
            'data' => json_encode($itemData),
            'error' => $e->getMessage(),
            'imported_at' => date('Y-m-d H:i:s'),
        ]);

        return [
            'success' => false,
            'error' => $e->getMessage(),
        ];
    }
}

function applyMappings(array $itemData): array {
    $mappings = Capsule::table('mod_import_mappings')->get();

    $result = [];
    foreach ($mappings as $mapping) {
        $value = $itemData[$mapping->source_field] ?? $mapping->default_value ?? '';

        if ($mapping->transform) {
            $value = applyTransform($mapping->transform, $value, $itemData);
        }

        $result[$mapping->target_field] = $value;
    }

    return $result;
}

function applyTransform(string $transform, $value, array $context): mixed {
    switch ($transform) {
        case 'uppercase':
            return strtoupper($value);

        case 'lowercase':
            return strtolower($value);

        case 'status_map':
            $statuses = [
                'active' => 'Active',
                'suspended' => 'Suspended',
                'cancelled' => 'Terminated',
                'pending' => 'Pending',
            ];
            return $statuses[strtolower($value)] ?? $value;

        case 'date_format':
            return date('Y-m-d', strtotime($value));

        case 'currency_convert':
            $rate = getCurrencyRate($context['source_currency'] ?? 'USD');
            return $value * $rate;

        default:
            return $value;
    }
}

function getExistingMapping(array $mappedData): ?array {
    return Capsule::table('mod_import_logs')
        ->where('source_id', $mappedData['source_id'])
        ->where('whmcs_service_id', '>', 0)
        ->orderBy('imported_at', 'desc')
        ->first();
}
```

### Service Creation

```php
function createNewService(array $data): array {
    // Find product by mapped product ID
    $product = Capsule::table('tblproducts')
        ->where('id', $data['product_id'])
        ->orWhere('name', 'like', '%' . ($data['product_name'] ?? '') . '%')
        ->first();

    if (!$product) {
        throw new \Exception('Product not found: ' . ($data['product_name'] ?? 'unknown'));
    }

    // Find or create client
    $client = findOrCreateClient($data);

    // Create service via WHMCS API
    $params = [
        'clientid' => $client->id,
        'pid' => $product->id,
        'domain' => $data['domain'] ?? '',
        'billingcycle' => $data['billing_cycle'] ?? 'Monthly',
        'regdate' => $data['created_date'] ?? date('Y-m-d'),
        'nextduedate' => $data['next_due_date'] ?? date('Y-m-d'),
        'status' => $data['status'] ?? 'Pending',
    ];

    $result = localAPI('CreateService', $params);

    if ($result['result'] !== 'success') {
        throw new \Exception('Failed to create service: ' . ($result['message'] ?? 'Unknown error'));
    }

    $serviceId = $result['serviceid'];

    // Save custom fields
    if (!empty($data['custom_fields'])) {
        foreach ($data['custom_fields'] as $key => $value) {
            saveCustomFieldValue($serviceId, $key, $value);
        }
    }

    // Store import mapping
    Capsule::table('mod_import_mapping_cache')->insert([
        'source_id' => $data['source_id'],
        'whmcs_service_id' => $serviceId,
        'imported_at' => date('Y-m-d H:i:s'),
    ]);

    return [
        'service_id' => $serviceId,
        'client_id' => $client->id,
    ];
}

function findOrCreateClient(array $data): object {
    $client = Capsule::table('tblclients')
        ->where('email', $data['client_email'] ?? '')
        ->first();

    if (!$client) {
        $params = [
            'firstname' => $data['client_firstname'] ?? 'Imported',
            'lastname' => $data['client_lastname'] ?? 'Customer',
            'email' => $data['client_email'] ?? 'import_' . uniqid() . '@import.local',
            'country' => $data['client_country'] ?? '',
            'phonenumber' => $data['client_phone'] ?? '',
        ];

        $result = localAPI('AddClient', $params);
        $client = Capsule::table('tblclients')->where('id', $result['clientid'])->first();
    }

    return $client;
}
```

### Cron Auto Import

```php
function {importmodule}_cron(): void {
    $autoImport = Capsule::table('mod_configuration')
        ->where('setting', 'auto_import')
        ->value('value');

    if (!$autoImport) {
        return;
    }

    $api = new \Import\ApiClient(getModuleConfig());

    try {
        $remoteItems = $api->fetchNewItems();

        if (empty($remoteItems)) {
            return;
        }

        $batchId = startImportBatch($remoteItems);
        $success = 0;
        $failed = 0;

        foreach ($remoteItems as $item) {
            $result = importItem($item, $batchId);

            if ($result['success']) {
                $success++;
            } else {
                $failed++;
            }

            Capsule::table('mod_import_batch')
                ->where('batch_id', $batchId)
                ->update([
                    'processed' => $success + $failed,
                    'success' => $success,
                    'failed' => $failed,
                ]);
        }

        Capsule::table('mod_import_batch')
            ->where('batch_id', $batchId)
            ->update([
                'status' => 'completed',
                'completed_at' => date('Y-m-d H:i:s'),
            ]);

        logActivity("Import batch $batchId completed: $success success, $failed failed");

    } catch (\Exception $e) {
        logActivity("Import cron failed: " . $e->getMessage());
    }
}
```

---

**Related Skills:**
- whmcs-addon-builder
- whmcs-migration-guide
- whmcs-api-integration
