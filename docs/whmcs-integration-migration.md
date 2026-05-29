# WHMCS Migration Integration

Complete guide for migrating data to and from WHMCS.

## Overview

Migrate clients, services, and data between WHMCS installations and external systems.

## Data Export

### Export Clients

```php
<?php
/**
 * Export all clients
 */
function exportClients(int $limit = 1000, int $offset = 0): array
{
    $clients = Capsule::table('tblclients')
        ->offset($offset)
        ->limit($limit)
        ->get();
    
    $export = [];
    
    foreach ($clients as $client) {
        $export[] = [
            'id' => $client->id,
            'email' => $client->email,
            'first_name' => $client->firstname,
            'last_name' => $client->lastname,
            'company' => $client->companyname,
            'address1' => $client->address1,
            'address2' => $client->address2,
            'city' => $client->city,
            'state' => $client->state,
            'postcode' => $client->postcode,
            'country' => $client->country,
            'phonenumber' => $client->phonenumber,
            'status' => $client->status,
            'created_at' => $client->datecreated,
            'updated_at' => $client->dateupdated,
        ];
    }
    
    return $export;
}

/**
 * Export services for client
 */
function exportClientServices(int $clientId): array
{
    $services = Capsule::table('tblhosting')
        ->where('userid', $clientId)
        ->get();
    
    $export = [];
    
    foreach ($services as $service) {
        $product = Capsule::table('tblproducts')
            ->where('id', $service->packageid)
            ->first();
        
        $export[] = [
            'service_id' => $service->id,
            'domain' => $service->domain,
            'username' => $service->username,
            'product_name' => $product->name ?? 'Unknown',
            'status' => $service->domainstatus,
            'registration_date' => $service->regdate,
            'next_due_date' => $service->nextduedate,
            'billing_cycle' => $service->billingcycle,
            'amount' => $service->amount,
        ];
    }
    
    return $export;
}

/**
 * Full account export
 */
function exportFullAccount(int $clientId): array
{
    return [
        'client' => exportClients(1, 0)[0] ?? null,
        'services' => exportClientServices($clientId),
        'domains' => exportDomains($clientId),
        'invoices' => exportInvoices($clientId),
        'tickets' => exportTickets($clientId),
    ];
}
```

## Data Import

### Import Clients

```php
<?php
/**
 * Import client from array
 */
function importClient(array $data): array
{
    // Check if client exists
    $existing = Capsule::table('tblclients')
        ->where('email', $data['email'])
        ->first();
    
    if ($existing) {
        return [
            'success' => false,
            'error' => 'Client with this email already exists',
            'existing_id' => $existing->id,
        ];
    }
    
    try {
        $clientId = Capsule::table('tblclients')->insertGetId([
            'email' => $data['email'],
            'firstname' => $data['first_name'] ?? '',
            'lastname' => $data['last_name'] ?? '',
            'companyname' => $data['company'] ?? '',
            'address1' => $data['address1'] ?? '',
            'address2' => $data['address2'] ?? '',
            'city' => $data['city'] ?? '',
            'state' => $data['state'] ?? '',
            'postcode' => $data['postcode'] ?? '',
            'country' => $data['country'] ?? 'US',
            'phonenumber' => $data['phonenumber'] ?? '',
            'password' => $data['password'] ?? createRandomPassword(),
            'status' => $data['status'] ?? 'Active',
            'datecreated' => $data['created_at'] ?? date('Y-m-d'),
        ]);
        
        return [
            'success' => true,
            'client_id' => $clientId,
        ];
        
    } catch (Exception $e) {
        return [
            'success' => false,
            'error' => $e->getMessage(),
        ];
    }
}

/**
 * Batch import clients
 */
function batchImportClients(array $clients): array
{
    $results = [
        'imported' => 0,
        'skipped' => 0,
        'failed' => 0,
        'errors' => [],
    ];
    
    foreach ($clients as $clientData) {
        $result = importClient($clientData);
        
        if ($result['success']) {
            $results['imported']++;
        } elseif (isset($result['existing_id'])) {
            $results['skipped']++;
            $results['existing_ids'][] = $result['existing_id'];
        } else {
            $results['failed']++;
            $results['errors'][] = [
                'email' => $clientData['email'] ?? 'unknown',
                'error' => $result['error'],
            ];
        }
    }
    
    return $results;
}
```

### Import Services

```php
<?php
/**
 * Import service
 */
function importService(int $clientId, array $data): array
{
    // Verify client exists
    $client = Capsule::table('tblclients')
        ->where('id', $clientId)
        ->first();
    
    if (!$client) {
        return [
            'success' => false,
            'error' => 'Client not found',
        ];
    }
    
    // Find or create product
    $product = Capsule::table('tblproducts')
        ->where('name', $data['product_name'])
        ->first();
    
    if (!$product) {
        return [
            'success' => false,
            'error' => 'Product not found: ' . $data['product_name'],
        ];
    }
    
    try {
        $serviceId = Capsule::table('tblhosting')->insertGetId([
            'userid' => $clientId,
            'packageid' => $product->id,
            'server' => $data['server_id'] ?? 0,
            'domain' => $data['domain'],
            'username' => $data['username'],
            'password' => encrypt($data['password'] ?? ''),
            'regdate' => $data['registration_date'] ?? date('Y-m-d'),
            'domainstatus' => $data['status'] ?? 'Pending',
            'billingcycle' => $data['billing_cycle'] ?? 'Monthly',
            'nextduedate' => $data['next_due_date'] ?? date('Y-m-d'),
            'paymentmethod' => $data['payment_method'] ?? 'paypal',
        ]);
        
        return [
            'success' => true,
            'service_id' => $serviceId,
        ];
        
    } catch (Exception $e) {
        return [
            'success' => false,
            'error' => $e->getMessage(),
        ];
    }
}
```

## Migration Tools

### Migration Wizard

```php
<?php
/**
 * Migration session manager
 */
class MigrationSession
{
    private string $sessionId;
    private array $data = [];
    
    public function __construct()
    {
        $this->sessionId = session_id() . '_migration';
        $this->load();
    }
    
    /**
     * Start new migration
     */
    public function start(string $sourceType): void
    {
        $this->data = [
            'session_id' => uniqid('mig_'),
            'source_type' => $sourceType,
            'status' => 'in_progress',
            'progress' => 0,
            'total' => 0,
            'imported' => 0,
            'skipped' => 0,
            'failed' => 0,
            'started_at' => date('Y-m-d H:i:s'),
            'logs' => [],
        ];
        
        $this->save();
    }
    
    /**
     * Update progress
     */
    public function updateProgress(int $imported, int $total): void
    {
        $this->data['imported'] = $imported;
        $this->data['total'] = $total;
        $this->data['progress'] = $total > 0 ? round(($imported / $total) * 100) : 0;
        $this->save();
    }
    
    /**
     * Add log entry
     */
    public function log(string $message, string $type = 'info'): void
    {
        $this->data['logs'][] = [
            'timestamp' => date('Y-m-d H:i:s'),
            'type' => $type,
            'message' => $message,
        ];
        $this->save();
    }
    
    /**
     * Complete migration
     */
    public function complete(): void
    {
        $this->data['status'] = 'completed';
        $this->data['completed_at'] = date('Y-m-d H:i:s');
        $this->save();
    }
    
    /**
     * Save to database
     */
    private function save(): void
    {
        Capsule::table('mod_migration_sessions')
            ->updateOrInsert(
                ['session_id' => $this->sessionId],
                [
                    'data' => json_encode($this->data),
                    'updated_at' => date('Y-m-d H:i:s'),
                ]
            );
    }
    
    /**
     * Load from database
     */
    private function load(): void
    {
        $session = Capsule::table('mod_migration_sessions')
            ->where('session_id', $this->sessionId)
            ->first();
        
        if ($session) {
            $this->data = json_decode($session->data, true);
        }
    }
    
    public function getData(): array
    {
        return $this->data;
    }
}
```

## External Platform Migration

### cPanel Migration

```php
<?php
/**
 * Import from cPanel backup
 */
function importFromCpanelBackup(string $backupPath): array
{
    $results = [
        'clients' => 0,
        'services' => 0,
        'domains' => 0,
        'errors' => [],
    ];
    
    // Extract backup
    $extractDir = sys_get_temp_dir() . '/cpanel_migration_' . uniqid();
    mkdir($extractDir);
    
    exec("tar -xzf {$backupPath} -C {$extractDir}");
    
    // Parse accounts
    $accountsFile = $extractDir . '/accounts.json';
    if (file_exists($accountsFile)) {
        $accounts = json_decode(file_get_contents($accountsFile), true);
        
        foreach ($accounts as $account) {
            // Import client
            $clientResult = importClient([
                'email' => $account['email'],
                'first_name' => $account['contact'][0]['first_name'] ?? 'Migration',
                'last_name' => $account['contact'][0]['last_name'] ?? 'User',
                'company' => $account['contact'][0]['company'] ?? '',
                'address1' => $account['contact'][0]['address'] ?? '',
                'city' => $account['contact'][0]['city'] ?? '',
                'state' => $account['contact'][0]['state'] ?? '',
                'postcode' => $account['contact'][0]['zip'] ?? '',
                'country' => $account['contact'][0]['country'] ?? 'US',
                'phonenumber' => $account['contact'][0]['phone'] ?? '',
            ]);
            
            if ($clientResult['success']) {
                $results['clients']++;
                
                // Import domain/service
                $serviceResult = importService($clientResult['client_id'], [
                    'product_name' => 'cPanel Hosting',
                    'domain' => $account['domain'],
                    'username' => $account['username'],
                    'status' => 'Active',
                ]);
                
                if ($serviceResult['success']) {
                    $results['services']++;
                }
            } else {
                $results['errors'][] = $clientResult['error'];
            }
        }
    }
    
    // Cleanup
    exec("rm -rf {$extractDir}");
    
    return $results;
}
```

## Best Practices

1. **Backup first** - Always backup before migration
2. **Validate data** - Check for duplicates and errors
3. **Batch processing** - Process in chunks to avoid timeout
4. **Logging** - Track all migration activities
5. **Rollback** - Provide ability to undo migration
6. **Verify** - Check imported data after completion

## Related Documentation

- [whmcs-integration-api.md](whmcs-integration-api.md)
- [whmcs-integration-backup.md](whmcs-integration-backup.md)
