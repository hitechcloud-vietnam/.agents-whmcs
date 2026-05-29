# WHMCS Data Restoration Workflow

## Purpose
Restore data from backups and archives in WHMCS.

## Prerequisites
- WHMCS installation
- Admin access
- Valid backup/ archive available

## Step-by-Step Process

### Step 1: Create Data Restoration Handler

**Create hooks/data_restore.php:**
```php
<?php
/**
 * WHMCS Data Restoration System
 */

use WHMCS\Database\Capsule;
use WHMCS\Carbon;

class DataRestore {
    
    private $restoreLog = [];
    
    /**
     * Restore from backup file
     */
    public function restoreFromBackup($backupFile, $options = []) {
        $defaults = [
            'restore_clients' => true,
            'restore_services' => true,
            'restore_domains' => true,
            'restore_invoices' => true,
            'overwrite_existing' => false,
            'dry_run' => false
        ];
        $options = array_merge($defaults, $options);
        
        if (!file_exists($backupFile)) {
            throw new Exception("Backup file not found: $backupFile");
        }
        
        $extension = pathinfo($backupFile, PATHINFO_EXTENSION);
        
        switch ($extension) {
            case 'sql':
                return $this->restoreFromSQL($backupFile, $options);
            case 'json':
                return $this->restoreFromJSON($backupFile, $options);
            case 'csv':
                return $this->restoreFromCSV($backupFile, $options);
            default:
                throw new Exception("Unsupported backup format: $extension");
        }
    }
    
    /**
     * Restore from SQL dump
     */
    private function restoreFromSQL($backupFile, $options) {
        // Read SQL file
        $sql = file_get_contents($backupFile);
        
        // Parse and execute statements
        $statements = $this->parseSQL($sql);
        
        $results = [
            'statements' => 0,
            'errors' => []
        ];
        
        foreach ($statements as $statement) {
            try {
                if (!$options['dry_run']) {
                    Capsule::statement($statement);
                }
                $results['statements']++;
            } catch (Exception $e) {
                $results['errors'][] = $e->getMessage();
            }
        }
        
        return $results;
    }
    
    /**
     * Parse SQL statements
     */
    private function parseSQL($sql) {
        // Remove comments
        $sql = preg_replace('/--.*$/m', '', $sql);
        $sql = preg_replace('/\/\*.*?\*\//s', '', $sql);
        
        // Split by semicolon
        $statements = array_filter(array_map('trim', explode(';', $sql)));
        
        return $statements;
    }
    
    /**
     * Restore from JSON backup
     */
    private function restoreFromJSON($backupFile, $options) {
        $data = json_decode(file_get_contents($backupFile), true);
        
        if (!$data) {
            throw new Exception("Invalid JSON backup file");
        }
        
        $results = [
            'clients' => 0,
            'services' => 0,
            'domains' => 0,
            'errors' => []
        ];
        
        // Restore clients
        if ($options['restore_clients'] && isset($data['clients'])) {
            foreach ($data['clients'] as $client) {
                try {
                    if (!$options['dry_run']) {
                        $this->restoreClient($client, $options);
                    }
                    $results['clients']++;
                } catch (Exception $e) {
                    $results['errors'][] = 'Client: ' . $e->getMessage();
                }
            }
        }
        
        // Restore services
        if ($options['restore_services'] && isset($data['services'])) {
            foreach ($data['services'] as $service) {
                try {
                    if (!$options['dry_run']) {
                        $this->restoreService($service, $options);
                    }
                    $results['services']++;
                } catch (Exception $e) {
                    $results['errors'][] = 'Service: ' . $e->getMessage();
                }
            }
        }
        
        return $results;
    }
    
    /**
     * Restore from CSV
     */
    private function restoreFromCSV($backupFile, $options) {
        $handle = fopen($backupFile, 'r');
        
        $headers = fgetcsv($handle);
        $results = ['rows' => 0, 'errors' => []];
        
        while (($row = fgetcsv($handle)) !== false) {
            $data = array_combine($headers, $row);
            
            try {
                if (!$options['dry_run']) {
                    $this->restoreClient($data, $options);
                }
                $results['rows']++;
            } catch (Exception $e) {
                $results['errors'][] = $e->getMessage();
            }
        }
        
        fclose($handle);
        return $results;
    }
    
    /**
     * Restore single client
     */
    private function restoreClient($data, $options) {
        // Check if exists
        $existing = Capsule::table('tblclients')
            ->where('email', $data['email'])
            ->first();
        
        if ($existing) {
            if (!$options['overwrite_existing']) {
                throw new Exception("Client already exists: " . $data['email']);
            }
            
            // Update existing
            unset($data['id'], $data['datecreated']);
            Capsule::table('tblclients')
                ->where('id', $existing->id)
                ->update($data);
        } else {
            // Insert new
            $data['uuid'] = Capsule::raw('UUID()');
            $data['datecreated'] = $data['datecreated'] ?? Carbon::now()->toDateTimeString();
            Capsule::table('tblclients')->insert($data);
        }
    }
    
    /**
     * Restore single service
     */
    private function restoreService($data, $options) {
        // Check if exists
        $existing = Capsule::table('tblhosting')
            ->where('userid', $data['userid'])
            ->where('domain', $data['domain'])
            ->first();
        
        if ($existing) {
            if (!$options['overwrite_existing']) {
                throw new Exception("Service already exists");
            }
            
            unset($data['id']);
            Capsule::table('tblhosting')
                ->where('id', $existing->id)
                ->update($data);
        } else {
            Capsule::table('tblhosting')->insert($data);
        }
    }
    
    /**
     * Restore specific client by ID
     */
    public function restoreClientById($clientId, $archiveTable = 'archive_clients') {
        $archived = Capsule::table($archiveTable)
            ->where('id', $clientId)
            ->first();
        
        if (!$archived) {
            throw new Exception("Client not found in archive");
        }
        
        $data = (array)$archived;
        unset($data['archived_at'], $data['original_table'], $data['created_at'], $data['updated_at']);
        
        // Restore to main table
        Capsule::table('tblclients')->insert($data);
        
        // Restore related records
        $this->restoreRelatedRecords($clientId);
        
        return ['success' => true, 'client_id' => $clientId];
    }
    
    /**
     * Restore related records
     */
    private function restoreRelatedRecords($clientId) {
        // Services
        $services = Capsule::table('archive_services')
            ->where('userid', $clientId)
            ->get();
        
        foreach ($services as $service) {
            $data = (array)$service;
            unset($data['archived_at'], $data['original_table']);
            Capsule::table('tblhosting')->insert($data);
        }
        
        // Domains
        $domains = Capsule::table('archive_domains')
            ->where('userid', $clientId)
            ->get();
        
        foreach ($domains as $domain) {
            $data = (array)$domain;
            unset($data['archived_at'], $data['original_table']);
            Capsule::table('tbldomains')->insert($data);
        }
    }
    
    /**
     * Verify restore
     */
    public function verifyRestore($clientId) {
        $client = Capsule::table('tblclients')->where('id', $clientId)->first();
        
        if (!$client) {
            return ['verified' => false, 'message' => 'Client not found'];
        }
        
        $services = Capsule::table('tblhosting')->where('userid', $clientId)->count();
        $domains = Capsule::table('tbldomains')->where('userid', $clientId)->count();
        
        return [
            'verified' => true,
            'client' => [
                'id' => $client->id,
                'email' => $client->email,
                'name' => $client->firstname . ' ' . $client->lastname
            ],
            'services_count' => $services,
            'domains_count' => $domains
        ];
    }
    
    /**
     * Get restore history
     */
    public function getRestoreHistory() {
        return Capsule::table('mod_restore_logs')
            ->orderBy('created_at', 'desc')
            ->limit(50)
            ->get();
    }
}
```

### Step 2: Execute Restore

```php
<?php
/**
 * Execute data restoration
 */
$restorer = new DataRestore();

// Dry run first
$preview = $restorer->restoreFromBackup('/path/to/backup.json', [
    'dry_run' => true,
    'overwrite_existing' => false
]);

print_r($preview);

// Execute restore
if ($_POST['confirm'] ?? false) {
    $results = $restorer->restoreFromBackup('/path/to/backup.json', [
        'dry_run' => false,
        'overwrite_existing' => false
    ]);
    
    echo "Restore complete.\n";
    print_r($results);
}
```

## Best Practices
- Always backup current data first
- Test restore on staging
- Verify restored data
- Document restore procedures
- Test integrity checks
- Maintain restore logs
- Use version control
- Schedule regular tests
- Train staff on restore
- Keep backup copies secure
