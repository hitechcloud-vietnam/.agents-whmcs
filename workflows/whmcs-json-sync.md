# WHMCS JSON Synchronization Workflow

## Purpose
Synchronize WHMCS data with external systems using JSON format.

## Prerequisites
- WHMCS installation
- External API/system access
- PHP cURL support

## Step-by-Step Process

### Step 1: Create JSON Sync Manager

**Create hooks/json_sync.php:**
```php
<?php
/**
 * WHMCS JSON Synchronization Manager
 */

use WHMCS\Database\Capsule;
use WHMCS\Carbon;

class JSONSync {
    
    private $config;
    private $lastSyncFile;
    private $syncLog = [];
    
    public function __construct($config = []) {
        $defaults = [
            'api_url' => '',
            'api_key' => '',
            'sync_direction' => 'bidirectional', // inbound, outbound, bidirectional
            'sync_interval' => 300, // seconds
            'entity_types' => ['clients', 'services', 'domains']
        ];
        $this->config = array_merge($defaults, $config);
        $this->lastSyncFile = ROOTDIR . '/data/last_sync.json';
    }
    
    /**
     * Perform full synchronization
     */
    public function sync() {
        $startTime = microtime(true);
        
        // Check sync interval
        if (!$this->shouldSync()) {
            return ['status' => 'skipped', 'reason' => 'Sync interval not reached'];
        }
        
        $results = [
            'start_time' => Carbon::now()->toDateTimeString(),
            'inbound' => [],
            'outbound' => []
        ];
        
        // Determine sync direction
        if ($this->config['sync_direction'] === 'bidirectional' || 
            $this->config['sync_direction'] === 'inbound') {
            $results['inbound'] = $this->syncInbound();
        }
        
        if ($this->config['sync_direction'] === 'bidirectional' || 
            $this->config['sync_direction'] === 'outbound') {
            $results['outbound'] = $this->syncOutbound();
        }
        
        // Update last sync time
        $this->updateLastSync();
        
        $results['end_time'] = Carbon::now()->toDateTimeString();
        $results['duration'] = round(microtime(true) - $startTime, 2);
        
        // Log sync
        $this->logSync($results);
        
        return $results;
    }
    
    /**
     * Check if sync should run
     */
    private function shouldSync() {
        $lastSync = $this->getLastSyncTime();
        if (!$lastSync) return true;
        
        $elapsed = time() - strtotime($lastSync);
        return $elapsed >= $this->config['sync_interval'];
    }
    
    /**
     * Get last sync timestamp
     */
    private function getLastSyncTime() {
        if (file_exists($this->lastSyncFile)) {
            $data = json_decode(file_get_contents($this->lastSyncFile), true);
            return $data['timestamp'] ?? null;
        }
        return null;
    }
    
    /**
     * Update last sync timestamp
     */
    private function updateLastSync() {
        $data = ['timestamp' => Carbon::now()->toDateTimeString()];
        file_put_contents($this->lastSyncFile, json_encode($data));
    }
    
    /**
     * Sync data from external system (inbound)
     */
    private function syncInbound() {
        $results = ['created' => 0, 'updated' => 0, 'errors' => []];
        
        // Fetch data from external system
        $externalData = $this->fetchExternalData();
        
        if (empty($externalData)) {
            return $results;
        }
        
        // Process each entity type
        foreach ($externalData as $entityType => $entities) {
            foreach ($entities as $entity) {
                try {
                    $result = $this->processInboundEntity($entityType, $entity);
                    if ($result['action'] === 'created') {
                        $results['created']++;
                    } elseif ($result['action'] === 'updated') {
                        $results['updated']++;
                    }
                } catch (Exception $e) {
                    $results['errors'][] = $e->getMessage();
                }
            }
        }
        
        return $results;
    }
    
    /**
     * Fetch data from external API
     */
    private function fetchExternalData() {
        if (empty($this->config['api_url'])) {
            return [];
        }
        
        $lastSync = $this->getLastSyncTime();
        
        $ch = curl_init();
        curl_setopt_array($ch, [
            CURLOPT_URL => $this->config['api_url'] . '/sync?since=' . urlencode($lastSync ?? ''),
            CURLOPT_RETURNTRANSFER => true,
            CURLOPT_HTTPHEADER => [
                'Authorization: Bearer ' . $this->config['api_key'],
                'Content-Type: application/json',
                'Accept: application/json'
            ],
            CURLOPT_TIMEOUT => 60
        ]);
        
        $response = curl_exec($ch);
        $httpCode = curl_getinfo($ch, CURLINFO_HTTP_CODE);
        curl_close($ch);
        
        if ($httpCode !== 200 || empty($response)) {
            $this->syncLog[] = "Failed to fetch data: HTTP $httpCode";
            return [];
        }
        
        return json_decode($response, true) ?? [];
    }
    
    /**
     * Process inbound entity
     */
    private function processInboundEntity($entityType, $data) {
        switch ($entityType) {
            case 'clients':
                return $this->processInboundClient($data);
            case 'services':
                return $this->processInboundService($data);
            case 'domains':
                return $this->processInboundDomain($data);
            default:
                return ['action' => 'skipped', 'reason' => 'Unknown entity type'];
        }
    }
    
    /**
     * Process inbound client
     */
    private function processInboundClient($data) {
        $email = strtolower($data['email']);
        
        $existing = Capsule::table('tblclients')
            ->where('email', $email)
            ->first();
        
        if ($existing) {
            // Update existing
            $updateData = [
                'firstname' => $data['firstname'] ?? $existing->firstname,
                'lastname' => $data['lastname'] ?? $existing->lastname,
                'companyname' => $data['companyname'] ?? $existing->companyname,
                'phonenumber' => $data['phonenumber'] ?? $existing->phonenumber
            ];
            
            Capsule::table('tblclients')
                ->where('id', $existing->id)
                ->update($updateData);
            
            return ['action' => 'updated', 'id' => $existing->id];
        } else {
            // Create new
            $userId = Capsule::table('tblclients')->insertGetId([
                'uuid' => Capsule::raw('UUID()'),
                'firstname' => $data['firstname'],
                'lastname' => $data['lastname'],
                'email' => $email,
                'companyname' => $data['companyname'] ?? '',
                'datecreated' => Carbon::now()->toDateTimeString(),
                'password' => $data['password'] ?? $this->generatePassword()
            ]);
            
            return ['action' => 'created', 'id' => $userId];
        }
    }
    
    /**
     * Process inbound service
     */
    private function processInboundService($data) {
        $email = strtolower($data['client_email']);
        $domain = $data['domain'] ?? '';
        
        $client = Capsule::table('tblclients')
            ->where('email', $email)
            ->first();
        
        if (!$client) {
            throw new Exception("Client not found: $email");
        }
        
        $existing = Capsule::table('tblhosting')
            ->where('userid', $client->id)
            ->where('domain', $domain)
            ->first();
        
        if ($existing) {
            Capsule::table('tblhosting')
                ->where('id', $existing->id)
                ->update([
                    'recurringamount' => $data['recurring_amount'] ?? $existing->recurringamount,
                    'domainstatus' => $data['status'] ?? $existing->domainstatus
                ]);
            
            return ['action' => 'updated', 'id' => $existing->id];
        } else {
            $serviceId = Capsule::table('tblhosting')->insertGetId([
                'userid' => $client->id,
                'packageid' => $data['product_id'],
                'domain' => $domain,
                'regdate' => $data['reg_date'] ?? Carbon::now()->toDateString(),
                'nextduedate' => $data['next_due_date'] ?? Carbon::now()->toDateString(),
                'domainstatus' => $data['status'] ?? 'Active',
                'recurringamount' => $data['recurring_amount'] ?? 0
            ]);
            
            return ['action' => 'created', 'id' => $serviceId];
        }
    }
    
    /**
     * Process inbound domain
     */
    private function processInboundDomain($data) {
        $email = strtolower($data['client_email']);
        
        $client = Capsule::table('tblclients')
            ->where('email', $email)
            ->first();
        
        if (!$client) {
            throw new Exception("Client not found: $email");
        }
        
        $existing = Capsule::table('tbldomains')
            ->where('userid', $client->id)
            ->where('domain', $data['domain'])
            ->first();
        
        if ($existing) {
            Capsule::table('tbldomains')
                ->where('id', $existing->id)
                ->update([
                    'domainstatus' => $data['status'] ?? $existing->domainstatus,
                    'expirydate' => $data['expiry_date'] ?? $existing->expirydate
                ]);
            
            return ['action' => 'updated', 'id' => $existing->id];
        } else {
            $domainId = Capsule::table('tbldomains')->insertGetId([
                'userid' => $client->id,
                'domain' => $data['domain'],
                'registrationperiod' => $data['registration_period'] ?? 1,
                'regdate' => $data['reg_date'] ?? Carbon::now()->toDateString(),
                'domainstatus' => $data['status'] ?? 'Active',
                'nextduedate' => $data['next_due_date'] ?? Carbon::now()->toDateString(),
                'expirydate' => $data['expiry_date'] ?? Carbon::now()->addYear()->toDateString()
            ]);
            
            return ['action' => 'created', 'id' => $domainId];
        }
    }
    
    /**
     * Sync data to external system (outbound)
     */
    private function syncOutbound() {
        $results = ['clients' => 0, 'services' => 0, 'domains' => 0, 'errors' => []];
        
        $lastSync = $this->getLastSyncTime();
        
        // Sync clients
        $clients = $this->getModifiedClients($lastSync);
        if (!empty($clients)) {
            $result = $this->pushToExternal('clients', $clients);
            $results['clients'] = count($clients);
            if (!$result['success']) {
                $results['errors'][] = $result['error'];
            }
        }
        
        // Sync services
        $services = $this->getModifiedServices($lastSync);
        if (!empty($services)) {
            $result = $this->pushToExternal('services', $services);
            $results['services'] = count($services);
            if (!$result['success']) {
                $results['errors'][] = $result['error'];
            }
        }
        
        // Sync domains
        $domains = $this->getModifiedDomains($lastSync);
        if (!empty($domains)) {
            $result = $this->pushToExternal('domains', $domains);
            $results['domains'] = count($domains);
            if (!$result['success']) {
                $results['errors'][] = $result['error'];
            }
        }
        
        return $results;
    }
    
    /**
     * Get modified clients since last sync
     */
    private function getModifiedClients($since) {
        $query = Capsule::table('tblclients')
            ->select(['id', 'firstname', 'lastname', 'email', 'companyname', 
                      'datecreated', 'status']);
        
        if ($since) {
            $query->where('datecreated', '>=', $since);
        }
        
        return $query->get()->toArray();
    }
    
    /**
     * Get modified services since last sync
     */
    private function getModifiedServices($since) {
        $query = Capsule::table('tblhosting')
            ->join('tblclients', 'tblhosting.userid', '=', 'tblclients.id')
            ->select(['tblhosting.id', 'tblclients.email as client_email', 
                      'tblhosting.domain', 'tblhosting.domainstatus',
                      'tblhosting.regdate', 'tblhosting.nextduedate']);
        
        if ($since) {
            $query->where('tblhosting.regdate', '>=', $since);
        }
        
        return $query->get()->toArray();
    }
    
    /**
     * Get modified domains since last sync
     */
    private function getModifiedDomains($since) {
        $query = Capsule::table('tbldomains')
            ->join('tblclients', 'tbldomains.userid', '=', 'tblclients.id')
            ->select(['tbldomains.id', 'tblclients.email as client_email',
                      'tbldomains.domain', 'tbldomains.domainstatus',
                      'tbldomains.regdate', 'tbldomains.expirydate']);
        
        if ($since) {
            $query->where('tbldomains.regdate', '>=', $since);
        }
        
        return $query->get()->toArray();
    }
    
    /**
     * Push data to external system
     */
    private function pushToExternal($entityType, $data) {
        if (empty($this->config['api_url'])) {
            return ['success' => false, 'error' => 'API URL not configured'];
        }
        
        $ch = curl_init();
        curl_setopt_array($ch, [
            CURLOPT_URL => $this->config['api_url'] . '/sync/' . $entityType,
            CURLOPT_RETURNTRANSFER => true,
            CURLOPT_POST => true,
            CURLOPT_POSTFIELDS => json_encode([$entityType => $data]),
            CURLOPT_HTTPHEADER => [
                'Authorization: Bearer ' . $this->config['api_key'],
                'Content-Type: application/json'
            ],
            CURLOPT_TIMEOUT => 60
        ]);
        
        $response = curl_exec($ch);
        $httpCode = curl_getinfo($ch, CURLINFO_HTTP_CODE);
        curl_close($ch);
        
        if ($httpCode >= 200 && $httpCode < 300) {
            return ['success' => true];
        }
        
        return ['success' => false, 'error' => "HTTP $httpCode: $response"];
    }
    
    /**
     * Log sync operation
     */
    private function logSync($results) {
        $this->syncLog[] = Carbon::now()->toDateTimeString() . ': ' . json_encode($results);
        
        // Save to file
        $logFile = ROOTDIR . '/data/sync_log.json';
        file_put_contents($logFile, json_encode($this->syncLog));
    }
    
    /**
     * Generate password
     */
    private function generatePassword($length = 12) {
        $chars = 'abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ0123456789!@#$%';
        return substr(str_shuffle($chars), 0, $length);
    }
    
    /**
     * Export data as JSON
     */
    public function exportJSON($entityType, $filters = []) {
        switch ($entityType) {
            case 'clients':
                $data = $this->getModifiedClients($filters['since'] ?? null);
                break;
            case 'services':
                $data = $this->getModifiedServices($filters['since'] ?? null);
                break;
            case 'domains':
                $data = $this->getModifiedDomains($filters['since'] ?? null);
                break;
            default:
                $data = [];
        }
        
        return json_encode($data, JSON_PRETTY_PRINT | JSON_UNESCAPED_UNICODE);
    }
}
```

### Step 2: Execute Sync

```php
<?php
/**
 * Execute JSON sync
 */
$sync = new JSONSync([
    'api_url' => 'https://external-api.com/v1',
    'api_key' => 'your-api-key',
    'sync_direction' => 'bidirectional',
    'sync_interval' => 300
]);

$result = $sync->sync();

echo "Sync Complete\n";
echo "Status: " . $result['status'] ?? 'completed' . "\n";
echo "Duration: " . ($result['duration'] ?? 0) . "s\n";

if (isset($result['inbound'])) {
    echo "Inbound - Created: " . $result['inbound']['created'] . ", Updated: " . $result['inbound']['updated'] . "\n";
}

if (isset($result['outbound'])) {
    echo "Outbound - Clients: " . $result['outbound']['clients'] . ", Services: " . $result['outbound']['services'] . "\n";
}
```

## Best Practices
- Implement proper error handling
- Use transactions for data integrity
- Monitor sync performance
- Log all sync operations
- Handle rate limiting
- Implement retry logic
- Use incremental syncs
- Verify data consistency
- Test sync thoroughly
- Document sync configuration
