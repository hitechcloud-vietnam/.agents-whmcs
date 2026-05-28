# Command Pattern Implementation

The Command Pattern encapsulates requests as objects, enabling parameterization, queuing, and undo functionality.

## Command Interface

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

### Base Command Class

```php
<?php
/**
 * Abstract base command
 */
abstract class AbstractCommand implements CommandInterface
{
    protected ?CommandHistory $history = null;

    public function setHistory(CommandHistory $history): void
    {
        $this->history = $history;
    }

    public function execute(): mixed
    {
        $this->beforeExecute();
        $result = $this->doExecute();
        $this->afterExecute();
        return $result;
    }

    public function undo(): void
    {
        $this->beforeUndo();
        $this->doUndo();
        $this->afterUndo();
    }

    /**
     * Template method implementations
     */
    protected function beforeExecute(): void {}
    protected function afterExecute(): void {}
    protected function beforeUndo(): void {}
    protected function afterUndo(): void {}

    abstract protected function doExecute(): mixed;
    abstract protected function doUndo(): void;
}
```

## Command Implementations

### Client Management Commands

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
    }

    public function getName(): string
    {
        return 'create_client';
    }

    protected function doExecute(): mixed
    {
        // Validate data
        $this->validateClientData();

        // Create client
        $this->createdClientId = Capsule::table('tblclients')->insertGetId([
            'firstname' => $this->clientData['firstname'],
            'lastname' => $this->clientData['lastname'],
            'email' => $this->clientData['email'],
            'companyname' => $this->clientData['companyname'] ?? '',
            'phonenumber' => $this->clientData['phonenumber'] ?? '',
            'datecreated' => date('Y-m-d H:i:s'),
            'registration_progress' => 'complete',
        ]);

        return $this->createdClientId;
    }

    protected function doUndo(): void
    {
        if ($this->createdClientId) {
            Capsule::table('tblclients')
                ->where('id', $this->createdClientId)
                ->delete();
        }
    }

    public function getCreatedClientId(): ?int
    {
        return $this->createdClientId;
    }

    private function validateClientData(): void
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
    }
}

/**
 * Update client command
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
    }

    public function getName(): string
    {
        return 'update_client';
    }

    protected function doExecute(): mixed
    {
        // Store previous state for undo
        $this->previousData = Capsule::table('tblclients')
            ->where('id', $this->clientId)
            ->first();

        if (!$this->previousData) {
            throw new CommandExecutionException("Client {$this->clientId} not found");
        }

        // Apply updates
        Capsule::table('tblclients')
            ->where('id', $this->clientId)
            ->update($this->updates);

        return $this->clientId;
    }

    protected function doUndo(): void
    {
        if ($this->previousData) {
            Capsule::table('tblclients')
                ->where('id', $this->clientId)
                ->update((array) $this->previousData);
        }
    }
}

/**
 * Delete client command
 */
class DeleteClientCommand extends AbstractCommand
{
    private int $clientId;
    private ?array $clientData = null;
    private array $relatedData = [];

    public function __construct(int $clientId)
    {
        $this->clientId = $clientId;
    }

    public function getName(): string
    {
        return 'delete_client';
    }

    protected function doExecute(): mixed
    {
        // Store data for potential undo
        $this->clientData = Capsule::table('tblclients')
            ->where('id', $this->clientId)
            ->first();

        if (!$this->clientData) {
            throw new CommandExecutionException("Client {$this->clientId} not found");
        }

        // Store related data
        $this->relatedData = [
            'orders' => Capsule::table('tblorders')
                ->where('userid', $this->clientId)
                ->get(),
            'invoices' => Capsule::table('tblinvoices')
                ->where('userid', $this->clientId)
                ->get(),
        ];

        // Soft delete or archive based on requirements
        Capsule::table('tblclients')
            ->where('id', $this->clientId)
            ->update(['status' => 'inactive']);

        return true;
    }

    protected function doUndo(): void
    {
        if ($this->clientData) {
            Capsule::table('tblclients')
                ->where('id', $this->clientId)
                ->update(['status' => 'active']);
        }
    }
}
```

### Order Commands

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
    }

    public function getName(): string
    {
        return 'create_order';
    }

    protected function doExecute(): mixed
    {
        $total = 0;

        foreach ($this->products as $product) {
            $total += $product['price'] * ($product['quantity'] ?? 1);
        }

        $this->orderId = Capsule::table('tblorders')->insertGetId([
            'userid' => $this->clientId,
            'ordernum' => generateOrderNumber(),
            'date' => date('Y-m-d H:i:s'),
            'status' => 'Pending',
            'paymentmethod' => $this->paymentMethod['method'] ?? 'paypal',
            'ipaddress' => $_SERVER['REMOTE_ADDR'] ?? '',
        ]);

        // Create order items
        foreach ($this->products as $product) {
            Capsule::table('tblorderitems')->insert([
                'orderid' => $this->orderId,
                'type' => $product['type'] ?? 'hosting',
                'relid' => $product['relid'] ?? 0,
                'qty' => $product['quantity'] ?? 1,
                'price' => $product['price'],
                'description' => $product['description'] ?? '',
            ]);
        }

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
        }
    }
}
```

## Command History

### Command History Manager

```php
<?php
/**
 * Command history for undo/redo
 */
class CommandHistory
{
    private array $executed = [];
    private array $undone = [];
    private int $maxHistory = 100;

    /**
     * Execute a command and add to history
     */
    public function execute(CommandInterface $command): mixed
    {
        $command->setHistory($this);

        $result = $command->execute();

        $this->executed[] = [
            'command' => $command,
            'timestamp' => date('Y-m-d H:i:s'),
            'result' => $result,
        ];

        // Clear redo stack
        $this->undone = [];

        // Trim history if needed
        while (count($this->executed) > $this->maxHistory) {
            array_shift($this->executed);
        }

        return $result;
    }

    /**
     * Undo the last command
     */
    public function undo(): bool
    {
        if (empty($this->executed)) {
            return false;
        }

        $entry = array_pop($this->executed);
        $entry['command']->undo();

        $this->undone[] = $entry;

        return true;
    }

    /**
     * Redo the last undone command
     */
    public function redo(): bool
    {
        if (empty($this->undone)) {
            return false;
        }

        $entry = array_pop($this->undone);
        $entry['command']->execute();

        $this->executed[] = $entry;

        return true;
    }

    /**
     * Check if undo is available
     */
    public function canUndo(): bool
    {
        return !empty($this->executed);
    }

    /**
     * Check if redo is available
     */
    public function canRedo(): bool
    {
        return !empty($this->undone);
    }

    /**
     * Get execution history
     */
    public function getHistory(): array
    {
        return $this->executed;
    }

    /**
     * Clear history
     */
    public function clear(): void
    {
        $this->executed = [];
        $this->undone = [];
    }
}
```

## Command Queue

### Async Command Queue

```php
<?php
/**
 * Command queue for async execution
 */
class CommandQueue
{
    private DatabaseQueue $queue;

    public function __construct(DatabaseQueue $queue)
    {
        $this->queue = $queue;
    }

    /**
     * Queue a command for execution
     */
    public function enqueue(CommandInterface $command, int $delay = 0): string
    {
        $job = new ExecuteCommandJob($command);
        return $this->queue->later($job, $delay);
    }

    /**
     * Execute commands from queue
     */
    public function process(): void
    {
        while ($command = $this->queue->pop()) {
            $command->execute();
        }
    }
}

/**
 * Job to execute a command
 */
class ExecuteCommandJob extends AbstractJob
{
    private string $commandClass;
    private array $commandData;

    public function __construct(CommandInterface $command)
    {
        $this->commandClass = get_class($command);
        $this->commandData = $this->serializeCommand($command);
    }

    public function handle(): void
    {
        $command = $this->deserializeCommand($this->commandData);
        $command->execute();
    }

    private function serializeCommand(CommandInterface $command): array
    {
        if ($command instanceof SerializableCommand) {
            return $command->serialize();
        }

        return [
            'class' => $this->commandClass,
            'data' => [],
        ];
    }

    private function deserializeCommand(array $data): CommandInterface
    {
        $class = $data['class'];
        return $class::deserialize($data['data']);
    }
}
```

## Command Factory

### Command Factory Pattern

```php
<?php
/**
 * Command factory
 */
class CommandFactory
{
    private array $registry = [];

    public function __construct()
    {
        $this->registerDefaults();
    }

    /**
     * Register a command type
     */
    public function register(string $name, string $commandClass): void
    {
        $this->registry[$name] = $commandClass;
    }

    /**
     * Create a command by name
     */
    public function make(string $name, array $parameters = []): CommandInterface
    {
        if (!isset($this->registry[$name])) {
            throw new CommandNotFoundException("Command not found: $name");
        }

        $class = $this->registry[$name];

        return new $class(...$parameters);
    }

    /**
     * Execute a command by name
     */
    public function execute(string $name, array $parameters = []): mixed
    {
        $command = $this->make($name, $parameters);
        return $command->execute();
    }

    private function registerDefaults(): void
    {
        $this->register('create_client', CreateClientCommand::class);
        $this->register('update_client', UpdateClientCommand::class);
        $this->register('delete_client', DeleteClientCommand::class);
        $this->register('create_order', CreateOrderCommand::class);
    }
}
```

## Serialization for Storage

### Serializable Commands

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
 * Create client command with serialization
 */
class CreateClientCommand extends AbstractCommand implements SerializableCommand
{
    private array $clientData;
    private ?int $createdClientId = null;

    public function __construct(array $clientData)
    {
        $this->clientData = $clientData;
    }

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

    // ... rest of implementation
}
```

## Best Practices

1. **Keep commands focused** - Single responsibility per command
2. **Make commands undoable** - Support rollback operations
3. **Use command history** - Enable undo/redo functionality
4. **Queue for async** - Process commands in background
5. **Implement validation** - Validate before execution
6. **Log executions** - Maintain audit trail
7. **Support serialization** - Enable persistent queues

## Related Patterns

- [Queue Processing](./queue-processing.md) - Async command handling
- [Service Layer](./service-layer.md) - Service commands
- [Event Sourcing](./event-sourcing.md) - Event-based commands