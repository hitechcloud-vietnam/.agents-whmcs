# WHMCS Database Events

## Skill Description
Implement database event listeners for WHMCS modules to track changes, trigger workflows, and maintain audit trails when records are created, updated, or deleted.

## Prerequisites
- WHMCS 7.0+ installation
- PHP 7.4+
- Understanding of observer pattern
- Database access

## Step-by-Step Implementation

### 1. Event Dispatcher
```php
<?php
// includes/database/EventDispatcher.php

namespace WHMCS\Module\YourModule\Database;

class EventDispatcher
{
    private static array $listeners = [];
    private static array $wildcardListeners = [];

    public static function listen(string $event, callable $listener): void
    {
        if (strpos($event, '*') !== false) {
            self::$wildcardListeners[$event][] = $listener;
        } else {
            self::$listeners[$event][] = $listener;
        }
    }

    public static function dispatch(string $event, array $data = []): void
    {
        $data['event'] = $event;
        $data['timestamp'] = time();

        // Dispatch exact match listeners
        if (isset(self::$listeners[$event])) {
            foreach (self::$listeners[$event] as $listener) {
                call_user_func($listener, $data);
            }
        }

        // Dispatch wildcard listeners
        foreach (self::$wildcardListeners as $pattern => $listeners) {
            if (self::matchesPattern($event, $pattern)) {
                foreach ($listeners as $listener) {
                    call_user_func($listener, $data);
                }
            }
        }
    }

    private static function matchesPattern(string $event, string $pattern): bool
    {
        $regex = '/^' . str_replace(['*', '/'], ['[^.]+', '\/'], preg_quote($pattern, '/')) . '$/';
        return (bool) preg_match($regex, $event);
    }

    public static function forget(string $event): void
    {
        unset(self::$listeners[$event]);
    }

    public static function flush(): void
    {
        self::$listeners = [];
        self::$wildcardListeners = [];
    }
}
```

### 2. Model Observer
```php
<?php
// includes/database/ModelObserver.php

namespace WHMCS\Module\YourModule\Database;

class ModelObserver
{
    private string $table;
    private string $primaryKey;

    public function __construct(string $table, string $primaryKey = 'id')
    {
        $this->table = $table;
        $this->primaryKey = $primaryKey;
    }

    public function created(array $data): void
    {
        EventDispatcher::dispatch("{$this->table}.created", [
            'table' => $this->table,
            'action' => 'create',
            'new_data' => $data,
            'id' => $data[$this->primaryKey] ?? null
        ]);

        $this->logChange('create', null, $data);
    }

    public function updated(array $oldData, array $newData): void
    {
        $changes = $this->calculateChanges($oldData, $newData);

        if (!empty($changes)) {
            EventDispatcher::dispatch("{$this->table}.updated", [
                'table' => $this->table,
                'action' => 'update',
                'old_data' => $oldData,
                'new_data' => $newData,
                'changes' => $changes,
                'id' => $newData[$this->primaryKey] ?? null
            ]);

            $this->logChange('update', $oldData, $newData);
        }
    }

    public function deleted(array $data): void
    {
        EventDispatcher::dispatch("{$this->table}.deleted", [
            'table' => $this->table,
            'action' => 'delete',
            'old_data' => $data,
            'id' => $data[$this->primaryKey] ?? null
        ]);

        $this->logChange('delete', $data, null);
    }

    private function calculateChanges(array $old, array $new): array
    {
        $changes = [];

        foreach ($new as $key => $value) {
            if (!array_key_exists($key, $old)) {
                continue;
            }

            if ($old[$key] !== $value) {
                $changes[$key] = [
                    'old' => $old[$key],
                    'new' => $value
                ];
            }
        }

        return $changes;
    }

    private function logChange(string $action, ?array $oldData, ?array $newData): void
    {
        global $db;

        $db->insert('mod_yourmodule_change_log', [
            'table_name' => $this->table,
            'action' => $action,
            'old_data' => $oldData ? json_encode($oldData) : null,
            'new_data' => $newData ? json_encode($newData) : null,
            'user_id' => $_SESSION['adminid'] ?? null,
            'ip_address' => $_SERVER['REMOTE_ADDR'] ?? '',
            'created_at' => date('Y-m-d H:i:s')
        ]);
    }
}
```

### 3. Automatic Observer Registration
```php
<?php
// includes/database/AutoObserver.php

namespace WHMCS\Module\YourModule\Database;

class AutoObserver
{
    private static array $observers = [];

    public static function register(string $table, ModelObserver $observer): void
    {
        self::$observers[$table] = $observer;
    }

    public static function getObserver(string $table): ?ModelObserver
    {
        return self::$observers[$table] ?? null;
    }

    public static function created(string $table, array $data): void
    {
        $observer = self::getObserver($table);

        if ($observer) {
            $observer->created($data);
        }
    }

    public static function updated(string $table, array $oldData, array $newData): void
    {
        $observer = self::getObserver($table);

        if ($observer) {
            $observer->updated($oldData, $newData);
        }
    }

    public static function deleted(string $table, array $data): void
    {
        $observer = self::getObserver($table);

        if ($observer) {
            $observer->deleted($data);
        }
    }

    public static function wrapOperation(string $table, callable $operation, string $type = 'update'): mixed
    {
        global $db;

        if ($type === 'update' || $type === 'delete') {
            // Get old data before operation
            $id = null;
            if (isset($operation['id'])) {
                $id = $operation['id'];
            }

            if ($id) {
                $oldResult = $db->select("SELECT * FROM {$table} WHERE id = ?", [$id]);
                $oldData = $oldResult[0] ?? null;
            }
        }

        // Execute operation
        $result = $operation();

        // Trigger events
        if ($type === 'create') {
            self::created($table, is_array($result) ? $result : ['id' => $result]);
        } elseif ($type === 'update' && isset($oldData)) {
            // Get new data
            $newResult = $db->select("SELECT * FROM {$table} WHERE id = ?", [$id]);
            $newData = $newResult[0] ?? null;

            if ($newData) {
                self::updated($table, $oldData, $newData);
            }
        } elseif ($type === 'delete' && isset($oldData)) {
            self::deleted($table, $oldData);
        }

        return $result;
    }
}
```

### 4. Event Listeners
```php
<?php
// includes/database/listeners/ServiceEventListener.php

namespace WHMCS\Module\YourModule\Database\Listeners;

class ServiceEventListener
{
    public function handle(array $data): void
    {
        $action = $data['action'] ?? '';
        $method = 'on' . ucfirst($action);

        if (method_exists($this, $method)) {
            $this->$method($data);
        }
    }

    public function onCreate(array $data): void
    {
        $serviceId = $data['new_data']['id'] ?? null;
        $userId = $data['new_data']['userid'] ?? null;

        if ($serviceId && $userId) {
            // Send welcome email
            $this->sendServiceWelcomeEmail($serviceId, $userId);

            // Update statistics
            $this->updateServiceStatistics($userId);

            // Create audit log
            logActivity("New service created: {$serviceId}");
        }
    }

    public function onUpdate(array $data): void
    {
        $changes = $data['changes'] ?? [];

        // Check for status change
        if (isset($changes['domainstatus'])) {
            $this->handleStatusChange($data);
        }

        // Check for date changes
        if (isset($changes['nextduedate'])) {
            $this->handleDueDateChange($data);
        }
    }

    public function onDelete(array $data): void
    {
        $serviceId = $data['old_data']['id'] ?? null;
        $userId = $data['old_data']['userid'] ?? null;

        if ($serviceId) {
            // Log termination
            logActivity("Service terminated: {$serviceId}");

            // Update statistics
            if ($userId) {
                $this->updateServiceStatistics($userId);
            }
        }
    }

    private function handleStatusChange(array $data): void
    {
        $newStatus = $data['changes']['domainstatus']['new'] ?? '';
        $serviceId = $data['id'] ?? 0;

        switch ($newStatus) {
            case 'Active':
                $this->activateService($serviceId);
                break;
            case 'Suspended':
                $this->suspendService($serviceId);
                break;
            case 'Terminated':
                $this->terminateService($serviceId);
                break;
        }
    }

    private function activateService(int $serviceId): void
    {
        logActivity("Service activated: {$serviceId}");
    }

    private function suspendService(int $serviceId): void
    {
        logActivity("Service suspended: {$serviceId}");
    }

    private function terminateService(int $serviceId): void
    {
        logActivity("Service terminated: {$serviceId}");
    }

    private function handleDueDateChange(array $data): void
    {
        $newDate = $data['changes']['nextduedate']['new'] ?? null;
        $serviceId = $data['id'] ?? 0;

        if ($newDate) {
            logActivity("Service {$serviceId} due date changed to {$newDate}");
        }
    }

    private function sendServiceWelcomeEmail(int $serviceId, int $userId): void
    {
        // Email sending logic
    }

    private function updateServiceStatistics(int $userId): void
    {
        global $db;

        $stats = $db->select(
            "SELECT
                COUNT(*) as total_services,
                SUM(CASE WHEN domainstatus = 'Active' THEN 1 ELSE 0 END) as active_services
             FROM tblhosting WHERE userid = ?",
            [$userId]
        );

        // Store or process statistics
    }
}
```

### 5. Database Event Hook
```php
<?php
// hooks.php

use WHMCS\Module\YourModule\Database\EventDispatcher;
use WHMCS\Module\YourModule\Database\AutoObserver;
use WHMCS\Module\YourModule\Database\ModelObserver;
use WHMCS\Module\YourModule\Database\Listeners\ServiceEventListener;

// Register observers
AutoObserver::register('tblhosting', new ModelObserver('tblhosting'));
AutoObserver::register('tblclients', new ModelObserver('tblclients'));
AutoObserver::register('tblinvoices', new ModelObserver('tblinvoices'));

// Register event listeners
EventDispatcher::listen('tblhosting.created', [new ServiceEventListener(), 'handle']);
EventDispatcher::listen('tblhosting.updated', [new ServiceEventListener(), 'handle']);
EventDispatcher::listen('tblhosting.deleted', [new ServiceEventListener(), 'handle']);

// Generic event logging
EventDispatcher::listen('*', function ($data) {
    logActivity('Database Event: ' . json_encode([
        'table' => $data['table'] ?? 'unknown',
        'action' => $data['action'] ?? 'unknown',
        'id' => $data['id'] ?? 'unknown'
    ]));
});
```

### 6. Change Log Table
```sql
CREATE TABLE IF NOT EXISTS mod_yourmodule_change_log (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    table_name VARCHAR(100) NOT NULL,
    action ENUM('create', 'update', 'delete') NOT NULL,
    old_data JSON,
    new_data JSON,
    user_id INT,
    ip_address VARCHAR(45),
    created_at DATETIME NOT NULL,
    INDEX idx_table_name (table_name),
    INDEX idx_action (action),
    INDEX idx_user_id (user_id),
    INDEX idx_created_at (created_at)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
```

## Common Pitfalls and Solutions

| Pitfall | Solution |
|---------|----------|
| Circular events | Implement event queue with depth limit |
| Performance impact | Make event handlers async where possible |
| Missing data | Always validate event data before processing |
| Event ordering | Implement priority system for listeners |
| Memory leaks | Unsubscribe listeners when done |

## Security Considerations

1. **Log all changes** - Maintain audit trail for compliance
2. **Validate event data** - Don't trust event data without validation
3. **Limit listener scope** - Don't perform long operations in listeners
4. **Secure change log** - Protect audit table from modification
5. **User attribution** - Always record who made changes

## Testing Checklist

- [ ] Test create event triggering
- [ ] Test update event with changes
- [ ] Test update event with no changes
- [ ] Test delete event triggering
- [ ] Test wildcard event listeners
- [ ] Test event ordering
- [ ] Test nested events
- [ ] Test performance impact

## Reference Links

- [Observer Pattern](https://en.wikipedia.org/wiki/Observer_pattern)
- [Laravel Events](https://laravel.com/docs/events)
- [Domain Events Pattern](https://martinfowler.com/articles/domainevents.html)
