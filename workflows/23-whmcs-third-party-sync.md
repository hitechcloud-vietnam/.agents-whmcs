# WHMCS Third-Party Sync Workflow

## Overview
This workflow covers bidirectional synchronization with third-party services.

## Step 1: Sync Manager

```php
<?php
// src/Service/SyncManager.php

namespace WHMCS\Module\Addon\YourModule\Service;

use WHMCS\Database\Capsule;

class SyncManager
{
    private $syncLog;
    private $externalServices = [];

    public function __construct()
    {
        $this->syncLog = new SyncLogger();
    }

    public function registerService(string $name, ExternalApiService $service): void
    {
        $this->externalServices[$name] = $service;
    }

    public function fullSync(): array
    {
        $results = [
            'started_at' => date('Y-m-d H:i:s'),
            'operations' => []
        ];

        foreach ($this->externalServices as $name => $service) {
            $this->syncLog->log("Starting sync with: $name");

            try {
                $result = $this->syncWithService($name, $service);
                $results['operations'][$name] = [
                    'success' => true,
                    'synced' => $result
                ];
            } catch (\Exception $e) {
                $results['operations'][$name] = [
                    'success' => false,
                    'error' => $e->getMessage()
                ];
                $this->syncLog->error("$name sync failed: " . $e->getMessage());
            }
        }

        $results['completed_at'] = date('Y-m-d H:i:s');
        return $results;
    }

    public function incrementalSync(string $serviceName): array
    {
        if (!isset($this->externalServices[$serviceName])) {
            throw new \Exception("Service not registered: $serviceName");
        }

        $lastSync = $this->getLastSyncTime($serviceName);
        $service = $this->externalServices[$serviceName];

        return $this->syncChangesSince($serviceName, $service, $lastSync);
    }

    private function syncWithService(string $name, ExternalApiService $service): array
    {
        $results = [
            'created' => 0,
            'updated' => 0,
            'deleted' => 0
        ];

        // Sync clients
        $externalClients = $service->get('/clients');
        foreach ($externalClients as $extClient) {
            $result = $this->syncClient($extClient);
            if ($result === 'created') $results['created']++;
            if ($result === 'updated') $results['updated']++;
        }

        // Sync to external
        $localClients = Capsule::table('tblclients')->get();
        foreach ($localClients as $client) {
            $this->syncToExternal($service, $client);
        }

        return $results;
    }

    private function syncChangesSince(string $name, ExternalApiService $service, string $since): array
    {
        $results = [];

        // Get changes since last sync
        $changes = $service->get('/changes', ['since' => $since]);

        foreach ($changes as $change) {
            switch ($change['type']) {
                case 'create':
                case 'update':
                    $this->syncClient($change['data']);
                    break;
                case 'delete':
                    $this->handleExternalDelete($change['entity'], $change['id']);
                    break;
            }
        }

        // Update last sync time
        $this->updateLastSyncTime($name, date('Y-m-d H:i:s'));

        return $results;
    }

    private function syncClient(array $externalData): string
    {
        $existing = Capsule::table('tblclients')
            ->where('email', $externalData['email'])
            ->first();

        if ($existing) {
            Capsule::table('tblclients')
                ->where('id', $existing->id)
                ->update([
                    'firstname' => $externalData['first_name'],
                    'lastname' => $externalData['last_name'],
                    'companyname' => $externalData['company'] ?? '',
                    'updated_at' => date('Y-m-d H:i:s')
                ]);
            return 'updated';
        } else {
            Capsule::table('tblclients')->insert([
                'firstname' => $externalData['first_name'],
                'lastname' => $externalData['last_name'],
                'email' => $externalData['email'],
                'password' => password_hash(bin2hex(random_bytes(16)), PASSWORD_DEFAULT),
                'country' => $externalData['country'] ?? 'US',
                'datecreated' => date('Y-m-d H:i:s'),
                'updated_at' => date('Y-m-d H:i:s')
            ]);
            return 'created';
        }
    }

    private function syncToExternal(ExternalApiService $service, $client): void
    {
        try {
            $service->post('/clients', [
                'external_id' => $client->id,
                'email' => $client->email,
                'first_name' => $client->firstname,
                'last_name' => $client->lastname,
                'company' => $client->companyname,
                'country' => $client->country
            ]);
        } catch (\Exception $e) {
            logActivity("Failed to sync client {$client->id} to external: " . $e->getMessage());
        }
    }

    private function handleExternalDelete(string $entity, string $externalId): void
    {
        // Handle deletion based on entity type
    }

    private function getLastSyncTime(string $serviceName): string
    {
        $record = Capsule::table('mod_sync_status')
            ->where('service_name', $serviceName)
            ->first();

        return $record ? $record->last_sync : '1970-01-01 00:00:00';
    }

    private function updateLastSyncTime(string $serviceName, string $time): void
    {
        Capsule::table('mod_sync_status')
            ->updateOrInsert(
                ['service_name' => $serviceName],
                ['last_sync' => $time, 'updated_at' => date('Y-m-d H:i:s')]
            );
    }
}

class SyncLogger
{
    public function log(string $message): void
    {
        Capsule::table('mod_sync_logs')->insert([
            'level' => 'info',
            'message' => $message,
            'created_at' => date('Y-m-d H:i:s')
        ]);
    }

    public function error(string $message): void
    {
        Capsule::table('mod_sync_logs')->insert([
            'level' => 'error',
            'message' => $message,
            'created_at' => date('Y-m-d H:i:s')
        ]);
    }
}
```

## Verification Checklist

- [ ] Sync manager implemented
- [ ] Services registered correctly
- [ ] Full sync working
- [ ] Incremental sync working
- [ ] Conflict resolution working
- [ ] Sync logging configured
