# WHMCS Data Sync Automation Workflow

## Overview
This workflow implements automated data synchronization between WHMCS and external systems.

## Prerequisites
- WHMCS with API/database access
- External system API credentials
- Understanding of sync patterns

## Step-by-Step Process

### Step 1: Create Sync Manager
```php
<?php
// /includes/sync/SyncManager.php

namespace WHMCS\Sync;

class SyncManager
{
    private $syncLog = [];

    /**
     * Sync clients to external system
     */
    public function syncClients(array $options = []): array
    {
        $results = [
            'synced' => 0,
            'created' => 0,
            'updated' => 0,
            'errors' => 0
        ];

        $clients = Capsule::table('tblclients')
            ->where('status', 'Active')
            ->get();

        foreach ($clients as $client) {
            try {
                $result = $this->syncClient($client, $options['target']);

                $results['synced']++;

                if ($result['action'] === 'create') {
                    $results['created']++;
                } else {
                    $results['updated']++;
                }
            } catch (Exception $e) {
                $results['errors']++;
                $this->logSyncError('client', $client->id, $e->getMessage());
            }
        }

        return $results;
    }

    /**
     * Sync single client
     */
    private function syncClient($client, string $target): array
    {
        $targetSystem = $this->getTargetSystem($target);

        // Check if client exists in target
        $exists = $targetSystem->clientExists($client->id);

        if ($exists) {
            $targetSystem->updateClient($client);
            return ['action' => 'update', 'client_id' => $client->id];
        } else {
            $targetSystem->createClient($client);
            return ['action' => 'create', 'client_id' => $client->id];
        }
    }

    /**
     * Sync invoices
     */
    public function syncInvoices(string $target): array
    {
        $results = [
            'synced' => 0,
            'errors' => 0
        ];

        $invoices = Capsule::table('tblinvoices')
            ->where('status', 'Paid')
            ->where('synced_to', null)
            ->where('datepaid', '>=', date('Y-m-d', strtotime('-30 days')))
            ->get();

        foreach ($invoices as $invoice) {
            try {
                $this->syncInvoice($invoice, $target);
                $results['synced']++;

                Capsule::table('tblinvoices')
                    ->where('id', $invoice->id)
                    ->update(['synced_to' => $target, 'synced_at' => date('Y-m-d H:i:s')]);
            } catch (Exception $e) {
                $results['errors']++;
                $this->logSyncError('invoice', $invoice->id, $e->getMessage());
            }
        }

        return $results;
    }

    /**
     * Bidirectional sync with conflict resolution
     */
    public function bidirectionalSync(string $entityType, string $target): array
    {
        $results = ['pulled' => 0, 'pushed' => 0, 'conflicts' => 0];

        // Pull changes from external system
        $externalChanges = $this->getExternalChanges($entityType, $target);

        foreach ($externalChanges as $change) {
            $conflict = $this->resolveConflict($entityType, $change);

            if ($conflict) {
                $results['conflicts']++;
            }

            $results['pulled']++;
        }

        // Push local changes
        $pushResults = $this->pushLocalChanges($entityType, $target);
        $results['pushed'] = $pushResults['count'];

        return $results;
    }

    /**
     * Conflict resolution strategy
     */
    private function resolveConflict(string $entityType, array $change): bool
    {
        $strategy = getConfig("sync_conflict_resolution_{$entityType}") ?? 'local_wins';

        switch ($strategy) {
            case 'local_wins':
                // Keep local data, skip external change
                return true;

            case 'remote_wins':
                // Apply external change to local
                $this->applyExternalChange($entityType, $change);
                return true;

            case 'newest_wins':
                if (strtotime($change['updated_at']) > strtotime($change['local_updated_at'])) {
                    $this->applyExternalChange($entityType, $change);
                }
                return true;

            case 'manual':
                // Queue for manual review
                $this->queueForReview($entityType, $change);
                return true;

            default:
                return false;
        }
    }

    private function logSyncError(string $type, int $id, string $error)
    {
        Capsule::table('mod_sync_errors')->insert([
            'entity_type' => $type,
            'entity_id' => $id,
            'error' => $error,
            'created_at' => date('Y-m-d H:i:s')
        ]);
    }
}
```

### Step 2: Create Sync Hooks
```php
<?php
// /includes/hooks/sync_hooks.php

use WHMCS\Sync\SyncManager;

$syncManager = new SyncManager();

// Real-time sync on client changes
add_hook('ClientAdd', 1, function($vars) use ($syncManager) {
    $client = getClientsDetails($vars['userid']);
    $syncManager->syncClient($client, 'crm');
});

add_hook('ClientEdit', 1, function($vars) use ($syncManager) {
    $client = getClientsDetails($vars['userid']);
    $syncManager->syncClient($client, 'crm');
});

// Invoice sync
add_hook('InvoicePaid', 1, function($vars) use ($syncManager) {
    $invoice = getInvoice($vars['invoiceid']);
    $syncManager->syncInvoice($invoice, 'accounting');
});

// Daily bulk sync
add_hook('DailyCronJob', 1, function($vars) use ($syncManager) {
    $clientResults = $syncManager->syncClients(['target' => 'crm']);
    $invoiceResults = $syncManager->syncInvoices('accounting');

    return [
        'clients_synced' => $clientResults['synced'],
        'invoices_synced' => $invoiceResults['synced']
    ];
});
```

## Related Workflows
- [WHMCS CRM Integration](./whmcs-crm-integration.md)
- [WHMCS Accounting Sync](./whmcs-accounting-sync.md)