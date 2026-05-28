# WHMCS Client Migration Workflow

## Purpose

Safe procedure for migrating individual clients or groups of clients between WHMCS installations, servers, or between different service configurations. Ensures data integrity and minimal service disruption.

## Prerequisites

- Source and destination WHMCS installations
- Database access to both systems
- Admin credentials for both systems
- Migration window scheduled
- Client communication prepared

## Workflow Steps

### Step 1: Identify Migration Scope

Determine what needs to be migrated:

```php
<?php
// migration_scope.php - Analyze client data for migration

require_once __DIR__ . '/init.php';

function analyzeClientForMigration($clientId)
{
    $client = Capsule::table('tblclients')
        ->where('id', $clientId)
        ->first();
    
    if (!$client) {
        return ['error' => 'Client not found'];
    }
    
    // Gather all related data
    $data = [
        'client' => $client,
        'contacts' => Capsule::table('tblcontacts')
            ->where('userid', $clientId)
            ->get(),
        'services' => Capsule::table('tblhosting')
            ->where('userid', $clientId)
            ->get(),
        'domains' => Capsule::table('tbldomain')
            ->where('userid', $clientId)
            ->get(),
        'invoices' => Capsule::table('tblinvoices')
            ->where('userid', $clientId)
            ->get(),
        'orders' => Capsule::table('tblorders')
            ->where('userid', $clientId)
            ->get(),
        'tickets' => Capsule::table('tbltickets')
            ->where('userid', $clientId)
            ->get(),
        'payment_methods' => Capsule::table('tblpaymentgateways')
            ->where('client_id', $clientId)
            ->get(),
        'files' => getClientFiles($clientId),
    ];
    
    // Calculate data size
    $data['total_records'] = array_sum([
        count($data['services']),
        count($data['domains']),
        count($data['invoices']),
        count($data['orders']),
        count($data['tickets']),
    ]);
    
    return $data;
}

function getClientFiles($clientId)
{
    $files = [];
    $uploadPaths = [
        '/var/www/whmcs/attachments/' . $clientId . '/',
        '/var/www/whmcs/downloads/client_' . $clientId . '/',
    ];
    
    foreach ($uploadPaths as $path) {
        if (is_dir($path)) {
            $files = array_merge($files, glob($path . '*'));
        }
    }
    
    return $files;
}

// Analyze multiple clients
$clientIds = [123, 124, 125]; // Client IDs to migrate
foreach ($clientIds as $clientId) {
    $scope = analyzeClientForMigration($clientId);
    echo "Client {$clientId}: {$scope['total_records']} records\n";
}
```

### Step 2: Create Migration Plan

Document the migration details:

```markdown
## Client Migration Plan

**Migration Date:** YYYY-MM-DD
**Migration Window:** 00:00 - 04:00 UTC

### Clients to Migrate
| Client ID | Client Name | Records | Estimated Time |
|-----------|-------------|---------|----------------|
| 123 | Acme Corp | 15 | 10 min |
| 124 | Tech Solutions | 8 | 5 min |
| 125 | Web Services Inc | 22 | 15 min |

### Migration Checklist
- [ ] Backup source database
- [ ] Test migration on staging
- [ ] Notify clients
- [ ] Freeze account changes
- [ ] Export client data
- [ ] Import to destination
- [ ] Transfer service credentials
- [ ] Verify data integrity
- [ ] Update DNS if needed
- [ ] Notify client of completion
- [ ] Monitor for 48 hours

### Service Actions Required
**Client 123:**
- Transfer hosting: old-server -> new-server
- Update nameservers for 2 domains
- Reconfigure SSL certificates

**Client 124:**
- Transfer VPS service
- Update DNS records

**Client 125:**
- Multiple services - coordinated migration
```

### Step 3: Prepare Source System

Set up source for migration:

```bash
#!/bin/bash
# prepare_source.sh

set -euo pipefail

MIGRATION_ID="migration_$(date +%Y%m%d_%H%M%S)"
BACKUP_DIR="/var/backups/client_migration/$MIGRATION_ID"

mkdir -p "$BACKUP_DIR"

echo "Preparing source system for migration..."

# Create complete backup of affected clients
CLIENT_IDS="123,124,125"

# Backup client data
mysqldump -u root -p \
    --single-transaction \
    --tables tblclients \
    --where="id IN ($CLIENT_IDS)" \
    whmcs_main > "$BACKUP_DIR/clients.sql"

mysqldump -u root -p \
    --single-transaction \
    --tables tblcontacts \
    --where="userid IN ($CLIENT_IDS)" \
    whmcs_main > "$BACKUP_DIR/contacts.sql"

mysqldump -u root -p \
    --single-transaction \
    --tables tblhosting \
    --where="userid IN ($CLIENT_IDS)" \
    whmcs_main > "$BACKUP_DIR/services.sql"

mysqldump -u root -p \
    --single-transaction \
    --tables tbldomain \
    --where="userid IN ($CLIENT_IDS)" \
    whmcs_main > "$BACKUP_DIR/domains.sql"

echo "Source backup complete: $BACKUP_DIR"

# Lock client accounts (optional - prevents changes during migration)
php << 'PHP'
<?php
require_once __DIR__ . '/init.php';

$clientIds = [123, 124, 125];
foreach ($clientIds as $clientId) {
    Capsule::table('tblclients')
        ->where('id', $clientId)
        ->update(['status' => 'inmigration']);
}
echo "Client accounts marked for migration";
PHP
```

### Step 4: Export Client Data

Extract data from source system:

```php
<?php
// export_clients.php

require_once __DIR__ . '/init.php';

class ClientExporter
{
    private $exportDir;
    private $clientIds;
    
    public function __construct(array $clientIds, $exportDir)
    {
        $this->clientIds = $clientIds;
        $this->exportDir = $exportDir;
    }
    
    public function export()
    {
        echo "Starting client data export...\n";
        
        foreach ($this->clientIds as $clientId) {
            $this->exportClient($clientId);
        }
        
        $this->exportTranslations();
        $this->generateManifest();
        
        echo "Export complete!\n";
    }
    
    private function exportClient($clientId)
    {
        $clientDir = "{$this->exportDir}/client_{$clientId}";
        mkdir($clientDir, 0755, true);
        
        // Export client record
        $client = Capsule::table('tblclients')
            ->where('id', $clientId)
            ->first();
        file_put_contents(
            "{$clientDir}/client.json",
            json_encode($client, JSON_PRETTY_PRINT)
        );
        
        // Export contacts
        $contacts = Capsule::table('tblcontacts')
            ->where('userid', $clientId)
            ->get();
        file_put_contents(
            "{$clientDir}/contacts.json",
            json_encode($contacts, JSON_PRETTY_PRINT)
        );
        
        // Export services
        $services = Capsule::table('tblhosting')
            ->where('userid', $clientId)
            ->get();
        file_put_contents(
            "{$clientDir}/services.json",
            json_encode($services, JSON_PRETTY_PRINT)
        );
        
        // Export domains
        $domains = Capsule::table('tbldomain')
            ->where('userid', $clientId)
            ->get();
        file_put_contents(
            "{$clientDir}/domains.json",
            json_encode($domains, JSON_PRETTY_PRINT)
        );
        
        // Export invoices (last 12 months)
        $invoices = Capsule::table('tblinvoices')
            ->where('userid', $clientId)
            ->where('date', '>=', date('Y-m-d', strtotime('-12 months')))
            ->get();
        file_put_contents(
            "{$clientDir}/invoices.json",
            json_encode($invoices, JSON_PRETTY_PRINT)
        );
        
        // Export orders
        $orders = Capsule::table('tblorders')
            ->where('userid', $clientId)
            ->get();
        file_put_contents(
            "{$clientDir}/orders.json",
            json_encode($orders, JSON_PRETTY_PRINT)
        );
        
        // Export custom fields
        $customFields = Capsule::table('tblcustomfieldsvalues')
            ->where('relid', $clientId)
            ->get();
        file_put_contents(
            "{$clientDir}/custom_fields.json",
            json_encode($customFields, JSON_PRETTY_PRINT)
        );
        
        // Copy uploaded files
        $this->copyClientFiles($clientId, $clientDir);
        
        echo "Exported client {$clientId}\n";
    }
    
    private function copyClientFiles($clientId, $clientDir)
    {
        $sourcePaths = [
            "/var/www/whmcs/attachments/{$clientId}/" => 'attachments',
            "/var/www/whmcs/downloads/client_{$clientId}/" => 'downloads',
        ];
        
        foreach ($sourcePaths as $path => $name) {
            if (is_dir($path)) {
                $destPath = "{$clientDir}/{$name}";
                mkdir($destPath, 0755, true);
                exec("cp -r " . escapeshellarg($path) . " " . escapeshellarg($destPath));
            }
        }
    }
    
    private function exportTranslations()
    {
        // Export any client-specific language strings
        // (rarely needed but may be relevant)
    }
    
    private function generateManifest()
    {
        $manifest = [
            'export_date' => date('c'),
            'whmcs_version' => \App::getVersion(),
            'client_count' => count($this->clientIds),
            'clients' => $this->clientIds,
        ];
        
        file_put_contents(
            "{$this->exportDir}/manifest.json",
            json_encode($manifest, JSON_PRETTY_PRINT)
        );
    }
}

// Run export
$exporter = new ClientExporter(
    [123, 124, 125],
    '/var/backups/client_migration/migration_20240101_120000'
);
$exporter->export();
```

### Step 5: Transfer Data to Destination

Import data to destination WHMCS:

```php
<?php
// import_clients.php

require_once __DIR__ . '/init.php';

class ClientImporter
{
    private $sourceDir;
    private $idMapping = []; // old_id => new_id
    
    public function __construct($sourceDir)
    {
        $this->sourceDir = $sourceDir;
    }
    
    public function import()
    {
        $manifest = json_decode(
            file_get_contents("{$this->sourceDir}/manifest.json"),
            true
        );
        
        echo "Importing {$manifest['client_count']} clients...\n";
        
        foreach ($manifest['clients'] as $oldClientId) {
            $newClientId = $this->importClient($oldClientId);
            $this->idMapping[$oldClientId] = $newClientId;
        }
        
        // Update service records with new client IDs
        $this->updateServiceReferences();
        
        // Copy files
        $this->copyImportedFiles();
        
        // Save ID mapping
        file_put_contents(
            "{$this->sourceDir}/id_mapping.json",
            json_encode($this->idMapping, JSON_PRETTY_PRINT)
        );
        
        echo "Import complete!\n";
        echo "ID Mapping:\n";
        print_r($this->idMapping);
    }
    
    private function importClient($oldClientId)
    {
        $clientDir = "{$this->sourceDir}/client_{$oldClientId}";
        
        if (!is_dir($clientDir)) {
            throw new Exception("Client directory not found: {$clientDir}");
        }
        
        // Load client data
        $clientData = json_decode(
            file_get_contents("{$clientDir}/client.json"),
            true
        );
        
        // Remove old IDs
        unset($clientData['id']);
        unset($clientData['uuid']);
        
        // Generate new UUID
        $clientData['uuid'] = \ Ramsey\Uuid\Uuid::uuid4()->toString();
        
        // Insert client
        $newClientId = Capsule::table('tblclients')->insertGetId($clientData);
        
        // Import contacts
        $contacts = json_decode(
            file_get_contents("{$clientDir}/contacts.json"),
            true
        );
        foreach ($contacts as $contact) {
            unset($contact['id']);
            $contact['userid'] = $newClientId;
            Capsule::table('tblcontacts')->insert($contact);
        }
        
        // Import custom fields
        $customFields = json_decode(
            file_get_contents("{$clientDir}/custom_fields.json"),
            true
        );
        foreach ($customFields as $field) {
            unset($field['id']);
            $field['relid'] = $newClientId;
            Capsule::table('tblcustomfieldsvalues')->insert($field);
        }
        
        // Import domains (separate migration)
        $this->importDomains($clientDir, $newClientId);
        
        echo "Imported client {$oldClientId} as {$newClientId}\n";
        
        return $newClientId;
    }
    
    private function importDomains($clientDir, $newClientId)
    {
        $domains = json_decode(
            file_get_contents("{$clientDir}/domains.json"),
            true
        );
        
        foreach ($domains as $domain) {
            $oldDomainId = $domain['id'];
            unset($domain['id']);
            $domain['userid'] = $newClientId;
            
            $newDomainId = Capsule::table('tbldomain')->insertGetId($domain);
            $this->idMapping["domain_{$oldDomainId}"] = $newDomainId;
        }
    }
    
    private function updateServiceReferences()
    {
        // Update hosting services with new client IDs
        foreach ($this->idMapping as $oldClientId => $newClientId) {
            Capsule::table('tblhosting')
                ->where('userid', $oldClientId)
                ->update(['userid' => $newClientId]);
            
            // Update related records
            Capsule::table('tblorders')
                ->where('userid', $oldClientId)
                ->update(['userid' => $newClientId]);
            
            Capsule::table('tblinvoices')
                ->where('userid', $oldClientId)
                ->update(['userid' => $newClientId]);
        }
    }
    
    private function copyImportedFiles()
    {
        foreach ($this->idMapping as $oldClientId => $newClientId) {
            $clientDir = "{$this->sourceDir}/client_{$oldClientId}";
            
            // Copy attachments
            $attachments = "{$clientDir}/attachments/";
            if (is_dir($attachments)) {
                $dest = "/var/www/whmcs/attachments/{$newClientId}/";
                mkdir($dest, 0755, true);
                exec("cp -r " . escapeshellarg($attachments) . " " . escapeshellarg($dest));
            }
            
            // Copy downloads
            $downloads = "{$clientDir}/downloads/";
            if (is_dir($downloads)) {
                $dest = "/var/www/whmcs/downloads/client_{$newClientId}/";
                mkdir($dest, 0755, true);
                exec("cp -r " . escapeshellarg($downloads) . " " . escapeshellarg($dest));
            }
        }
    }
}

// Run import
$importer = new ClientImporter('/var/backups/client_migration/migration_20240101_120000');
$importer->import();
```

### Step 6: Handle Service Provisioning

Transfer hosting services:

```php
<?php
// transfer_services.php

require_once __DIR__ . '/init.php';

// Load ID mapping
$idMapping = json_decode(
    file_get_contents('/var/backups/client_migration/migration_20240101_120000/id_mapping.json'),
    true
);

// Get services to transfer
$services = Capsule::table('tblhosting')
    ->whereIn('userid', array_values($idMapping))
    ->get();

foreach ($services as $service) {
    echo "Processing service {$service->id}: {$service->domain}\n";
    
    // Determine provisioning action
    switch ($service->domainstatus) {
        case 'Active':
            // Suspend on old server, create on new
            provisionOnNewServer($service);
            break;
        case 'Pending':
            // Mark for provisioning
            markForProvisioning($service);
            break;
        case 'Suspended':
            // Just update server reference
            updateServerReference($service);
            break;
    }
}

function provisionOnNewServer($service)
{
    // Use WHMCS provisioning module
    $module = $service->servertype;
    
    if (!function_exists($module . '_CreateAccount')) {
        echo "Module {$module} not found\n";
        return false;
    }
    
    // Create account on new server
    $result = ServerFunctions::call($module, 'CreateAccount', $service->id);
    
    if ($result === 'success') {
        Capsule::table('tblhosting')
            ->where('id', $service->id)
            ->update(['domainstatus' => 'Active']);
        
        echo "Service {$service->id} provisioned successfully\n";
    }
    
    return $result;
}
```

### Step 7: Verify Migration

Validate imported data:

```php
<?php
// verify_migration.php

require_once __DIR__ . '/init.php';

class MigrationVerifier
{
    private $sourceDir;
    private $errors = [];
    
    public function __construct($sourceDir)
    {
        $this->sourceDir = $sourceDir;
    }
    
    public function verify()
    {
        $manifest = json_decode(
            file_get_contents("{$this->sourceDir}/manifest.json"),
            true
        );
        
        echo "Verifying migration for {$manifest['client_count']} clients...\n";
        
        foreach ($manifest['clients'] as $oldClientId) {
            $this->verifyClient($oldClientId);
        }
        
        if (empty($this->errors)) {
            echo "\nVerification PASSED - No errors found\n";
            return true;
        } else {
            echo "\nVerification FAILED - Errors found:\n";
            foreach ($this->errors as $error) {
                echo "  - {$error}\n";
            }
            return false;
        }
    }
    
    private function verifyClient($oldClientId)
    {
        $clientDir = "{$this->sourceDir}/client_{$oldClientId}";
        
        // Load source data
        $sourceClient = json_decode(
            file_get_contents("{$clientDir}/client.json"),
            true
        );
        
        // Check destination
        $destClient = Capsule::table('tblclients')
            ->where('email', $sourceClient['email'])
            ->first();
        
        if (!$destClient) {
            $this->errors[] = "Client {$oldClientId}: Not found in destination";
            return;
        }
        
        // Verify key fields
        $checks = [
            'email' => $sourceClient['email'] === $destClient->email,
            'firstname' => $sourceClient['firstname'] === $destClient->firstname,
            'lastname' => $sourceClient['lastname'] === $destClient->lastname,
        ];
        
        foreach ($checks as $field => $passed) {
            if (!$passed) {
                $this->errors[] = "Client {$oldClientId}: Field '{$field}' mismatch";
            }
        }
        
        // Count related records
        $sourceServices = json_decode(
            file_get_contents("{$clientDir}/services.json"),
            true
        );
        $destServiceCount = Capsule::table('tblhosting')
            ->where('userid', $destClient->id)
            ->count();
        
        if (count($sourceServices) !== $destServiceCount) {
            $this->errors[] = "Client {$oldClientId}: Service count mismatch";
        }
        
        echo "Client {$oldClientId} verified\n";
    }
}

// Run verification
$verifier = new MigrationVerifier('/var/backups/client_migration/migration_20240101_120000');
$success = $verifier->verify();
exit($success ? 0 : 1);
```

### Step 8: Post-Migration Tasks

Complete migration steps:

```bash
#!/bin/bash
# post_migration.sh

echo "Running post-migration tasks..."

# Clear WHMCS cache
php /var/www/whmcs/crons/cron.php?a=clearCache

# Rebuild search index
php /var/www/whmcs/crons/rebuildsearchindex.php

# Send migration completion notifications
php << 'PHP'
<?php
require_once __DIR__ . '/init.php';

// Notify migrated clients
$idMapping = json_decode(
    file_get_contents('/var/backups/client_migration/migration_20240101_120000/id_mapping.json'),
    true
);

foreach ($idMapping as $oldId => $newId) {
    $client = Capsule::table('tblclients')
        ->where('id', $newId)
        ->first();
    
    send_email(
        $client->email,
        'Migration Complete',
        'migration_complete',
        ['client_name' => $client->firstname]
    );
}
PHP

# Update source system (optional - mark clients as migrated)
php << 'PHP'
<?php
require_once __DIR__ . '/init.php';

// Mark clients as migrated (prevents duplicate imports)
$clientIds = [123, 124, 125];
foreach ($clientIds as $clientId) {
    Capsule::table('tblclients')
        ->where('id', $clientId)
        ->update(['status' => 'migrated']);
}
PHP

echo "Post-migration tasks complete"
```

## Verification Checklist

- [ ] Source backup created
- [ ] All client data exported
- [ ] Files transferred
- [ ] Data imported to destination
- [ ] Client records verified
- [ ] Services provisioned
- [ ] DNS updated if needed
- [ ] SSL certificates transferred
- [ ] Cache cleared
- [ ] Clients notified
- [ ] Monitoring established
- [ ] Post-migration support plan ready

## Related Skills and Documentation

- [WHMCS Database Migration](whmcs-database-migration-workflow.md)
- [WHMCS Domain Transfer](whmcs-domain-transfer-workflow.md)
- [WHMCS SSL Renewal](whmcs-ssl-renewal-workflow.md)
- WHMCS Client Management: https://docs.whmcs.com/Clients

## Notes

- Always backup before migration
- Test on staging first
- Communicate with clients about downtime
- Plan for DNS propagation time
- Keep source on standby for rollback
- Monitor for issues post-migration
