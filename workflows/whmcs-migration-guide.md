# WHMCS Migration Guide

## Purpose

Comprehensive guide for migrating data into WHMCS, including client data, services, domains, billing history, and custom fields from various source systems.

## Prerequisites

- WHMCS installation (source or destination)
- Source system database access
- Migration tool or custom migration script
- Data mapping documentation
- Test environment for validation

## Workflow Steps

### Step 1: Pre-Migration Planning

Document your migration requirements:

```php
// migration_config.php

return [
    'source_system' => 'legacy_billing',
    'whmcs_version' => '8.x',
    'batch_size' => 100,
    'dry_run' => true,
    
    // Data mapping
    'mapping' => [
        'clients' => [
            'source_table' => 'customers',
            'source_fields' => [
                'id' => 'customer_id',
                'email' => 'email_address',
                'firstname' => 'first_name',
                'lastname' => 'last_name',
                'companyname' => 'company_name',
                'address1' => 'street_address',
                'city' => 'city',
                'state' => 'state_province',
                'postcode' => 'zip_code',
                'country' => 'country_code',
                'phonenumber' => 'phone_number',
                'password' => 'password_hash',
                'created_at' => 'registration_date',
            ],
            'transformations' => [
                'country' => 'strtoupper',
                'created_at' => 'convertToWhmcsDate',
            ],
        ],
        'services' => [
            'source_table' => 'subscriptions',
            'source_fields' => [
                'id' => 'subscription_id',
                'client_id' => 'customer_id',
                'product_id' => 'product_code',
                'domain' => 'subscription_domain',
                'regdate' => 'start_date',
                'nextduedate' => 'next_billing_date',
                'billingcycle' => 'billing_interval',
            ],
        ],
        'domains' => [
            'source_table' => 'domain_names',
            'source_fields' => [
                'id' => 'domain_id',
                'client_id' => 'customer_id',
                'domain' => 'domain_name',
                'registrationdate' => 'registered_date',
                'expirydate' => 'expiration_date',
                'registrar' => 'registry_name',
            ],
        ],
    ],
    
    // Custom field mappings
    'custom_fields' => [
        'client' => [
            'source_custom_field' => 'loyalty_tier',
            'whmcs_field_name' => 'Loyalty Level',
            'type' => 'dropdown',
            'options' => 'Gold,Silver,Bronze',
        ],
    ],
];
```

### Step 2: Database Connection Setup

Configure connections to both source and destination:

```php
// MigrationDatabase.php

class MigrationDatabase
{
    private $source;
    private $whmcs;
    
    public function __construct(array $sourceConfig, array $whmcsConfig)
    {
        $this->source = new PDO(
            "mysql:host={$sourceConfig['host']};dbname={$sourceConfig['database']}",
            $sourceConfig['username'],
            $sourceConfig['password']
        );
        
        $this->whmcs = Capsule::connection()->getPdo();
    }
    
    /**
     * Fetch records from source database
     */
    public function fetchSourceRecords(string $table, array $fields, int $offset = 0, int $limit = 100): array
    {
        $fieldList = implode(', ', array_keys($fields));
        $paramPlaceholders = implode(', ', array_fill(0, count($fields), '?'));
        
        $sql = "SELECT {$fieldList} FROM {$table} LIMIT {$limit} OFFSET {$offset}";
        $stmt = $this->source->prepare($sql);
        $stmt->execute();
        
        $results = [];
        while ($row = $stmt->fetch(PDO::FETCH_ASSOC)) {
            // Map field names
            $mapped = [];
            foreach ($fields as $destField => $sourceField) {
                $mapped[$destField] = $row[$sourceField] ?? null;
            }
            $results[] = $mapped;
        }
        
        return $results;
    }
    
    /**
     * Get total count from source
     */
    public function getSourceCount(string $table): int
    {
        $stmt = $this->source->query("SELECT COUNT(*) FROM {$table}");
        return (int) $stmt->fetchColumn();
    }
}
```

### Step 3: Client Migration

Implement client data migration:

```php
// ClientMigration.php

class ClientMigration
{
    private $db;
    private $mapper;
    private $idMapping = []; // source_id => whmcs_id
    
    public function __construct(MigrationDatabase $db)
    {
        $this->db = $db;
        $this->mapper = new FieldMapper();
    }
    
    /**
     * Migrate all clients
     */
    public function migrateClients(int $batchSize = 100): MigrationResult
    {
        $total = $this->db->getSourceCount('customers');
        $processed = 0;
        $errors = [];
        
        $fields = [
            'email' => 'email_address',
            'firstname' => 'first_name',
            'lastname' => 'last_name',
            'companyname' => 'company_name',
            'address1' => 'street_address',
            'city' => 'city',
            'state' => 'state_province',
            'postcode' => 'zip_code',
            'country' => 'country_code',
            'phonenumber' => 'phone_number',
            'created_at' => 'registration_date',
        ];
        
        for ($offset = 0; $offset < $total; $offset += $batchSize) {
            $records = $this->db->fetchSourceRecords('customers', $fields, $offset, $batchSize);
            
            foreach ($records as $record) {
                try {
                    $result = $this->migrateSingleClient($record);
                    $this->idMapping[$record['source_customer_id']] = $result['client_id'];
                    $processed++;
                } catch (Exception $e) {
                    $errors[] = [
                        'source_id' => $record['source_customer_id'],
                        'error' => $e->getMessage(),
                    ];
                }
            }
        }
        
        return new MigrationResult($processed, $total, $errors);
    }
    
    /**
     * Migrate single client
     */
    private function migrateSingleClient(array $data): array
    {
        // Transform data
        $data = $this->transformClientData($data);
        
        // Check if client already exists
        $existing = Capsule::table('tblclients')
            ->where('email', $data['email'])
            ->first();
        
        if ($existing) {
            return ['client_id' => $existing->id, 'action' => 'skipped'];
        }
        
        // Generate password if available
        if (empty($data['password'])) {
            $data['password'] = generateRandomString(16);
            $data['password_type'] = 'random';
        }
        
        // Insert client
        $clientId = Capsule::table('tblclients')->insertGetId([
            'email' => $data['email'],
            'firstname' => $data['firstname'],
            'lastname' => $data['lastname'],
            'companyname' => $data['companyname'] ?? '',
            'address1' => $data['address1'] ?? '',
            'city' => $data['city'] ?? '',
            'state' => $data['state'] ?? '',
            'postcode' => $data['postcode'] ?? '',
            'country' => $data['country'] ?? 'US',
            'phonenumber' => $data['phonenumber'] ?? '',
            'password' => $this->hashPassword($data['password'] ?? ''),
            'emailverifystatus' => 1,
            'created_at' => $data['created_at'] ?? date('Y-m-d H:i:s'),
            'lastlogin' => null,
            'ip' => '',
            'host' => '',
        ]);
        
        return ['client_id' => $clientId, 'action' => 'created'];
    }
    
    /**
     * Transform client data
     */
    private function transformClientData(array $data): array
    {
        // Uppercase country code
        if (isset($data['country'])) {
            $data['country'] = strtoupper($data['country']);
        }
        
        // Convert date format
        if (isset($data['created_at'])) {
            $data['created_at'] = date('Y-m-d H:i:s', strtotime($data['created_at']));
        }
        
        // Validate required fields
        if (empty($data['email']) || !filter_var($data['email'], FILTER_VALIDATE_EMAIL)) {
            throw new Exception('Invalid email address');
        }
        
        return $data;
    }
    
    /**
     * Hash password for WHMCS
     */
    private function hashPassword(string $password): string
    {
        return password_hash($password, PASSWORD_DEFAULT);
    }
    
    /**
     * Get ID mapping for related records
     */
    public function getIdMapping(): array
    {
        return $this->idMapping;
    }
}

/**
 * Field mapping helper
 */
class FieldMapper
{
    public function map(array $source, array $mapping): array
    {
        $result = [];
        foreach ($mapping as $destField => $sourceField) {
            $result[$destField] = $source[$sourceField] ?? null;
        }
        return $result;
    }
    
    public function applyTransformations(array $data, array $transformations): array
    {
        foreach ($transformations as $field => $transform) {
            if (isset($data[$field]) && method_exists($this, $transform)) {
                $data[$field] = $this->$transform($data[$field]);
            }
        }
        return $data;
    }
}
```

### Step 4: Service Migration

Migrate hosting services and products:

```php
// ServiceMigration.php

class ServiceMigration
{
    private $db;
    private $clientMapping;
    private $productMapping = [];
    
    public function __construct(MigrationDatabase $db, array $clientMapping)
    {
        $this->db = $db;
        $this->clientMapping = $clientMapping;
    }
    
    /**
     * Set up product ID mapping
     */
    public function setProductMapping(array $mapping): void
    {
        $this->productMapping = $mapping;
    }
    
    /**
     * Migrate services
     */
    public function migrateServices(int $batchSize = 100): MigrationResult
    {
        $total = $this->db->getSourceCount('subscriptions');
        $processed = 0;
        $errors = [];
        
        $fields = [
            'id' => 'subscription_id',
            'client_id' => 'customer_id',
            'domain' => 'subscription_domain',
            'regdate' => 'start_date',
            'nextduedate' => 'next_billing_date',
            'billingcycle' => 'billing_interval',
            'amount' => 'monthly_price',
            'domainstatus' => 'status',
        ];
        
        for ($offset = 0; $offset < $total; $offset += $batchSize) {
            $records = $this->db->fetchSourceRecords('subscriptions', $fields, $offset, $batchSize);
            
            foreach ($records as $record) {
                try {
                    $this->migrateSingleService($record);
                    $processed++;
                } catch (Exception $e) {
                    $errors[] = [
                        'source_id' => $record['subscription_id'],
                        'error' => $e->getMessage(),
                    ];
                }
            }
        }
        
        return new MigrationResult($processed, $total, $errors);
    }
    
    /**
     * Migrate single service
     */
    private function migrateSingleService(array $data): int
    {
        // Map client ID
        $whmcsClientId = $this->clientMapping[$data['client_id']] ?? null;
        if (!$whmcsClientId) {
            throw new Exception('Client not found');
        }
        
        // Map product
        $productId = $this->productMapping[$data['product_id']] ?? null;
        if (!$productId) {
            throw new Exception('Product not found');
        }
        
        // Transform billing cycle
        $billingCycle = $this->mapBillingCycle($data['billingcycle']);
        
        // Transform status
        $status = $this->mapStatus($data['status']);
        
        // Insert service
        return Capsule::table('tblhosting')->insertGetId([
            'userid' => $whmcsClientId,
            'packageid' => $productId,
            'domain' => $data['domain'] ?? '',
            'regdate' => date('Y-m-d', strtotime($data['regdate'])),
            'nextduedate' => date('Y-m-d', strtotime($data['nextduedate'])),
            'billingcycle' => $billingCycle,
            'amount' => $data['amount'] ?? 0,
            'domainstatus' => $status,
        ]);
    }
    
    private function mapBillingCycle(string $cycle): string
    {
        $mapping = [
            'monthly' => 'Monthly',
            'quarterly' => 'Quarterly',
            'semiannually' => 'Semi-Annual',
            'annually' => 'Annual',
            'biennially' => 'Biennial',
            'triennially' => 'Triennial',
            'one-time' => 'One Time',
        ];
        
        return $mapping[strtolower($cycle)] ?? 'Monthly';
    }
    
    private function mapStatus(string $status): string
    {
        $mapping = [
            'active' => 'Active',
            'suspended' => 'Suspended',
            'cancelled' => 'Cancelled',
            'terminated' => 'Terminated',
            'pending' => 'Pending',
        ];
        
        return $mapping[strtolower($status)] ?? 'Active';
    }
}
```

### Step 5: Domain Migration

Migrate domain registrations:

```php
// DomainMigration.php

class DomainMigration
{
    private $db;
    private $clientMapping;
    
    public function __construct(MigrationDatabase $db, array $clientMapping)
    {
        $this->db = $db;
        $this->clientMapping = $clientMapping;
    }
    
    public function migrateDomains(int $batchSize = 100): MigrationResult
    {
        $total = $this->db->getSourceCount('domain_names');
        $processed = 0;
        $errors = [];
        
        $fields = [
            'id' => 'domain_id',
            'domain' => 'domain_name',
            'registrar' => 'registry_name',
            'registrationdate' => 'registered_date',
            'expirydate' => 'expiration_date',
            'status' => 'domain_status',
            'dns_management' => 'dns_enabled',
            'email_forwarding' => 'email_fwd_enabled',
            'id_protection' => 'privacy_enabled',
        ];
        
        for ($offset = 0; $offset < $total; $offset += $batchSize) {
            $records = $this->db->fetchSourceRecords('domain_names', $fields, $offset, $batchSize);
            
            foreach ($records as $record) {
                try {
                    $this->migrateSingleDomain($record);
                    $processed++;
                } catch (Exception $e) {
                    $errors[] = [
                        'domain' => $record['domain'],
                        'error' => $e->getMessage(),
                    ];
                }
            }
        }
        
        return new MigrationResult($processed, $total, $errors);
    }
    
    private function migrateSingleDomain(array $data): int
    {
        $whmcsClientId = $this->clientMapping[$data['client_id']] ?? null;
        
        return Capsule::table('tbldomains')->insertGetId([
            'userid' => $whmcsClientId ?? 0,
            'domain' => $data['domain'],
            'registrationdate' => date('Y-m-d', strtotime($data['registrationdate'])),
            'expirydate' => date('Y-m-d', strtotime($data['expirydate'])),
            'registrar' => $data['registrar'] ?? '',
            'registrationperiod' => 1,
            'status' => $this->mapDomainStatus($data['status']),
            'dnsmanagement' => $data['dns_management'] ? 1 : 0,
            'emailforwarding' => $data['email_forwarding'] ? 1 : 0,
            'idprotection' => $data['id_protection'] ? 1 : 0,
        ]);
    }
    
    private function mapDomainStatus(string $status): string
    {
        $mapping = [
            'active' => 'Active',
            'expired' => 'Expired',
            'pending' => 'Pending',
            'transferred' => 'Transferred Away',
        ];
        
        return $mapping[strtolower($status)] ?? 'Active';
    }
}
```

### Step 6: Migration Validation

Validate migrated data:

```php
// MigrationValidator.php

class MigrationValidator
{
    private $expectedCounts = [];
    
    public function setExpectedCounts(array $counts): void
    {
        $this->expectedCounts = $counts;
    }
    
    /**
     * Validate all migrated data
     */
    public function validateAll(): ValidationReport
    {
        $report = new ValidationReport();
        
        $report->addCheck('Client Count', $this->validateClientCount());
        $report->addCheck('Service Count', $this->validateServiceCount());
        $report->addCheck('Domain Count', $this->validateDomainCount());
        $report->addCheck('Email Validity', $this->validateEmails());
        $report->addCheck('Password Hashes', $this->validatePasswords());
        $report->addCheck('Data Integrity', $this->validateDataIntegrity());
        
        return $report;
    }
    
    private function validateClientCount(): CheckResult
    {
        $actual = Capsule::table('tblclients')->count();
        $expected = $this->expectedCounts['clients'] ?? $actual;
        
        return new CheckResult(
            $actual >= $expected * 0.95, // Allow 5% variance
            "Expected ~{$expected}, found {$actual} clients"
        );
    }
    
    private function validateEmails(): CheckResult
    {
        $invalid = Capsule::table('tblclients')
            ->whereRaw("email NOT LIKE '%@%.%'")
            ->count();
        
        return new CheckResult(
            $invalid === 0,
            "Found {$invalid} clients with invalid email addresses"
        );
    }
    
    private function validatePasswords(): CheckResult
    {
        $nullPasswords = Capsule::table('tblclients')
            ->whereNull('password')
            ->orWhere('password', '')
            ->count();
        
        return new CheckResult(
            $nullPasswords < 10, // Allow small number
            "Found {$nullPasswords} clients without passwords"
        );
    }
}
```

## Best Practices

1. **Always test on staging first** - Validate all data before production migration
2. **Back up everything** - Create complete backups before starting
3. **Use transactions** - Wrap migrations in transactions for rollback
4. **Process in batches** - Avoid memory issues with large datasets
5. **Log everything** - Maintain detailed migration logs
6. **Verify relationships** - Ensure foreign key consistency
7. **Notify users** - Inform clients about account changes
8. **Keep old system read-only** - Maintain for rollback during transition

## Common Pitfalls to Avoid

1. **Not mapping custom fields** - Lose important data
2. **Ignoring password migration** - Users must reset passwords
3. **Skipping validation** - Bad data causes long-term issues
4. **Not handling duplicates** - Merge conflicts break the system
5. **Forgetting email verification** - SPAM filters may block emails
6. **Migrating in production** - Always use a test environment first
7. **Not preserving billing history** - Audit trail is critical
8. **Rushing the process** - Take time for thorough validation
