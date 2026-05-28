# WHMCS Data Sync Skill
# Version: 1.0 | Updated: 2026-05-29

## Purpose

Guide for data synchronization patterns in WHMCS including external systems, bidirectional sync, and conflict resolution.

## When to Use

- Syncing with external billing systems
- CRM integration
- Inventory synchronization
- Multi-instance WHMCS sync

## Data Sync Patterns

### 1. Sync Manager Class

```php
<?php
namespace WHMCS\Sync;

class SyncManager {
    private array $syncConfig;
    private string $lockFile;
    private int $maxRetries = 3;

    public function __construct() {
        $this->lockFile = WHMCS_ROOT . 'storage/logs/sync.lock';
        $this->loadConfiguration();
    }

    public function synchronize(string $syncType, array $options = []): array {
        // Check if another sync is running
        if ($this->isLocked()) {
            return ['status' => 'locked', 'message' => 'Another sync is in progress'];
        }

        $this->acquireLock();

        try {
            $startTime = microtime(true);
            $result = $this->executeSync($syncType, $options);
            $duration = microtime(true) - $startTime;

            $this->logSync($syncType, $result, $duration);
            $this->releaseLock();

            return $result;

        } catch (\Exception $e) {
            $this->releaseLock();
            $this->logError($syncType, $e);
            throw $e;
        }
    }

    private function executeSync(string $syncType, array $options): array {
        $syncers = [
            'clients' => SyncClients::class,
            'services' => SyncServices::class,
            'invoices' => SyncInvoices::class,
            'domains' => SyncDomains::class,
            'tickets' => SyncTickets::class,
        ];

        if (!isset($syncers[$syncType])) {
            throw new \InvalidArgumentException("Unknown sync type: $syncType");
        }

        $syncer = new $syncers[$syncType]($this->syncConfig);
        return $syncer->execute($options);
    }

    private function isLocked(): bool {
        if (!file_exists($this->lockFile)) {
            return false;
        }

        $lockData = json_decode(file_get_contents($this->lockFile), true);
        $lockTime = $lockData['timestamp'] ?? 0;

        // Lock expires after 30 minutes
        if (time() - $lockTime > 1800) {
            $this->releaseLock();
            return false;
        }

        return true;
    }

    private function acquireLock(): void {
        $lockData = [
            'pid' => getmypid(),
            'timestamp' => time(),
            'type' => 'sync',
        ];

        file_put_contents($this->lockFile, json_encode($lockData), LOCK_EX);
    }

    private function releaseLock(): void {
        if (file_exists($this->lockFile)) {
            unlink($this->lockFile);
        }
    }

    private function loadConfiguration(): void {
        $this->syncConfig = [
            'endpoint' => Capsule::table('tblconfiguration')
                ->where('setting', 'sync_endpoint')
                ->first()->value ?? '',
            'api_key' => Capsule::table('tblconfiguration')
                ->where('setting', 'sync_api_key')
                ->first()->value ?? '',
            'batch_size' => 100,
            'retry_delay' => 5000, // milliseconds
        ];
    }

    private function logSync(string $type, SyncResult $result, float $timestamp): void {
        Capsule::table('mod_sync_logs')->insert([
            'sync_type' => $type,
            'status' => $result->getStatus(),
            'records_synced' => $result->getSyncedCount(),
            'records_failed' => $result->getFailedCount(),
            'duration_ms' => round($timestamp * 1000),
            'synced_at' => date('Y-m-d H:i:s'),
        ]);
    }

    private function logError(string $type, \Exception $e): void {
        Capsule::table('mod_sync_logs')->insert([
            'sync_type' => $type,
            'status' => 'error',
            'error_message' => $e->getMessage(),
            'synced_at' => date('Y-m-d H:i:s'),
        ]);

        logActivity("Sync error: {$type} - " . $e->getMessage());
    }
}
```

### 2. Client Syncer

```php
<?php
namespace WHMCS\Sync;

class SyncClients {
    private array $config;
    private array $localChanges = [];

    public function __construct(array $config) {
        $this->config = $config;
    }

    public function execute(array $options): SyncResult {
        $direction = $options['direction'] ?? 'both'; // inbound, outbound, both

        $result = new SyncResult();

        if (in_array($direction, ['outbound', 'both'])) {
            $outboundResult = $this->syncOutbound();
            $result->merge($outboundResult);
        }

        if (in_array($direction, ['inbound', 'both'])) {
            $inboundResult = $this->syncInbound();
            $result->merge($inboundResult);
        }

        return $result;
    }

    private function syncOutbound(): SyncResult {
        $result = new SyncResult();

        // Get clients modified since last sync
        $lastSync = $this->getLastSyncTime('clients_outbound');
        $clients = Capsule::table('tblclients')
            ->where('updated_at', '>', $lastSync)
            ->orWhere('created_at', '>', $lastSync)
            ->get();

        foreach ($clients as $client) {
            try {
                $data = $this->transformClientForExport($client);
                $this->pushToExternal($data);
                $result->incrementSynced();
                $this->markSynced('clients', $client->id, 'outbound');
            } catch (\Exception $e) {
                $result->incrementFailed($e->getMessage());
            }
        }

        $this->updateLastSyncTime('clients_outbound');

        return $result;
    }

    private function syncInbound(): SyncResult {
        $result = new SyncResult();
        $remoteClients = $this->fetchFromExternal();

        foreach ($remoteClients as $remoteClient) {
            $checsum = crc32(json_encode($remoteClient));
            $localClient = $this->findLocalClient($remoteClient['external_id']);

            if ($localClient && $this->isUnchanged($localClient, $remoteClient)) {
                continue; // Skip unchanged records
            }

            try {
                if ($localClient) {
                    $this->updateLocalClient($localClient->id, $remoteClient);
                    $this->markSynced('clients', $localClient->id, 'inbound');
                } else {
                    $newId = $this->createLocalClient($remoteClient);
                    $this->markSynced('clients', $newId, 'inbound');
                }
                $result->incrementSynced();
            } catch (\Exception $e) {
                $result->incrementFailed($e->getMessage());
            }
        }

        $this->updateLastSyncTime('clients_inbound');

        return $result;
    }

    private function transformClientForExport(array $client): array {
        return [
            'external_id' => $client['id'],
            'first_name' => $client['firstname'],
            'last_name' => $client['lastname'],
            'email' => $client['email'],
            'phone' => $client['phonenumber'],
            'company' => $client['companyname'],
            'address' => [
                'street' => $client['address1'],
                'city' => $client['city'],
                'state' => $client['state'],
                'postal_code' => $client['postcode'],
                'country' => $client['country'],
            ],
            'password' => $client['password'], // Encrypted
            'sync_checksum' => crc32(json_encode($client)),
        ];
    }

    private function fetchFromExternal(): array {
        $lastSync = $this->getLastSyncTime('clients_inbound');
        $ch = curl_init();

        curl_setopt_array($ch, [
            CURLOPT_URL => $this->config['endpoint'] . '/clients?since=' . urlencode($lastSync),
            CURLOPT_RETURNTRANSFER => true,
            CURLOPT_TIMEOUT => 60,
            CURLOPT_HTTPHEADER => [
                'Authorization: Bearer ' . $this->config['api_key'],
                'Accept: application/json',
            ],
        ]);

        $response = curl_exec($ch);
        $httpCode = curl_getinfo($ch, CURLINFO_HTTP_CODE);
        curl_close($ch);

        if ($httpCode >= 400) {
            throw new \Exception("External API error: HTTP $httpCode");
        }

        return json_decode($response, true) ?: [];
    }

    private function pushToExternal(array $data): void {
        $ch = curl_init();

        curl_setopt_array($ch, [
            CURLOPT_URL => $this->config['endpoint'] . '/clients',
            CURLOPT_RETURNTRANSFER => true,
            CURLOPT_POST => true,
            CURLOPT_POSTFIELDS => json_encode($data),
            CURLOPT_HTTPHEADER => [
                'Authorization: Bearer ' . $this->config['api_key'],
                'Content-Type: application/json',
            ],
        ]);

        $response = curl_exec($ch);
        $httpCode = curl_getinfo($ch, CURLINFO_HTTP_CODE);
        curl_close($ch);

        if ($httpCode >= 400) {
            throw new \Exception("Push failed: HTTP $httpCode");
        }
    }

    private function findLocalClient(string $externalId): ?object {
        return Capsule::table('mod_client_sync_map')
            ->where('external_id', $externalId)
            ->first();
    }

    private function isUnchanged(object $localClient, array $remoteData): bool {
        return $localClient->sync_checksum === $remoteData['sync_checksum'];
    }

    private function createLocalClient(array $data): int {
        $id = Capsule::table('tblclients')->insertGetId([
            'firstname' => $data['first_name'],
            'lastname' => $data['last_name'],
            'email' => $data['email'],
            'phonenumber' => $data['phone'] ?? '',
            'companyname' => $data['company'] ?? '',
            'address1' => $data['address']['street'] ?? '',
            'city' => $data['address']['city'] ?? '',
            'state' => $data['address']['state'] ?? '',
            'postcode' => $data['address']['postal_code'] ?? '',
            'country' => $data['address']['country'] ?? '',
            'datecreated' => date('Y-m-d H:i:s'),
            'password' => $data['password'] ?? '',
        ]);

        Capsule::table('mod_client_sync_map')->insert([
            'local_id' => $id,
            'external_id' => $data['external_id'],
            'sync_checksum' => $data['sync_checksum'],
            'synced_at' => date('Y-m-d H:i:s'),
        ]);

        return $id;
    }

    private function updateLocalClient(int $localId, array $data): void {
        Capsule::table('tblclients')
            ->where('id', $localId)
            ->update([
                'firstname' => $data['first_name'],
                'lastname' => $data['last_name'],
                'email' => $data['email'],
                'phonenumber' => $data['phone'] ?? '',
                'companyname' => $data['company'] ?? '',
                'address1' => $data['address']['street'] ?? '',
                'city' => $data['address']['city'] ?? '',
                'state' => $data['address']['state'] ?? '',
                'postcode' => $data['address']['postal_code'] ?? '',
                'country' => $data['address']['country'] ?? '',
                'updated_at' => date('Y-m-d H:i:s'),
            ]);

        Capsule::table('mod_client_sync_map')
            ->where('local_id', $localId)
            ->update([
                'sync_checksum' => $data['sync_checksum'],
                'synced_at' => date('Y-m-d H:i:s'),
            ]);
    }

    private function getLastSyncTime(string $type): string {
        $record = Capsule::table('mod_sync_markers')
            ->where('sync_type', $type)
            ->first();

        return $record ? $record->last_sync : date('Y-m-d H:i:s', strtotime('-1 day'));
    }

    private function updateLastSyncTime(string $type): void {
        Capsule::table('mod_sync_markers')
            ->updateOrInsert(
                ['sync_type' => $type],
                ['last_sync' => date('Y-m-d H:i:s')]
            );
    }

    private function markSynced(string $entity, int $id, string $direction): void {
        // Implementation for marking records as synced
    }
}
```

### 3. Sync Result Class

```php
<?php
namespace WHMCS\Sync;

class SyncResult {
    private int $syncedCount = 0;
    private int $failedCount = 0;
    private array $errors = [];
    private string $status = 'success';

    public function incrementSynced(): void {
        $this->syncedCount++;
    }

    public function incrementFailed(string $error = ''): void {
        $this->failedCount++;
        if ($error) {
            $this->errors[] = $error;
        }
        $this->status = 'partial';
    }

    public function merge(SyncResult $other): void {
        $this->syncedCount += $other->syncedCount;
        $this->failedCount += $other->failedCount;
        $this->errors = array_merge($this->errors, $other->errors);

        if ($other->status === 'failed') {
            $this->status = 'failed';
        } elseif ($other->status === 'partial' && $this->status === 'success') {
            $this->status = 'partial';
        }
    }

    public function getSyncedCount(): int { return $this->syncedCount; }
    public function getFailedCount(): int { return $this->failedCount; }
    public function getErrors(): array { return $this->errors; }
    public function getStatus(): string { return $this->status; }
}
```

### 4. Conflict Resolution

```php
<?php
namespace WHMCS\Sync;

class ConflictResolver {
    const STRATEGY_LAST_WRITE_WINS = 'last_write_wins';
    const STRATEGY_LOCAL_WINS = 'local_wins';
    const STRATEGY_REMOTE_WINS = 'remote_wins';
    const STRATEGY_MANUAL = 'manual';

    private string $strategy;

    public function __construct(string $strategy = self::STRATEGY_LAST_WRITE_WINS) {
        $this->strategy = $strategy;
    }

    public function resolve(array $localData, array $remoteData, array $fieldMapping): array {
        $localModified = $localData['updated_at'] ?? $localData['created_at'] ?? 0;
        $remoteModified = $remoteData['updated_at'] ?? time();

        switch ($this->strategy) {
            case self::STRATEGY_LAST_WRITE_WINS:
                return $remoteModified > $localModified ? $remoteData : $localData;

            case self::STRATEGY_LOCAL_WINS:
                return $localData;

            case self::STRATEGY_REMOTE_WINS:
                return $remoteData;

            case self::STRATEGY_MANUAL:
                $this->queueForManualResolution($localData, $remoteData, $fieldMapping);
                return null;

            default:
                throw new \InvalidArgumentException("Unknown strategy: {$this->strategy}");
        }
    }

    private function queueForManualResolution(array $localData, array $remoteData, array $fieldMapping): void {
        Capsule::table('mod_sync_conflicts')->insert([
            'entity_type' => 'client',
            'local_data' => json_encode($localData),
            'remote_data' => json_encode($remoteData),
            'field_mapping' => json_encode($fieldMapping),
            'status' => 'pending',
            'created_at' => date('Y-m-d H:i:s'),
        ]);
    }

    public function resolveFieldLevel(array $localData, array $remoteData, array $conflicts): array {
        $resolved = $localData;

        foreach ($conflicts as $field => $timestamp) {
            $localModified = $localData[$field . '_updated_at'] ?? 0;
            $remoteModified = $remoteData[$field . '_updated_at'] ?? 0;

            if ($remoteModified > $localModified) {
                $resolved[$field] = $remoteData[$field];
            }
        }

        return $resolved;
    }
}
```

### 5. Database Schema

```php
<?php
function createSyncTables(): void {
    Capsule::schema()->create('mod_sync_logs', function($t) {
        $t->increments('id');
        $t->string('sync_type'); // clients, services, etc.
        $t->string('status'); // success, partial, error
        $t->integer('records_synced')->unsigned()->default(0);
        $t->integer('records_failed')->unsigned()->default(0);
        $t->integer('duration_ms')->unsigned()->default(0);
        $t->text('error_message')->nullable();
        $t->timestamp('synced_at');
    });

    Capsule::schema()->create('mod_sync_markers', function($t) {
        $t->increments('id');
        $t->string('sync_type')->unique(); // clients_outbound, services_inbound, etc.
        $t->timestamp('last_sync');
    });

    Capsule::schema()->create('mod_sync_conflicts', function($t) {
        $t->increments('id');
        $t->string('entity_type');
        $t->integer('local_id')->unsigned();
        $t->string('external_id');
        $t->json('local_data');
        $t->json('remote_data');
        $t->string('status')->default('pending'); // pending, resolved
        $t->string('resolution'); // local, remote, merged
        $t->integer('resolved_by')->unsigned()->nullable();
        $t->timestamp('created_at');
        $t->timestamp('resolved_at')->nullable();
    });

    Capsule::schema()->create('mod_client_sync_map', function($t) {
        $t->increments('id');
        $t->integer('local_id')->unsigned()->unique();
        $t->string('external_id');
        $t->string('sync_checksum');
        $t->timestamp('synced_at');
    });
}
```

## Hook Integration

```php
<?php
// hooks.php
add_hook('DailyCronJob', 1, function($vars) {
    $manager = new SyncManager();

    // Sync clients
    $result = $manager->synchronize('clients', ['direction' => 'both']);

    // Sync services
    $result = $manager->synchronize('services', ['direction' => 'both']);

    if ($result['status'] === 'error') {
        sendAdminNotification('error', 'Sync failed', ['details' => $result]);
    }
});

add_hook('AfterClientUpdate', 1, function($vars) {
    // Queue client for next sync
    Capsule::table('mod_sync_queue')->insert([
        'entity_type' => 'client',
        'entity_id' => $vars['user_id'],
        'action' => 'update',
        'queued_at' => date('Y-m-d H:i:s'),
    ]);
});
```

## Checklist

- [ ] Sync manager class
- [ ] Entity syncer implementations (clients, services, invoices, etc.)
- [ ] Lock mechanism for preventing duplicate syncs
- [ ] Incremental sync with change tracking
- [ ] Conflict resolution strategies
- [ ] Error handling and retry logic
- [ ] Sync logging
- [ ] Manual conflict resolution queue
- [ ] Cron job integration

---

**Related Skills:**
- whmcs-api-integration
- whmcs-domain-sync
- whmcs-remote-import
- whmcs-cron-automation
