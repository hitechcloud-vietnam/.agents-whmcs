# Memento Pattern in WHMCS

The Memento Pattern captures and externalizes an object's internal state so it can be restored later. In WHMCS, this pattern is excellent for implementing undo functionality, state snapshots, and rollback capabilities for complex operations.

## Overview

Memento pattern stores object state without breaking encapsulation:
- Capture object state at a point in time
- Externalize state for later restoration
- Preserve encapsulation boundaries
- Support undo/redo operations
- Enable state history for auditing

## Core Structure

### Memento

```php
<?php
// includes/Memento/Memento.php

namespace CustomModule\Memento;

class Memento
{
    protected $state;
    protected string $timestamp;
    protected ?string $label;

    public function __construct($state, ?string $label = null)
    {
        $this->state = $state;
        $this->timestamp = date('Y-m-d H:i:s');
        $this->label = $label;
    }

    public function getState()
    {
        return $this->state;
    }

    public function getTimestamp(): string
    {
        return $this->timestamp;
    }

    public function getLabel(): ?string
    {
        return $this->label;
    }
}
```

## Real-World WHMCS Examples

### Client Service Memento

```php
<?php
// includes/Memento/ClientMemento.php

namespace CustomModule\Memento;

class ClientMemento extends Memento
{
    public function __construct(array $clientData, ?string $label = null)
    {
        parent::__construct($clientData, $label);
    }

    public function getClientId(): ?int
    {
        return $this->state['id'] ?? null;
    }

    public function getEmail(): ?string
    {
        return $this->state['email'] ?? null;
    }

    public function getSnapshotTime(): string
    {
        return $this->timestamp;
    }
}
```

```php
<?php
// includes/Memento/Originator.php

namespace CustomModule\Memento;

abstract class Originator
{
    protected array $history = [];
    protected int $maxHistorySize = 50;

    public function saveState(?string $label = null): Memento
    {
        $memento = $this->createMemento($label);
        $this->pushHistory($memento);
        return $memento;
    }

    public function restore(Memento $memento): void
    {
        $this->setState($memento->getState());
    }

    public function undo(): bool
    {
        $memento = $this->popHistory();
        if ($memento === null) {
            return false;
        }

        $this->setState($memento->getState());
        return true;
    }

    public function getHistory(): array
    {
        return $this->history;
    }

    public function clearHistory(): void
    {
        $this->history = [];
    }

    public function getHistorySize(): int
    {
        return count($this->history);
    }

    protected function pushHistory(Memento $memento): void
    {
        array_push($this->history, $memento);

        // Limit history size
        while (count($this->history) > $this->maxHistorySize) {
            array_shift($this->history);
        }
    }

    protected function popHistory(): ?Memento
    {
        return array_pop($this->history);
    }

    abstract protected function createMemento(?string $label): Memento;
    abstract protected function getState(): array;
    abstract protected function setState(array $state): void;
}
```

```php
<?php
// includes/Memento/ClientOriginator.php

namespace CustomModule\Memento;

use WHMCS\Database\Capsule;

class ClientOriginator extends Originator
{
    protected int $clientId;
    protected array $currentData = [];

    public function __construct(int $clientId)
    {
        $this->clientId = $clientId;
        $this->loadCurrentState();
    }

    protected function loadCurrentState(): void
    {
        $client = Capsule::table('tblclients')->find($this->clientId);

        if ($client) {
            $this->currentData = (array)$client;
        }
    }

    protected function createMemento(?string $label): Memento
    {
        return new ClientMemento($this->currentData, $label);
    }

    protected function getState(): array
    {
        return $this->currentData;
    }

    protected function setState(array $state): void
    {
        $this->currentData = $state;
    }

    public function update(array $data, bool $saveHistory = true): bool
    {
        // Save current state before update
        if ($saveHistory) {
            $this->saveState('before_update');
        }

        // Merge new data with current state
        $this->currentData = array_merge($this->currentData, $data);

        // Persist to database
        return Capsule::table('tblclients')
            ->where('id', $this->clientId)
            ->update($data) > 0;
    }

    public function updateWithRollback(array $data): bool
    {
        $previousState = $this->currentData;

        try {
            return $this->update($data, true);
        } catch (\Exception $e) {
            $this->currentData = $previousState;
            throw $e;
        }
    }

    public function getRestorePoints(): array
    {
        return array_map(function($memento) {
            return [
                'timestamp' => $memento->getTimestamp(),
                'label' => $memento->getLabel()
            ];
        }, $this->history);
    }

    public function restoreToTimestamp(string $timestamp): bool
    {
        foreach ($this->history as $memento) {
            if ($memento->getTimestamp() === $timestamp) {
                $this->restore($memento);

                Capsule::table('tblclients')
                    ->where('id', $this->clientId)
                    ->update($this->currentData);

                return true;
            }
        }

        return false;
    }
}
```

### Invoice Memento with Persistence

```php
<?php
// includes/Memento/MementoStore.php

namespace CustomModule\Memento;

use WHMCS\Database\Capsule;

class MementoStore
{
    protected string $entityType;
    protected int $entityId;

    public function __construct(string $entityType, int $entityId)
    {
        $this->entityType = $entityType;
        $this->entityId = $entityId;
    }

    public function save(Memento $memento): int
    {
        return Capsule::table('mod_mementos')->insertGetId([
            'entity_type' => $this->entityType,
            'entity_id' => $this->entityId,
            'state' => json_encode($memento->getState()),
            'label' => $memento->getLabel(),
            'created_at' => $memento->getTimestamp()
        ]);
    }

    public function load(int $id): ?Memento
    {
        $record = Capsule::table('mod_mementos')
            ->where('id', $id)
            ->where('entity_type', $this->entityType)
            ->where('entity_id', $this->entityId)
            ->first();

        if (!$record) {
            return null;
        }

        $state = json_decode($record->state, true);
        $memento = new Memento($state, $record->label);

        return $memento;
    }

    public function getHistory(int $limit = 20): array
    {
        $records = Capsule::table('mod_mementos')
            ->where('entity_type', $this->entityType)
            ->where('entity_id', $this->entityId)
            ->orderBy('created_at', 'desc')
            ->limit($limit)
            ->get();

        return array_map(function($record) {
            $state = json_decode($record->state, true);
            $memento = new Memento($state, $record->label);
            return $memento;
        }, $records->toArray());
    }

    public function getLatest(): ?Memento
    {
        $records = $this->getHistory(1);
        return $records[0] ?? null;
    }

    public function delete(int $id): bool
    {
        return Capsule::table('mod_mementos')
            ->where('id', $id)
            ->delete() > 0;
    }

    public function clearHistory(): int
    {
        return Capsule::table('mod_mementos')
            ->where('entity_type', $this->entityType)
            ->where('entity_id', $this->entityId)
            ->delete();
    }

    public function prune(int $keepCount): int
    {
        // Keep only the most recent N snapshots
        $toDelete = Capsule::table('mod_mementos')
            ->where('entity_type', $this->entityType)
            ->where('entity_id', $this->entityId)
            ->orderBy('created_at', 'desc')
            ->skip($keepCount)
            ->delete();

        return $toDelete;
    }
}
```

### Service State Memento

```php
<?php
// includes/Memento/ServiceStateMemento.php

namespace CustomModule\Memento;

class ServiceStateMemento extends Memento
{
    public function __construct(array $serviceData, ?string $label = null)
    {
        parent::__construct($serviceData, $label);
    }

    public function getServiceId(): ?int
    {
        return $this->state['id'] ?? null;
    }

    public function getStatus(): string
    {
        return $this->state['domainstatus'] ?? 'Unknown';
    }

    public function getDomain(): ?string
    {
        return $this->state['domain'] ?? null;
    }

    public function getUsername(): ?string
    {
        return $this->state['username'] ?? null;
    }
}
```

```php
<?php
// includes/Memento/ServiceOriginator.php

namespace CustomModule\Memento;

use WHMCS\Database\Capsule;

class ServiceOriginator extends Originator
{
    protected int $serviceId;
    protected array $currentState = [];
    protected MementoStore $store;

    public function __construct(int $serviceId)
    {
        $this->serviceId = $serviceId;
        $this->store = new MementoStore('service', $serviceId);
        $this->loadCurrentState();
    }

    protected function loadCurrentState(): void
    {
        $service = Capsule::table('tblhosting')->find($this->serviceId);

        if ($service) {
            $this->currentState = (array)$service;
        }
    }

    protected function createMemento(?string $label): Memento
    {
        return new ServiceStateMemento($this->currentState, $label);
    }

    protected function getState(): array
    {
        return $this->currentState;
    }

    protected function setState(array $state): void
    {
        $this->currentState = $state;
    }

    /**
     * Save state to persistent storage
     */
    public function saveToStorage(?string $label = null): int
    {
        $memento = $this->createMemento($label);
        return $this->store->save($memento);
    }

    /**
     * Restore from persistent storage
     */
    public function restoreFromStorage(int $mementoId): bool
    {
        $memento = $this->store->load($mementoId);

        if ($memento === null) {
            return false;
        }

        $this->restore($memento);

        Capsule::table('tblhosting')
            ->where('id', $this->serviceId)
            ->update($this->currentState);

        return true;
    }

    /**
     * Get all restore points from storage
     */
    public function getRestorePoints(): array
    {
        return $this->store->getHistory();
    }

    /**
     * Suspend service with state capture
     */
    public function suspendWithState(string $reason): bool
    {
        $this->saveToStorage("before_suspend_{$reason}");

        $this->currentState['domainstatus'] = 'Suspended';
        $this->currentState['suspension_reason'] = $reason;
        $this->currentState['suspended_at'] = date('Y-m-d H:i:s');

        return Capsule::table('tblhosting')
            ->where('id', $this->serviceId)
            ->update([
                'domainstatus' => 'Suspended',
                'suspension_reason' => $reason,
                'suspended_at' => date('Y-m-d H:i:s')
            ]) > 0;
    }

    /**
     * Unsuspend and restore previous state
     */
    public function unsuspendWithRestore(): bool
    {
        $previousState = $this->store->getLatest();

        if ($previousState === null) {
            // No state to restore, just activate
            $this->currentState['domainstatus'] = 'Active';
            return Capsule::table('tblhosting')
                ->where('id', $this->serviceId)
                ->update(['domainstatus' => 'Active']) > 0;
        }

        // Restore previous state
        $this->restore($previousState);

        // Clear suspension-specific fields
        unset($this->currentState['suspension_reason']);
        unset($this->currentState['suspended_at']);

        // Keep other state changes
        return Capsule::table('tblhosting')
            ->where('id', $this->serviceId)
            ->update($this->currentState) > 0;
    }

    /**
     * Terminate with full state archive
     */
    public function terminateWithArchive(): bool
    {
        // Save complete state
        $this->saveToStorage('pre_termination');

        $this->currentState['domainstatus'] = 'Terminated';
        $this->currentState['termination_date'] = date('Y-m-d H:i:s');

        // Archive related data
        $this->archiveRelatedData();

        return Capsule::table('tblhosting')
            ->where('id', $this->serviceId)
            ->update([
                'domainstatus' => 'Terminated',
                'termination_date' => date('Y-m-d H:i:s')
            ]) > 0;
    }

    protected function archiveRelatedData(): void
    {
        // Archive addons
        $addons = Capsule::table('tblhostingaddons')
            ->where('hostingid', $this->serviceId)
            ->get();

        foreach ($addons as $addon) {
            Capsule::table('mod_service_addon_archive')->insert([
                'service_id' => $this->serviceId,
                'addon_id' => $addon->id,
                'addon_data' => json_encode((array)$addon),
                'archived_at' => date('Y-m-d H:i:s')
            ]);
        }
    }
}
```

### Configuration Memento

```php
<?php
// includes/Memento/ConfigMemento.php

namespace CustomModule\Memento;

class ConfigMemento extends Memento
{
    protected string $version;
    protected ?string $author;

    public function __construct(array $config, ?string $label = null, ?string $version = null)
    {
        parent::__construct($config, $label);
        $this->version = $version ?? '1.0';
        $this->author = $_SESSION['adminid'] ?? 'system';
    }

    public function getVersion(): string
    {
        return $this->version;
    }

    public function getAuthor(): ?string
    {
        return $this->author;
    }
}
```

```php
<?php
// includes/Memento/ConfigOriginator.php

namespace CustomModule\Memento;

use WHMCS\Database\Capsule;

class ConfigOriginator extends Originator
{
    protected string $configKey;
    protected array $currentConfig = [];
    protected bool $isDirty = false;

    public function __construct(string $configKey)
    {
        $this->configKey = $configKey;
        $this->loadConfig();
    }

    protected function loadConfig(): void
    {
        $config = Capsule::table('mod_module_config')
            ->where('config_key', $this->configKey)
            ->first();

        if ($config) {
            $this->currentConfig = json_decode($config->config_value, true) ?? [];
        }
    }

    protected function createMemento(?string $label): Memento
    {
        return new ConfigMemento($this->currentConfig, $label);
    }

    protected function getState(): array
    {
        return $this->currentConfig;
    }

    protected function setState(array $state): void
    {
        $this->currentConfig = $state;
        $this->isDirty = true;
    }

    public function update(array $changes, ?string $changeDescription = null): bool
    {
        // Save current state
        $this->saveState($changeDescription ?? 'before_update');

        // Apply changes
        $this->currentConfig = array_merge($this->currentConfig, $changes);
        $this->isDirty = true;

        // Persist
        return $this->persist();
    }

    public function updateKey(string $key, $value, ?string $description = null): bool
    {
        return $this->update([$key => $value], $description ?? "before_changing_{$key}");
    }

    public function revertKey(string $key): bool
    {
        // Find last state with this key
        foreach (array_reverse($this->history) as $memento) {
            $state = $memento->getState();
            if (isset($state[$key])) {
                $this->currentConfig[$key] = $state[$key];
                $this->isDirty = true;
                return $this->persist();
            }
        }

        return false;
    }

    public function revertToVersion(string $label): bool
    {
        foreach ($this->history as $memento) {
            if ($memento->getLabel() === $label) {
                $this->restore($memento);
                $this->isDirty = true;
                return $this->persist();
            }
        }

        return false;
    }

    protected function persist(): bool
    {
        return Capsule::table('mod_module_config')
            ->updateOrInsert(
                ['config_key' => $this->configKey],
                [
                    'config_value' => json_encode($this->currentConfig),
                    'updated_at' => date('Y-m-d H:i:s')
                ]
            ) > 0;
    }

    public function isModified(): bool
    {
        return $this->isDirty;
    }

    public function getChangeHistory(int $limit = 10): array
    {
        return array_slice($this->history, -$limit);
    }
}
```

### Usage Examples

```php
<?php
// Client management with undo capability

$originator = new ClientOriginator(123);

// Update with history tracking
$originator->update([
    'firstname' => 'John',
    'lastname' => 'Smith'
], 'Updated name after marriage');

// Check history
echo "Current restore points: " . count($originator->getHistory()) . "\n";

// Undo the change
$originator->undo();

// Restore to specific point
$restorePoints = $originator->getRestorePoints();
if (!empty($restorePoints)) {
    $originator->restoreToTimestamp($restorePoints[0]['timestamp']);
}
```

```php
<?php
// Service suspension with rollback

$serviceOriginator = new ServiceOriginator(456);

// Suspend with state capture
$serviceOriginator->suspendWithState('Non-payment - Invoice #123');

// Later, restore and unsuspend
$serviceOriginator->unsuspendWithRestore();

// Get all restore points
$points = $serviceOriginator->getRestorePoints();
foreach ($points as $point) {
    echo "{$point->getTimestamp()}: {$point->getLabel()}\n";
}
```

```php
<?php
// Configuration with version history

$configOriginator = new ConfigOriginator('payment_gateway_settings');

// Make changes
$configOriginator->updateKey('api_key', 'new_api_key_here', 'API key rotation');
$configOriginator->updateKey('webhook_url', 'https://new-webhook.com', 'Webhook URL change');

// Revert specific key
$configOriginator->revertKey('webhook_url');

// Revert to specific version
$configOriginator->revertToVersion('before_update');

// Get change history
$history = $configOriginator->getChangeHistory(10);
```

### Snapshot Manager

```php
<?php
// includes/Memento/SnapshotManager.php

namespace CustomModule\Memento;

class SnapshotManager
{
    protected array $snapshots = [];
    protected string $storagePath;

    public function __construct(string $storagePath = null)
    {
        $this->storagePath = $storagePath ?? sys_get_temp_dir();
    }

    public function takeSnapshot(string $entityType, int $entityId, array $data, ?string $label = null): string
    {
        $snapshotId = uniqid("snap_{$entityType}_");

        $snapshot = [
            'id' => $snapshotId,
            'entity_type' => $entityType,
            'entity_id' => $entityId,
            'data' => $data,
            'label' => $label,
            'timestamp' => date('Y-m-d H:i:s')
        ];

        $this->snapshots[$snapshotId] = $snapshot;

        // Persist to disk
        $this->persistSnapshot($snapshot);

        return $snapshotId;
    }

    public function restoreSnapshot(string $snapshotId): ?array
    {
        // Check memory
        if (isset($this->snapshots[$snapshotId])) {
            return $this->snapshots[$snapshotId]['data'];
        }

        // Check disk
        $filePath = $this->getSnapshotPath($snapshotId);
        if (file_exists($filePath)) {
            $data = file_get_contents($filePath);
            return json_decode($data, true)['data'] ?? null;
        }

        return null;
    }

    public function getSnapshotInfo(string $snapshotId): ?array
    {
        if (isset($this->snapshots[$snapshotId])) {
            return $this->snapshots[$snapshotId];
        }

        $filePath = $this->getSnapshotPath($snapshotId);
        if (file_exists($filePath)) {
            return json_decode(file_get_contents($filePath), true);
        }

        return null;
    }

    public function listSnapshots(string $entityType, int $entityId): array
    {
        $matches = [];

        foreach ($this->snapshots as $snapshot) {
            if ($snapshot['entity_type'] === $entityType && $snapshot['entity_id'] === $entityId) {
                $matches[] = $snapshot;
            }
        }

        // Also check disk
        $files = glob($this->storagePath . "/snap_{$entityType}_{$entityId}_*.json");

        foreach ($files as $file) {
            $data = json_decode(file_get_contents($file), true);
            $matches[] = $data;
        }

        return $matches;
    }

    public function deleteSnapshot(string $snapshotId): bool
    {
        unset($this->snapshots[$snapshotId]);

        $filePath = $this->getSnapshotPath($snapshotId);
        if (file_exists($filePath)) {
            return unlink($filePath);
        }

        return false;
    }

    public function cleanupOldSnapshots(string $entityType, int $entityId, int $keepCount = 10): int
    {
        $snapshots = $this->listSnapshots($entityType, $entityId);

        // Sort by timestamp
        usort($snapshots, fn($a, $b) => strcmp($a['timestamp'], $b['timestamp']));

        $deleted = 0;
        while (count($snapshots) > $keepCount) {
            $oldest = array_shift($snapshots);
            if ($this->deleteSnapshot($oldest['id'])) {
                $deleted++;
            }
        }

        return $deleted;
    }

    protected function getSnapshotPath(string $snapshotId): string
    {
        return $this->storagePath . "/{$snapshotId}.json";
    }

    protected function persistSnapshot(array $snapshot): void
    {
        $filePath = $this->getSnapshotPath($snapshot['id']);
        file_put_contents($filePath, json_encode($snapshot, JSON_PRETTY_PRINT));
    }
}
```

## Pros

- **Encapsulation**: State stored without exposing internals
- **Undo Support**: Easy to restore previous states
- **Audit Trail**: Complete history of changes
- **Rollback**: Can revert problematic changes

## Cons

- **Memory**: Storing many states uses memory
- **Complexity**: More code than simple updates
- **Storage**: Persistent storage required for long-term snapshots

## Best Practices

1. Limit history size to prevent memory issues
2. Use persistent storage for critical data
3. Label snapshots for easy identification
4. Consider compression for large state objects
5. Implement cleanup for old snapshots