# WHMCS Command Class Reference

**Version:** 8.x | **Updated:** 2026-05-29
**Related Skills:** `_command-pattern`, `queue-processing`, `service-layer`

---

## Overview

The Command Pattern in WHMCS encapsulates operations as objects, enabling:
- Parameterized operations
- Operation queuing
- Undo/redo functionality
- Async execution
- Audit trails

This reference covers command implementations, queue processing, and best practices for WHMCS module development.

---

## Command Interface

### Basic Interface

```php
<?php
/**
 * Command interface
 */
interface CommandInterface
{
    /**
     * Execute the command
     */
    public function execute(): mixed;

    /**
     * Undo the command
     */
    public function undo(): void;

    /**
     * Get command name
     */
    public function getName(): string;
}
```

---

## Base Command Class

### Abstract Implementation

```php
<?php
/**
 * Abstract base command
 */
abstract class AbstractCommand implements CommandInterface
{
    protected ?CommandHistory $history = null;
    protected array $data = [];
    protected ?\DateTime $executedAt = null;

    public function setHistory(CommandHistory $history): void
    {
        $this->history = $history;
    }

    public function execute(): mixed
    {
        $this->validate();
        $this->beforeExecute();
        $result = $this->doExecute();
        $this->executedAt = new \DateTime();
        $this->afterExecute($result);
        return $result;
    }

    public function undo(): void
    {
        $this->beforeUndo();
        $this->doUndo();
        $this->afterUndo();
    }

    public function getName(): string
    {
        return static::class;
    }

    public function getExecutedAt(): ?\DateTime
    {
        return $this->executedAt;
    }

    public function getData(): array
    {
        return $this->data;
    }

    /**
     * Template method implementations
     */
    protected function validate(): void {} // Override for validation
    protected function beforeExecute(): void {} // Override for pre-execution
    protected function afterExecute(mixed $result): void {} // Override for post-execution
    protected function beforeUndo(): void {} // Override for pre-undo
    protected function afterUndo(): void {} // Override for post-undo

    abstract protected function doExecute(): mixed;
    abstract protected function doUndo(): void;
}
```

---

## Client Commands

### CreateClientCommand

```php
<?php
/**
 * Create client command
 */
class CreateClientCommand extends AbstractCommand
{
    private array $clientData;
    private ?int $createdClientId = null;

    public function __construct(array $clientData)
    {
        $this->clientData = $clientData;
        $this->data = $clientData;
    }

    public function getName(): string
    {
        return 'create_client';
    }

    protected function validate(): void
    {
        $required = ['firstname', 'lastname', 'email'];

        foreach ($required as $field) {
            if (empty($this->clientData[$field])) {
                throw new CommandValidationException("Missing required field: $field");
            }
        }

        if (!filter_var($this->clientData['email'], FILTER_VALIDATE_EMAIL)) {
            throw new CommandValidationException('Invalid email address');
        }

        // Check for duplicate email
        $exists = Capsule::table('tblclients')
            ->where('email', $this->clientData['email'])
            ->exists();

        if ($exists) {
            throw new CommandValidationException('Email already exists');
        }
    }

    protected function doExecute(): mixed
    {
        $this->createdClientId = Capsule::table('tblclients')->insertGetId([
            'firstname' => $this->clientData['firstname'],
            'lastname' => $this->clientData['lastname'],
            'email' => $this->clientData['email'],
            'companyname' => $this->clientData['companyname'] ?? '',
            'phonenumber' => $this->clientData['phonenumber'] ?? '',
            'country' => $this->clientData['country'] ?? 'US',
            'datecreated' => date('Y-m-d H:i:s'),
            'password' => password_hash($this->clientData['password'] ?? '', PASSWORD_DEFAULT),
        ]);

        logActivity('Client created via Command: ' . $this->createdClientId);

        return $this->createdClientId;
    }

    protected function doUndo(): void
    {
        if ($this->createdClientId) {
            Capsule::table('tblclients')
                ->where('id', $this->createdClientId)
                ->delete();

            logActivity('Client ' . $this->createdClientId . ' undone via Command');
        }
    }

    public function getCreatedClientId(): ?int
    {
        return $this->createdClientId;
    }
}
```

### UpdateClientCommand

```php
<?php
/**
 * Update client command with undo support
 */
class UpdateClientCommand extends AbstractCommand
{
    private int $clientId;
    private array $updates;
    private ?array $previousData = null;

    public function __construct(int $clientId, array $updates)
    {
        $this->clientId = $clientId;
        $this->updates = $updates;
        $this->data = ['client_id' => $clientId, 'updates' => $updates];
    }

    public function getName(): string
    {
        return 'update_client';
    }

    protected function validate(): void
    {
        $this->previousData = Capsule::table('tblclients')
            ->where('id', $this->clientId)
            ->first();

        if (!$this->previousData) {
            throw new CommandValidationException("Client {$this->clientId} not found");
        }
    }

    protected function doExecute(): mixed
    {
        $filteredUpdates = array_intersect_key(
            $this->updates,
            array_flip(['firstname', 'lastname', 'email', 'companyname', 'phonenumber'])
        );

        Capsule::table('tblclients')
            ->where('id', $this->clientId)
            ->update($filteredUpdates);

        logActivity('Client updated via Command: ' . $this->clientId);

        return $this->clientId;
    }

    protected function doUndo(): void
    {
        if ($this->previousData) {
            Capsule::table('tblclients')
                ->where('id', $this->clientId)
                ->update((array) $this->previousData);

            logActivity('Client ' . $this->clientId . ' restored via Command undo');
        }
    }
}
```

---

## Order Commands

### CreateOrderCommand

```php
<?php
/**
 * Create order command
 */
class CreateOrderCommand extends AbstractCommand
{
    private int $clientId;
    private array $products;
    private array $paymentMethod;
    private ?int $orderId = null;

    public function __construct(int $clientId, array $products, array $paymentMethod = [])
    {
        $this->clientId = $clientId;
        $this->products = $products;
        $this->paymentMethod = $paymentMethod;
        $this->data = ['client_id' => $clientId, 'products' => $products];
    }

    public function getName(): string
    {
        return 'create_order';
    }

    protected function validate(): void
    {
        if (empty($this->products)) {
            throw new CommandValidationException('No products specified');
        }

        foreach ($this->products as $product) {
            if (empty($product['product_id']) || empty($product['billingcycle'])) {
                throw new CommandValidationException('Invalid product data');
            }
        }
    }

    protected function doExecute(): mixed
    {
        $total = 0;
        foreach ($this->products as $product) {
            $total += ($product['price'] ?? 0) * ($product['quantity'] ?? 1);
        }

        $this->orderId = Capsule::table('tblorders')->insertGetId([
            'userid' => $this->clientId,
            'ordernum' => generateOrderNumber(),
            'date' => date('Y-m-d H:i:s'),
            'status' => 'Pending',
            'paymentmethod' => $this->paymentMethod['method'] ?? 'paypal',
            'ipaddress' => $_SERVER['REMOTE_ADDR'] ?? '',
            'total' => $total,
        ]);

        foreach ($this->products as $product) {
            Capsule::table('tblorderitems')->insert([
                'orderid' => $this->orderId,
                'userid' => $this->clientId,
                'type' => $product['type'] ?? 'hosting',
                'relid' => $product['product_id'],
                'qty' => $product['quantity'] ?? 1,
                'billingcycle' => $product['billingcycle'] ?? 'monthly',
                'amount' => $product['price'] ?? 0,
                'description' => $product['description'] ?? '',
            ]);
        }

        logActivity('Order created via Command: ' . $this->orderId);

        return $this->orderId;
    }

    protected function doUndo(): void
    {
        if ($this->orderId) {
            Capsule::table('tblorders')
                ->where('id', $this->orderId)
                ->update(['status' => 'Cancelled']);

            Capsule::table('tblorderitems')
                ->where('orderid', $this->orderId)
                ->delete();

            logActivity('Order ' . $this->orderId . ' undone via Command');
        }
    }
}
```

---

## Service Commands

### ProvisionServiceCommand

```php
<?php
/**
 * Provision service command
 */
class ProvisionServiceCommand extends AbstractCommand
{
    private int $orderId;
    private ?int $serviceId = null;
    private array $result = [];

    public function __construct(int $orderId)
    {
        $this->orderId = $orderId;
        $this->data = ['order_id' => $orderId];
    }

    public function getName(): string
    {
        return 'provision_service';
    }

    protected function beforeExecute(): void
    {
        // Check prerequisites
        $order = Capsule::table('tblorders')
            ->where('id', $this->orderId)
            ->first();

        if (!$order) {
            throw new CommandValidationException('Order not found');
        }

        if ($order->status !== 'Pending') {
            throw new CommandValidationException('Order must be pending');
        }
    }

    protected function doExecute(): mixed
    {
        $order = Capsule::table('tblorders')->where('id', $this->orderId)->first();

        // Get order items
        $items = Capsule::table('tblorderitems')
            ->where('orderid', $this->orderId)
            ->get();

        foreach ($items as $item) {
            if ($item->type === 'hosting') {
                $result = $this->provisionHosting($item, $order);
                if (isset($result['service_id'])) {
                    $this->serviceId = $result['service_id'];
                }
            }
        }

        // Update order status
        Capsule::table('tblorders')
            ->where('id', $this->orderId)
            ->update(['status' => 'Active']);

        return [
            'order_id' => $this->orderId,
            'service_id' => $this->serviceId,
            'result' => $this->result,
        ];
    }

    private function provisionHosting($item, $order): array
    {
        $product = Capsule::table('tblproducts')
            ->where('id', $item->relid)
            ->first();

        $server = Capsule::table('tblservers')
            ->where('type', $product->servertype)
            ->first();

        $params = [
            'server' => $server,
            'service' => ['id' => $item->relid],
            'client' => ['id' => $order->userid],
            'username' => 'user' . $item->id,
            'password' => generatePassword(),
            'domain' => 'service-' . $item->id . '.example.com',
        ];

        $module = \WHMCS\Module\Server::factory($product->servertype);
        $result = $module->CreateAccount($params);

        if ($result === 'success') {
            $serviceId = Capsule::table('tblhosting')->insertGetId([
                'userid' => $order->userid,
                'orderid' => $this->orderId,
                'packageid' => $item->relid,
                'serverid' => $server->id,
                'domain' => $params['domain'],
                'username' => $params['username'],
                'password' => encrypt($params['password']),
                'regdate' => date('Y-m-d H:i:s'),
                'nextduedate' => date('Y-m-d H:i:s', strtotime('+1 month')),
                'domainstatus' => 'Active',
            ]);

            return ['service_id' => $serviceId, 'status' => 'success'];
        }

        return ['error' => $result, 'status' => 'failed'];
    }

    protected function doUndo(): void
    {
        if ($this->serviceId) {
            $service = Capsule::table('tblhosting')
                ->where('id', $this->serviceId)
                ->first();

            if ($service) {
                $product = Capsule::table('tblproducts')
                    ->where('id', $service->packageid)
                    ->first();

                $server = Capsule::table('tblservers')
                    ->where('id', $service->serverid)
                    ->first();

                $module = \WHMCS\Module\Server::factory($product->servertype);
                $module->TerminateAccount([
                    'server' => $server,
                    'service' => $service,
                ]);

                Capsule::table('tblhosting')
                    ->where('id', $this->serviceId)
                    ->delete();
            }

            Capsule::table('tblorders')
                ->where('id', $this->orderId)
                ->update(['status' => 'Pending']);
        }
    }
}
```

---

## Command History

### History Manager

```php
<?php
/**
 * Command history for undo/redo support
 */
class CommandHistory
{
    private array $executed = [];
    private array $undone = [];
    private int $maxHistory = 100;

    /**
     * Execute command and add to history
     */
    public function execute(CommandInterface $command): mixed
    {
        $command->setHistory($this);

        try {
            $result = $command->execute();

            $this->executed[] = [
                'command' => $command,
                'timestamp' => date('Y-m-d H:i:s'),
                'result' => $result,
            ];

            $this->trimHistory();

            return $result;

        } catch (\Exception $ex) {
            logActivity('Command execution failed: ' . $ex->getMessage());
            throw $ex;
        }
    }

    /**
     * Undo last command
     */
    public function undo(): bool
    {
        if (empty($this->executed)) {
            return false;
        }

        $entry = array_pop($this->executed);

        try {
            $entry['command']->undo();

            $this->undone[] = [
                'command' => $entry['command'],
                'timestamp' => date('Y-m-d H:i:s'),
                'original_result' => $entry['result'],
            ];

            return true;

        } catch (\Exception $ex) {
            logActivity('Command undo failed: ' . $ex->getMessage());
            $this->executed[] = $entry;
            throw $ex;
        }
    }

    /**
     * Redo last undone command
     */
    public function redo(): bool
    {
        if (empty($this->undone)) {
            return false;
        }

        $entry = array_pop($this->undone);

        try {
            $entry['command']->execute();

            $this->executed[] = [
                'command' => $entry['command'],
                'timestamp' => date('Y-m-d H:i:s'),
                'result' => 'redo',
            ];

            return true;

        } catch (\Exception $ex) {
            logActivity('Command redo failed: ' . $ex->getMessage());
            throw $ex;
        }
    }

    public function canUndo(): bool
    {
        return !empty($this->executed);
    }

    public function canRedo(): bool
    {
        return !empty($this->undone);
    }

    public function getHistory(): array
    {
        return $this->executed;
    }

    public function clear(): void
    {
        $this->executed = [];
        $this->undone = [];
    }

    private function trimHistory(): void
    {
        while (count($this->executed) > $this->maxHistory) {
            array_shift($this->executed);
        }
    }
}
```

---

## Command Queue

### Async Queue Processing

```php
<?php
/**
 * Queue commands for async execution
 */
class CommandQueue
{
    private string $queueTable = 'mod_command_queue';

    /**
     * Queue a command for execution
     */
    public function enqueue(CommandInterface $command, int $delay = 0, int $priority = 5): string
    {
        $jobId = uniqid('cmd_');

        Capsule::table($this->queueTable)->insert([
            'job_id' => $jobId,
            'command_class' => get_class($command),
            'command_data' => json_encode($this->serializeCommand($command)),
            'status' => 'pending',
            'priority' => $priority,
            'delay_until' => $delay > 0 ? date('Y-m-d H:i:s', time() + $delay) : null,
            'created_at' => date('Y-m-d H:i:s'),
        ]);

        return $jobId;
    }

    /**
     * Process queue
     */
    public function process(int $limit = 50): array
    {
        $processed = [];

        $jobs = Capsule::table($this->queueTable)
            ->where('status', 'pending')
            ->whereRaw("(delay_until IS NULL OR delay_until <= NOW())")
            ->orderBy('priority', 'desc')
            ->orderBy('created_at', 'asc')
            ->limit($limit)
            ->get();

        foreach ($jobs as $job) {
            try {
                $command = $this->deserializeCommand($job);

                $result = $command->execute();

                Capsule::table($this->queueTable)
                    ->where('id', $job->id)
                    ->update([
                        'status' => 'completed',
                        'result' => json_encode($result),
                        'completed_at' => date('Y-m-d H:i:s'),
                    ]);

                $processed[] = [
                    'job_id' => $job->job_id,
                    'status' => 'success',
                    'result' => $result,
                ];

            } catch (\Exception $e) {
                Capsule::table($this->queueTable)
                    ->where('id', $job->id)
                    ->update([
                        'status' => 'failed',
                        'error' => $e->getMessage(),
                        'attempts' => $job->attempts + 1,
                    ]);

                $processed[] = [
                    'job_id' => $job->job_id,
                    'status' => 'failed',
                    'error' => $e->getMessage(),
                ];
            }
        }

        return $processed;
    }

    private function serializeCommand(CommandInterface $command): array
    {
        if ($command instanceof SerializableCommand) {
            return $command->serialize();
        }

        return [
            'class' => get_class($command),
            'data' => $command->getData(),
        ];
    }

    private function deserializeCommand($job): CommandInterface
    {
        $data = json_decode($job->command_data, true);
        $class = $data['class'] ?? $job->command_class;

        if (class_exists($class)) {
            $method = (new \ReflectionClass($class))->getMethod('deserialize');

            if ($method->isStatic()) {
                return $class::deserialize($data['data']);
            }
        }

        // Fallback to direct instantiation
        $command = new $class($data['data'] ?? []);
        return $command;
    }
}
```

### Serializable Command

```php
<?php
/**
 * Serializable command interface
 */
interface SerializableCommand extends CommandInterface
{
    public function serialize(): array;
    public static function deserialize(array $data): self;
}

/**
 * Serializable create client command
 */
class CreateClientCommand extends AbstractCommand implements SerializableCommand
{
    public function serialize(): array
    {
        return [
            'client_data' => $this->clientData,
        ];
    }

    public static function deserialize(array $data): self
    {
        return new self($data['client_data']);
    }
}
```

---

## Command Factory

### Factory Pattern Implementation

```php
<?php
/**
 * Command factory for creating commands
 */
class CommandFactory
{
    private array $registry = [];

    static private ?CommandFactory $instance = null;

    static public function getInstance(): self
    {
        if (self::$instance === null) {
            self::$instance = new self();
        }
        return self::$instance;
    }

    public function register(string $name, string $commandClass): void
    {
        $this->registry[$name] = $commandClass;
    }

    public function make(string $name, array $parameters = []): CommandInterface
    {
        if (!isset($this->registry[$name])) {
            throw new CommandNotFoundException("Command not found: $name");
        }

        $class = $this->registry[$name];
        return new $class(...$parameters);
    }

    public function execute(string $name, array $parameters = []): mixed
    {
        $command = $this->make($name, $parameters);
        return $command->execute();
    }

    public function listCommands(): array
    {
        return array_keys($this->registry);
    }
}

// Register defaults
$factory = CommandFactory::getInstance();

$factory->register('create_client', CreateClientCommand::class);
$factory->register('update_client', UpdateClientCommand::class);
$factory->register('create_order', CreateOrderCommand::class);
$factory->register('provision_service', ProvisionServiceCommand::class);
```

---

## Cron Processing

### Daily Command Processing

```php
<?php
add_hook('DailyCronJob', 1, function() {
    $queue = new CommandQueue();

    $processed = $queue->process(100);

    $success = array_filter($processed, fn($p) => $p['status'] === 'success');
    $failed = array_filter($processed, fn($p) => $p['status'] === 'failed');

    if (count($failed) > 0) {
        logActivity('Command queue: ' . count($failed) . ' jobs failed');
    }

    logActivity('Command queue processed: ' . count($success) . ' succeeded, '
              . count($failed) . ' failed');
});
```

---

## Best Practices

1. **Single responsibility** - Each command should do one thing
2. **Immutable data** - Store original data for undo support
3. **Validation** - Validate before execution
4. **Logging** - Log all executions and undo operations
5. **Serialization** - Implement SerializableCommand for queue support
6. **Exception handling** - Catch and log all exceptions
7. **Transaction safety** - Use database transactions where needed
8. **Idempotency** - Design commands to be safely re-runnable

---

## Exception Classes

```php
<?php
class CommandValidationException extends \Exception {}
class CommandExecutionException extends \Exception {}
class CommandNotFoundException extends \Exception {}
class CommandUndoException extends \Exception {}
```

---

## Related Documentation

- [Command Pattern Guide](command-pattern.md)
- [Queue Processing](queue-processing.md)
- [Service Layer](service-layer.md)
