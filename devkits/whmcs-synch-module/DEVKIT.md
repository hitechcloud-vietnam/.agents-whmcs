# WHMCS Sync Module DevKit
# Version: 1.0 | Updated: 2026-05-28

## DevKit Structure

```
devkits/whmcs-synch-module/
├── sync.php             # Sync controller
├── lib/
│   ├── SyncEngine.php    # Core sync engine
│   ├── Syncer.php        # Syncer implementations
│   └── Queue.php         # Sync queue handler
├── templates/
│   └── admin.tpl         # Admin interface
└── hooks.php            # Hook integrations
```

## Sync Controller Template

```php
<?php
/**
 * WHMCS Sync Module: {Module}
 * DevKit Template
 * 
 * Installation: Upload to modules/addons/{module}/
 */

if (!defined("WHMCS")) {
    die("Direct access denied");
}

use WHMCS\Database\Capsule;

/**
 * Config function
 */
function {module}_config(): array {
    return [
        'name' => '{Sync Module}',
        'description' => 'Synchronize data with external systems',
        'version' => '1.0',
        'author' => '{Author}',
    ];
}

/**
 * Activate
 */
function {module}_activate(): array {
    Capsule::schema()->create('mod_{module}_sync_log', function($t) {
        $t->increments('id');
        $t->string('sync_type');
        $t->string('entity_type');
        $t->integer('entity_id');
        $t->string('action'); // create, update, delete
        $t->string('status'); // pending, synced, failed
        $t->text('payload');
        $t->text('response');
        $t->timestamp('created_at');
        $t->timestamp('synced_at');
    });
    
    Capsule::schema()->create('mod_{module}_sync_queue', function($t) {
        $t->increments('id');
        $t->string('entity_type');
        $t->integer('entity_id');
        $t->string('action');
        $t->text('data');
        $t->integer('attempts');
        $t->string('status');
        $t->text('error_message');
        $t->timestamp('next_attempt');
        $t->timestamp('created_at');
    });
    
    Capsule::schema()->create('mod_{module}_sync_map', function($t) {
        $t->increments('id');
        $t->string('local_type');
        $t->integer('local_id');
        $t->string('remote_type');
        $t->string('remote_id');
        $t->timestamp('synced_at');
    });
    
    return ['status' => 'success', 'description' => 'Module activated'];
}

/**
 * Deactivate
 */
function {module}_deactivate(): array {
    Capsule::schema()->dropIfExists('mod_{module}_sync_log');
    Capsule::schema()->dropIfExists('mod_{module}_sync_queue');
    Capsule::schema()->dropIfExists('mod_{module}_sync_map');
    
    return ['status' => 'success'];
}

/**
 * Output function
 */
function {module}_output(array $vars): void {
    $action = $_REQUEST['action'] ?? 'dashboard';
    
    switch ($action) {
        case 'settings':
            {module}_showSettings();
            break;
        case 'sync':
            {module}_runSync();
            break;
        case 'logs':
            {module}_showLogs();
            break;
        case 'queue':
            {module}_showQueue();
            break;
        case 'test':
            {module}_testConnection();
            break;
        default:
            {module}_showDashboard();
    }
}

/**
 * Show Dashboard
 */
function {module}_showDashboard(): void {
    $stats = [
        'pending' => Capsule::table('mod_{module}_sync_queue')
            ->where('status', 'pending')
            ->count(),
        'failed' => Capsule::table('mod_{module}_sync_queue')
            ->where('status', 'failed')
            ->count(),
        'synced_today' => Capsule::table('mod_{module}_sync_log')
            ->whereDate('synced_at', date('Y-m-d'))
            ->count(),
        'total_synced' => Capsule::table('mod_{module}_sync_map')
            ->count(),
    ];
    
    echo <<<HTML
<div class="sync-module">
    <div class="row">
        <div class="col-md-12">
            <div class="panel panel-default">
                <div class="panel-heading">
                    <h3 class="panel-title">Data Synchronization</h3>
                </div>
                <div class="panel-body">
                    <div class="row">
                        <div class="col-md-3">
                            <div class="stat-box">
                                <div class="stat-value">{$stats['pending']}</div>
                                <div class="stat-label">Pending Sync</div>
                            </div>
                        </div>
                        <div class="col-md-3">
                            <div class="stat-box">
                                <div class="stat-value">{$stats['failed']}</div>
                                <div class="stat-label">Failed</div>
                            </div>
                        </div>
                        <div class="col-md-3">
                            <div class="stat-box">
                                <div class="stat-value">{$stats['synced_today']}</div>
                                <div class="stat-label">Synced Today</div>
                            </div>
                        </div>
                        <div class="col-md-3">
                            <div class="stat-box">
                                <div class="stat-value">{$stats['total_synced']}</div>
                                <div class="stat-label">Total Synced</div>
                            </div>
                        </div>
                    </div>
                </div>
            </div>
        </div>
    </div>
    
    <div class="row">
        <div class="col-md-12">
            <div class="panel panel-default">
                <div class="panel-heading">
                    <h3 class="panel-title">Sync Actions</h3>
                </div>
                <div class="panel-body">
                    <div class="btn-group">
                        <a href="?module={module}&action=sync&type=clients" class="btn btn-primary">
                            <i class="fa fa-users"></i> Sync Clients
                        </a>
                        <a href="?module={module}&action=sync&type=services" class="btn btn-primary">
                            <i class="fa fa-server"></i> Sync Services
                        </a>
                        <a href="?module={module}&action=sync&type=all" class="btn btn-success">
                            <i class="fa fa-sync"></i> Full Sync
                        </a>
                    </div>
                    
                    <div class="btn-group pull-right">
                        <a href="?module={module}&action=settings" class="btn btn-default">
                            <i class="fa fa-cog"></i> Settings
                        </a>
                        <a href="?module={module}&action=queue" class="btn btn-default">
                            <i class="fa fa-list"></i> Queue
                        </a>
                        <a href="?module={module}&action=logs" class="btn btn-default">
                            <i class="fa fa-file-alt"></i> Logs
                        </a>
                        <a href="?module={module}&action=test" class="btn btn-info">
                            <i class="fa fa-plug"></i> Test Connection
                        </a>
                    </div>
                </div>
            </div>
        </div>
    </div>
    
    <div class="row">
        <div class="col-md-12">
            <div class="panel panel-default">
                <div class="panel-heading">
                    <h3 class="panel-title">Recent Sync Activity</h3>
                </div>
                <div class="panel-body">
                    <table class="table table-striped">
                        <thead>
                            <tr>
                                <th>Time</th>
                                <th>Type</th>
                                <th>Entity</th>
                                <th>Action</th>
                                <th>Status</th>
                            </tr>
                        </thead>
                        <tbody>
HTML;
    
    $recentLogs = Capsule::table('mod_{module}_sync_log')
        ->orderBy('id', 'desc')
        ->limit(10)
        ->get();
    
    foreach ($recentLogs as $log) {
        $statusClass = $log->status === 'synced' ? 'success' : 'danger';
        echo "<tr>
            <td>{$log->synced_at}</td>
            <td>{$log->sync_type}</td>
            <td>{$log->entity_type} #{$log->entity_id}</td>
            <td>{$log->action}</td>
            <td><span class='label label-{$statusClass}'>{$log->status}</span></td>
        </tr>";
    }
    
    echo "</tbody></table></div></div></div></div></div>";
}
```

## Sync Engine Class

```php
<?php
namespace {Module};

use WHMCS\Database\Capsule;

class SyncEngine {
    
    private string $apiUrl;
    private string $apiKey;
    private int $timeout = 30;
    private array $config;
    
    public function __construct(array $config = []) {
        $this->config = $config;
        $this->apiUrl = $config['api_url'] ?? '';
        $this->apiKey = $config['api_key'] ?? '';
    }
    
    public function syncClients(): array {
        $clients = Capsule::table('tblclients')
            ->where('created_at', '>=', date('Y-m-d', strtotime('-1 day')))
            ->get();
        
        $synced = 0;
        $failed = 0;
        
        foreach ($clients as $client) {
            try {
                $this->syncClient($client);
                $synced++;
            } catch (\Exception $e) {
                $failed++;
                $this->logSync('client', $client->id, 'create', 'failed', null, $e->getMessage());
            }
        }
        
        return ['synced' => $synced, 'failed' => $failed];
    }
    
    public function syncClient(object $client): array {
        $data = [
            'local_id' => $client->id,
            'email' => $client->email,
            'first_name' => $client->firstname,
            'last_name' => $client->lastname,
            'company' => $client->companyname,
            'phone' => $client->phonenumber,
            'address' => [
                'street' => $client->address1,
                'city' => $client->city,
                'state' => $client->state,
                'postal_code' => $client->postcode,
                'country' => $client->country,
            ],
            'created_at' => $client->created_at,
        ];
        
        $response = $this->apiCall('POST', '/api/clients', $data);
        
        // Save mapping
        $this->saveMapping('client', $client->id, 'client', $response['id'] ?? '');
        
        $this->logSync('client', $client->id, 'create', 'synced', $data, $response);
        
        return $response;
    }
    
    public function syncServices(): array {
        $services = Capsule::table('tblhosting')
            ->select(['tblhosting.*', 'tblclients.email', 'tblproducts.name as product_name'])
            ->join('tblclients', 'tblhosting.userid', '=', 'tblclients.id')
            ->join('tblproducts', 'tblhosting.packageid', '=', 'tblproducts.id')
            ->where('tblhosting.regdate', '>=', date('Y-m-d', strtotime('-1 day')))
            ->get();
        
        $synced = 0;
        $failed = 0;
        
        foreach ($services as $service) {
            try {
                $this->syncService($service);
                $synced++;
            } catch (\Exception $e) {
                $failed++;
            }
        }
        
        return ['synced' => $synced, 'failed' => $failed];
    }
    
    public function syncService(object $service): array {
        $data = [
            'local_id' => $service->id,
            'client_email' => $service->email,
            'product_name' => $service->product_name,
            'domain' => $service->domain,
            'status' => $service->domainstatus,
            'regdate' => $service->regdate,
            'next_due_date' => $service->nextduedate,
            'billing_cycle' => $service->billingcycle,
        ];
        
        $response = $this->apiCall('POST', '/api/services', $data);
        
        $this->saveMapping('service', $service->id, 'service', $response['id'] ?? '');
        $this->logSync('service', $service->id, 'create', 'synced', $data, $response);
        
        return $response;
    }
    
    public function fetchRemoteData(string $type, string $remoteId): array {
        return $this->apiCall('GET', "/api/{$type}/{$remoteId}");
    }
    
    public function updateRemote(string $type, string $remoteId, array $data): array {
        return $this->apiCall('PUT', "/api/{$type}/{$remoteId}", $data);
    }
    
    public function deleteRemote(string $type, string $remoteId): array {
        return $this->apiCall('DELETE', "/api/{$type}/{$remoteId}");
    }
    
    public function apiCall(string $method, string $endpoint, array $data = []): array {
        $ch = curl_init();
        
        $url = rtrim($this->apiUrl, '/') . $endpoint;
        
        curl_setopt_array($ch, [
            CURLOPT_URL => $url,
            CURLOPT_RETURNTRANSFER => true,
            CURLOPT_TIMEOUT => $this->timeout,
            CURLOPT_HTTPHEADER => [
                'Authorization: Bearer ' . $this->apiKey,
                'Content-Type: application/json',
            ],
        ]);
        
        if ($method === 'POST') {
            curl_setopt($ch, CURLOPT_POST, true);
            curl_setopt($ch, CURLOPT_POSTFIELDS, json_encode($data));
        } elseif ($method === 'PUT') {
            curl_setopt($ch, CURLOPT_CUSTOMREQUEST, 'PUT');
            curl_setopt($ch, CURLOPT_POSTFIELDS, json_encode($data));
        } elseif ($method === 'DELETE') {
            curl_setopt($ch, CURLOPT_CUSTOMREQUEST, 'DELETE');
        }
        
        $response = curl_exec($ch);
        $httpCode = curl_getinfo($ch, CURLINFO_HTTP_CODE);
        $error = curl_error($ch);
        curl_close($ch);
        
        if ($error) {
            throw new \Exception('cURL Error: ' . $error);
        }
        
        $result = json_decode($response, true) ?? [];
        
        if ($httpCode >= 400) {
            throw new \Exception('API Error: ' . ($result['message'] ?? "HTTP {$httpCode}"));
        }
        
        return $result;
    }
    
    private function saveMapping(string $localType, int $localId, string $remoteType, string $remoteId): void {
        Capsule::table('mod_{module}_sync_map')->updateOrInsert(
            ['local_type' => $localType, 'local_id' => $localId],
            [
                'remote_type' => $remoteType,
                'remote_id' => $remoteId,
                'synced_at' => date('Y-m-d H:i:s'),
            ]
        );
    }
    
    private function logSync(string $syncType, int $entityId, string $action, string $status, ?array $payload, ?array $response): void {
        Capsule::table('mod_{module}_sync_log')->insert([
            'sync_type' => $syncType,
            'entity_type' => $syncType,
            'entity_id' => $entityId,
            'action' => $action,
            'status' => $status,
            'payload' => json_encode($payload),
            'response' => json_encode($response),
            'created_at' => date('Y-m-d H:i:s'),
            'synced_at' => date('Y-m-d H:i:s'),
        ]);
    }
    
    public function addToQueue(string $entityType, int $entityId, string $action, array $data): void {
        Capsule::table('mod_{module}_sync_queue')->insert([
            'entity_type' => $entityType,
            'entity_id' => $entityId,
            'action' => $action,
            'data' => json_encode($data),
            'attempts' => 0,
            'status' => 'pending',
            'next_attempt' => date('Y-m-d H:i:s'),
            'created_at' => date('Y-m-d H:i:s'),
        ]);
    }
    
    public function processQueue(int $limit = 50): array {
        $pending = Capsule::table('mod_{module}_sync_queue')
            ->where('status', 'pending')
            ->where('next_attempt', '<=', date('Y-m-d H:i:s'))
            ->limit($limit)
            ->get();
        
        $processed = 0;
        
        foreach ($pending as $item) {
            try {
                $data = json_decode($item->data, true);
                $method = "sync" . ucfirst($item->entity_type);
                
                if (method_exists($this, $method)) {
                    $this->$method((object) $data);
                }
                
                Capsule::table('mod_{module}_sync_queue')
                    ->where('id', $item->id)
                    ->update(['status' => 'completed']);
                
                $processed++;
            } catch (\Exception $e) {
                Capsule::table('mod_{module}_sync_queue')
                    ->where('id', $item->id)
                    ->update([
                        'attempts' => $item->attempts + 1,
                        'error_message' => $e->getMessage(),
                        'next_attempt' => date('Y-m-d H:i:s', strtotime('+1 hour')),
                    ]);
            }
        }
        
        return ['processed' => $processed];
    }
}
```

## Sync Hooks

```php
<?php
/**
 * Sync Module Hooks
 */

if (!defined("WHMCS")) {
    die("Direct access denied");
}

// Sync client on creation
add_hook('ClientAdd', 1, function(array $vars) {
    $client = Capsule::table('tblclients')->where('id', $vars['userid'])->first();
    
    $engine = new \{Module}\SyncEngine([
        'api_url' => $config['api_url'],
        'api_key' => $config['api_key'],
    ]);
    
    try {
        $engine->syncClient($client);
    } catch (\Exception $e) {
        $engine->addToQueue('client', $vars['userid'], 'create', (array) $client);
    }
});

// Sync service on provisioning
add_hook('AfterModuleCreate', 1, function(array $vars) {
    $service = Capsule::table('tblhosting')
        ->where('id', $vars['serviceid'])
        ->first();
    
    $engine = new \{Module}\SyncEngine($config);
    
    try {
        $engine->syncService($service);
    } catch (\Exception $e) {
        $engine->addToQueue('service', $vars['serviceid'], 'create', (array) $service);
    }
});

// Daily queue processing
add_hook('DailyCronJob', 1, function() {
    $engine = new \{Module}\SyncEngine($config);
    $result = $engine->processQueue(100);
    logActivity("{Module}: Processed {$result['processed']} queue items");
});
```

## Checklist

```
Pre-Dev:
□ Identify external system API
□ Understand data structures
□ Plan sync frequency
□ Design mapping strategy
□ Plan conflict resolution

Development:
□ Create sync tables
□ Implement SyncEngine class
□ Add API client methods
□ Create sync methods (clients, services, etc.)
□ Add mapping table management
□ Create queue processing
□ Add retry logic
□ Create hook integrations
□ Build admin interface
□ Add logging

Testing:
□ Test API connection
□ Test client sync
□ Test service sync
□ Test queue processing
□ Test retry on failure
□ Test conflict resolution
□ Verify data integrity
□ Test with webhooks
□ Test scheduled sync
```