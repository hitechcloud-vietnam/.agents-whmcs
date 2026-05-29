# WHMCS Data Transfer Workflow

## Purpose
Transfer data between WHMCS installations or systems.

## Prerequisites
- WHMCS installation
- Source and destination access
- Admin access

## Step-by-Step Process

### Step 1: Create Data Transfer Handler

**Create hooks/data_transfer.php:**
```php
<?php
/**
 * WHMCS Data Transfer System
 */

use WHMCS\Database\Capsule;
use WHMCS\Carbon;

class DataTransfer {
    
    private $sourceConfig;
    private $destinationConfig;
    
    public function __construct($sourceConfig = [], $destinationConfig = []) {
        $this->sourceConfig = $sourceConfig;
        $this->destinationConfig = $destinationConfig;
    }
    
    /**
     * Export data package
     */
    public function exportPackage($clientIds, $options = []) {
        $defaults = [
            'include_services' => true,
            'include_domains' => true,
            'include_invoices' => true,
            'include_tickets' => true,
            'include_orders' => true,
            'include_notes' => true
        ];
        $options = array_merge($defaults, $options);
        
        $package = [
            'export_date' => Carbon::now()->toDateTimeString(),
            'whmcs_version' => \WHMCS\Config\Setting::getValue('Version'),
            'clients' => []
        ];
        
        foreach ($clientIds as $clientId) {
            $client = Capsule::table('tblclients')->where('id', $clientId)->first();
            
            if (!$client) continue;
            
            $clientData = [
                'client' => (array)$client,
                'services' => [],
                'domains' => [],
                'invoices' => [],
                'tickets' => [],
                'orders' => []
            ];
            
            if ($options['include_services']) {
                $clientData['services'] = Capsule::table('tblhosting')
                    ->where('userid', $clientId)
                    ->get()->toArray();
            }
            
            if ($options['include_domains']) {
                $clientData['domains'] = Capsule::table('tbldomains')
                    ->where('userid', $clientId)
                    ->get()->toArray();
            }
            
            if ($options['include_invoices']) {
                $clientData['invoices'] = Capsule::table('tblinvoices')
                    ->where('userid', $clientId)
                    ->get()->toArray();
            }
            
            if ($options['include_tickets']) {
                $clientData['tickets'] = Capsule::table('tbltickets')
                    ->where('userid', $clientId)
                    ->get()->toArray();
            }
            
            if ($options['include_orders']) {
                $clientData['orders'] = Capsule::table('tblorders')
                    ->where('userid', $clientId)
                    ->get()->toArray();
            }
            
            $package['clients'][$clientId] = $clientData;
        }
        
        return $package;
    }
    
    /**
     * Export to JSON file
     */
    public function exportToFile($clientIds, $filePath, $options = []) {
        $package = $this->exportPackage($clientIds, $options);
        
        $json = json_encode($package, JSON_PRETTY_PRINT | JSON_UNESCAPED_UNICODE);
        file_put_contents($filePath, $json);
        
        return [
            'file' => $filePath,
            'size' => filesize($filePath),
            'clients' => count($package['clients'])
        ];
    }
    
    /**
     * Import data package
     */
    public function importPackage($package, $options = []) {
        $defaults = [
            'update_existing' => false,
            'map_client_ids' => true,
            'generate_new_ids' => true
        ];
        $options = array_merge($defaults, $options);
        
        $results = [
            'imported' => 0,
            'updated' => 0,
            'errors' => []
        ];
        
        $clientIdMap = [];
        
        foreach ($package['clients'] as $originalId => $clientData) {
            try {
                $newClientId = $this->importClient($clientData, $options);
                $clientIdMap[$originalId] = $newClientId;
                $results['imported']++;
            } catch (Exception $e) {
                $results['errors'][] = "Client $originalId: " . $e->getMessage();
            }
        }
        
        // Import related data with ID mapping
        foreach ($clientIdMap as $originalId => $newId) {
            $clientData = $package['clients'][$originalId];
            
            try {
                if (!empty($clientData['services'])) {
                    $this->importServices($clientData['services'], $clientIdMap);
                }
                
                if (!empty($clientData['domains'])) {
                    $this->importDomains($clientData['domains'], $clientIdMap);
                }
                
            } catch (Exception $e) {
                $results['errors'][] = "Related data for $originalId: " . $e->getMessage();
            }
        }
        
        return $results;
    }
    
    /**
     * Import single client
     */
    private function importClient($data, $options) {
        $client = $data['client'];
        unset($client['id']);
        
        // Check if email exists
        $existing = Capsule::table('tblclients')
            ->where('email', $client['email'])
            ->first();
        
        if ($existing) {
            if ($options['update_existing']) {
                Capsule::table('tblclients')
                    ->where('id', $existing->id)
                    ->update($client);
                return $existing->id;
            }
            throw new Exception("Client already exists");
        }
        
        $client['uuid'] = Capsule::raw('UUID()');
        $client['datecreated'] = $client['datecreated'] ?? Carbon::now()->toDateTimeString();
        
        return Capsule::table('tblclients')->insertGetId($client);
    }
    
    /**
     * Import services
     */
    private function importServices($services, $clientIdMap) {
        foreach ($services as $service) {
            $originalClientId = $service->userid;
            
            if (!isset($clientIdMap[$originalClientId])) continue;
            
            unset($service->id);
            $service->userid = $clientIdMap[$originalClientId];
            
            Capsule::table('tblhosting')->insert((array)$service);
        }
    }
    
    /**
     * Import domains
     */
    private function importDomains($domains, $clientIdMap) {
        foreach ($domains as $domain) {
            $originalClientId = $domain->userid;
            
            if (!isset($clientIdMap[$originalClientId])) continue;
            
            unset($domain->id);
            $domain->userid = $clientIdMap[$originalClientId];
            
            Capsule::table('tbldomains')->insert((array)$domain);
        }
    }
    
    /**
     * Import from file
     */
    public function importFromFile($filePath, $options = []) {
        $package = json_decode(file_get_contents($filePath), true);
        
        if (!$package) {
            throw new Exception("Invalid package file");
        }
        
        return $this->importPackage($package, $options);
    }
}
```

### Step 2: Execute Transfer

```php
<?php
$transfer = new DataTransfer();

// Export clients to file
$export = $transfer->exportToFile([1, 2, 3], '/path/to/export.json');
echo "Exported {$export['clients']} clients\n";

// Import from file
$results = $transfer->importFromFile('/path/to/export.json', [
    'update_existing' => false
]);
print_r($results);
```

## Best Practices
- Verify data integrity
- Test on staging first
- Backup both systems
- Document transfer
- Monitor for errors
- Validate imported data
- Test functionality
