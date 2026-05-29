# WHMCS API-Based Sync Workflow

## Purpose
Synchronize WHMCS data with external systems using the WHMCS API.

## Prerequisites
- WHMCS installation
- API credentials
- External system access

## Step-by-Step Process

### Step 1: Create API Sync Client

**Create hooks/api_sync.php:**
```php
<?php
/**
 * WHMCS API Synchronization Client
 */

use WHMCS\Database\Capsule;
use WHMCS\Carbon;

class APISyncClient {
    
    private $apiUrl;
    private $apiKey;
    private $whmcsApiKey;
    
    public function __construct($config = []) {
        $this->apiUrl = $config['api_url'] ?? '';
        $this->apiKey = $config['api_key'] ?? '';
        $this->whmcsApiKey = $config['whmcs_api_key'] ?? '';
    }
    
    /**
     * Make WHMCS API call
     */
    public function whmcsApiCall($action, $postData = []) {
        $postData['action'] = $action;
        $postData['username'] = $postData['username'] ?? 'admin';
        $postData['password'] = hash('sha256', $this->whmcsApiKey);
        $postData['responsetype'] = 'json';
        
        $ch = curl_init();
        curl_setopt_array($ch, [
            CURLOPT_URL => rtrim(\WHMCS\Config\Setting::getValue('SystemURL'), '/') . '/includes/api.php',
            CURLOPT_POST => true,
            CURLOPT_POSTFIELDS => http_build_query($postData),
            CURLOPT_RETURNTRANSFER => true,
            CURLOPT_TIMEOUT => 30
        ]);
        
        $response = curl_exec($ch);
        curl_close($ch);
        
        return json_decode($response, true);
    }
    
    /**
     * Make external API call
     */
    public function externalApiCall($endpoint, $method = 'GET', $data = []) {
        $url = rtrim($this->apiUrl, '/') . '/' . ltrim($endpoint, '/');
        
        $ch = curl_init();
        curl_setopt_array($ch, [
            CURLOPT_URL => $url,
            CURLOPT_RETURNTRANSFER => true,
            CURLOPT_TIMEOUT => 30,
            CURLOPT_HTTPHEADER => [
                'Authorization: Bearer ' . $this->apiKey,
                'Content-Type: application/json'
            ]
        ]);
        
        if ($method === 'POST') {
            curl_setopt($ch, CURLOPT_POST, true);
            curl_setopt($ch, CURLOPT_POSTFIELDS, json_encode($data));
        }
        
        $response = curl_exec($ch);
        $httpCode = curl_getinfo($ch, CURLINFO_HTTP_CODE);
        curl_close($ch);
        
        return [
            'success' => $httpCode >= 200 && $httpCode < 300,
            'code' => $httpCode,
            'data' => json_decode($response, true)
        ];
    }
    
    /**
     * Sync clients to external system
     */
    public function syncClientsToExternal($lastSync = null) {
        $query = Capsule::table('tblclients');
        
        if ($lastSync) {
            $query->where('datecreated', '>=', $lastSync)
                  ->orWhere('updated_at', '>=', $lastSync);
        }
        
        $clients = $query->get();
        $synced = 0;
        $errors = [];
        
        foreach ($clients as $client) {
            $result = $this->externalApiCall('/clients', 'POST', [
                'external_id' => $client->id,
                'email' => $client->email,
                'first_name' => $client->firstname,
                'last_name' => $client->lastname,
                'company' => $client->companyname,
                'phone' => $client->phonenumber,
                'address' => [
                    'line1' => $client->address1,
                    'line2' => $client->address2,
                    'city' => $client->city,
                    'state' => $client->state,
                    'postal_code' => $client->postcode,
                    'country' => $client->country
                ]
            ]);
            
            if ($result['success']) {
                $synced++;
            } else {
                $errors[] = "Client {$client->id}: Error " . $result['code'];
            }
        }
        
        return ['synced' => $synced, 'errors' => $errors];
    }
    
    /**
     * Sync clients from external system
     */
    public function syncClientsFromExternal($lastSync = null) {
        $endpoint = '/clients';
        if ($lastSync) {
            $endpoint .= '?updated_since=' . urlencode($lastSync);
        }
        
        $result = $this->externalApiCall($endpoint);
        
        if (!$result['success']) {
            return ['synced' => 0, 'errors' => ['Failed to fetch clients']];
        }
        
        $synced = 0;
        $created = 0;
        $updated = 0;
        $errors = [];
        
        foreach ($result['data'] as $externalClient) {
            try {
                $existing = Capsule::table('tblclients')
                    ->where('email', strtolower($externalClient['email']))
                    ->first();
                
                if ($existing) {
                    Capsule::table('tblclients')
                        ->where('id', $existing->id)
                        ->update([
                            'firstname' => $externalClient['first_name'],
                            'lastname' => $externalClient['last_name'],
                            'companyname' => $externalClient['company'] ?? '',
                            'phonenumber' => $externalClient['phone'] ?? '',
                            'updated_at' => Carbon::now()->toDateTimeString()
                        ]);
                    $updated++;
                } else {
                    Capsule::table('tblclients')->insert([
                        'uuid' => Capsule::raw('UUID()'),
                        'firstname' => $externalClient['first_name'],
                        'lastname' => $externalClient['last_name'],
                        'email' => strtolower($externalClient['email']),
                        'companyname' => $externalClient['company'] ?? '',
                        'phonenumber' => $externalClient['phone'] ?? '',
                        'datecreated' => Carbon::now()->toDateTimeString()
                    ]);
                    $created++;
                }
                $synced++;
            } catch (Exception $e) {
                $errors[] = $e->getMessage();
            }
        }
        
        return [
            'synced' => $synced,
            'created' => $created,
            'updated' => $updated,
            'errors' => $errors
        ];
    }
    
    /**
     * Sync services
     */
    public function syncServices($lastSync = null) {
        // Similar implementation for services
    }
    
    /**
     * Sync orders/invoices
     */
    public function syncOrders($lastSync = null) {
        // Similar implementation for orders
    }
    
    /**
     * Full synchronization
     */
    public function fullSync() {
        $results = [
            'start_time' => Carbon::now()->toDateTimeString(),
            'clients' => [],
            'services' => [],
            'errors' => []
        ];
        
        // Get last sync time
        $lastSync = $this->getLastSyncTime();
        
        // Sync clients
        $results['clients'] = [
            'to_external' => $this->syncClientsToExternal($lastSync),
            'from_external' => $this->syncClientsFromExternal($lastSync)
        ];
        
        // Update sync time
        $this->updateLastSyncTime();
        
        $results['end_time'] = Carbon::now()->toDateTimeString();
        
        return $results;
    }
    
    /**
     * Get last sync timestamp
     */
    private function getLastSyncTime() {
        $setting = Capsule::table('tblsettings')
            ->where('setting', 'api_sync_last_run')
            ->first();
        
        return $setting ? $setting->value : null;
    }
    
    /**
     * Update last sync timestamp
     */
    private function updateLastSyncTime() {
        Capsule::table('tblsettings')
            ->updateOrInsert(
                ['setting' => 'api_sync_last_run'],
                ['value' => Carbon::now()->toDateTimeString()]
            );
    }
}
```

### Step 2: Execute API Sync

```php
<?php
/**
 * Execute API synchronization
 */
$sync = new APISyncClient([
    'api_url' => 'https://external-system.com/api/v1',
    'api_key' => 'your-api-key',
    'whmcs_api_key' => 'your-whmcs-api-key'
]);

$result = $sync->fullSync();

echo "API Sync Complete\n";
echo "Start: " . $result['start_time'] . "\n";
echo "End: " . $result['end_time'] . "\n";
echo "Clients synced: " . $result['clients']['to_external']['synced'] . "\n";
```

## Best Practices
- Implement rate limiting
- Use transactions for data integrity
- Log all sync operations
- Handle API errors gracefully
- Implement retry logic
- Monitor sync performance
- Use incremental syncs
- Verify data consistency
- Test sync thoroughly
- Document API endpoints
