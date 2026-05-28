# CQRS Pattern Basics

Command Query Responsibility Segregation (CQRS) separates read and write operations into different models.

## CQRS Architecture

### Command Interface

```php
<?php
/**
 * Command interface
 */
interface Command
{
    /**
     * Get command name
     */
    public function getName(): string;

    /**
     * Validate command
     */
    public function validate(): void;
}

/**
 * Base command
 */
abstract class AbstractCommand implements Command
{
    protected array $errors = [];

    public function validate(): void
    {
        // Override in subclasses
    }

    public function hasErrors(): bool
    {
        return !empty($this->errors);
    }

    public function getErrors(): array
    {
        return $this->errors;
    }

    protected function addError(string $field, string $message): void
    {
        $this->errors[$field][] = $message;
    }
}
```

### Query Interface

```php
<?php
/**
 * Query interface
 */
interface Query
{
    /**
     * Get query name
     */
    public function getName(): string;
}

/**
 * Base query
 */
abstract class AbstractQuery implements Query
{
    protected array $criteria = [];

    public function __construct(array $criteria = [])
    {
        $this->criteria = $criteria;
    }

    public function getCriteria(): array
    {
        return $this->criteria;
    }

    public function get(string $key, $default = null)
    {
        return $this->criteria[$key] ?? $default;
    }
}
```

## Command Implementations

### Create Client Command

```php
<?php
/**
 * Create client command
 */
class CreateClientCommand extends AbstractCommand
{
    private string $firstname;
    private string $lastname;
    private string $email;
    private ?string $company;
    private ?string $phone;

    public function __construct(array $data)
    {
        $this->firstname = $data['firstname'] ?? '';
        $this->lastname = $data['lastname'] ?? '';
        $this->email = $data['email'] ?? '';
        $this->company = $data['companyname'] ?? null;
        $this->phone = $data['phonenumber'] ?? null;
    }

    public function getName(): string
    {
        return 'CreateClient';
    }

    public function validate(): void
    {
        if (empty($this->firstname)) {
            $this->addError('firstname', 'First name is required');
        }

        if (empty($this->lastname)) {
            $this->addError('lastname', 'Last name is required');
        }

        if (empty($this->email)) {
            $this->addError('email', 'Email is required');
        } elseif (!filter_var($this->email, FILTER_VALIDATE_EMAIL)) {
            $this->addError('email', 'Invalid email format');
        }
    }

    public function execute(): Client
    {
        $client = Client::create([
            'firstname' => $this->firstname,
            'lastname' => $this->lastname,
            'email' => $this->email,
            'companyname' => $this->company,
            'phonenumber' => $this->phone,
        ]);

        $this->save($client);

        // Dispatch event
        $this->dispatch(new ClientCreatedEvent($client->getId()));

        return $client;
    }

    private function save(Client $client): void
    {
        Capsule::table('tblclients')->insert($client->toArray());
    }

    private function dispatch(Event $event): void
    {
        // Dispatch through event bus
    }
}
```

### Update Order Command

```php
<?php
/**
 * Update order command
 */
class UpdateOrderCommand extends AbstractCommand
{
    private int $orderId;
    private array $updates;

    public function __construct(int $orderId, array $updates)
    {
        $this->orderId = $orderId;
        $this->updates = $updates;
    }

    public function getName(): string
    {
        return 'UpdateOrder';
    }

    public function validate(): void
    {
        if ($this->orderId <= 0) {
            $this->addError('order_id', 'Invalid order ID');
        }

        $allowedFields = ['status', 'notes', 'payment_method'];
        foreach (array_keys($this->updates) as $field) {
            if (!in_array($field, $allowedFields)) {
                $this->addError($field, "Field '$field' cannot be updated");
            }
        }
    }

    public function execute(): Order
    {
        Capsule::table('tblorders')
            ->where('id', $this->orderId)
            ->update($this->updates);

        return $this->findOrder($this->orderId);
    }

    private function findOrder(int $id): Order
    {
        $data = Capsule::table('tblorders')->find($id);
        return Order::fromArray((array) $data);
    }
}
```

## Query Implementations

### Get Client Query

```php
<?php
/**
 * Get client query
 */
class GetClientQuery extends AbstractQuery
{
    private ?int $clientId;
    private ?string $email;

    public function __construct(array $criteria)
    {
        $this->clientId = $criteria['client_id'] ?? null;
        $this->email = $criteria['email'] ?? null;
    }

    public function getName(): string
    {
        return 'GetClient';
    }

    public function execute(): ?ClientDTO
    {
        if ($this->clientId) {
            return $this->findById($this->clientId);
        }

        if ($this->email) {
            return $this->findByEmail($this->email);
        }

        return null;
    }

    private function findById(int $id): ?ClientDTO
    {
        $data = Capsule::table('tblclients')
            ->where('id', $id)
            ->first();

        if (!$data) {
            return null;
        }

        return ClientDTO::fromArray((array) $data);
    }

    private function findByEmail(string $email): ?ClientDTO
    {
        $data = Capsule::table('tblclients')
            ->where('email', $email)
            ->first();

        if (!$data) {
            return null;
        }

        return ClientDTO::fromArray((array) $data);
    }
}
```

### Search Clients Query

```php
<?php
/**
 * Search clients query
 */
class SearchClientsQuery extends AbstractQuery
{
    private string $term;
    private int $limit;
    private int $offset;

    public function __construct(array $criteria)
    {
        $this->term = $criteria['term'] ?? '';
        $this->limit = min($criteria['limit'] ?? 20, 100);
        $this->offset = max($criteria['offset'] ?? 0, 0);
    }

    public function getName(): string
    {
        return 'SearchClients';
    }

    public function execute(): PaginatedResult
    {
        $query = Capsule::table('tblclients');

        if (!empty($this->term)) {
            $term = '%' . $this->term . '%';
            $query->where(function ($q) use ($term) {
                $q->where('firstname', 'LIKE', $term)
                    ->orWhere('lastname', 'LIKE', $term)
                    ->orWhere('email', 'LIKE', $term)
                    ->orWhere('companyname', 'LIKE', $term);
            });
        }

        $total = $query->count();

        $records = $query
            ->orderBy('id', 'desc')
            ->limit($this->limit)
            ->offset($this->offset)
            ->get();

        $items = array_map(
            fn($r) => ClientDTO::fromArray((array) $r),
            $records
        );

        return new PaginatedResult($items, $total, $this->limit, $this->offset);
    }
}
```

### Get Client Orders Query

```php
<?php
/**
 * Get client orders query
 */
class GetClientOrdersQuery extends AbstractQuery
{
    private int $clientId;
    private ?string $status;
    private int $limit;

    public function __construct(array $criteria)
    {
        $this->clientId = $criteria['client_id'];
        $this->status = $criteria['status'] ?? null;
        $this->limit = min($criteria['limit'] ?? 50, 100);
    }

    public function getName(): string
    {
        return 'GetClientOrders';
    }

    public function execute(): array
    {
        $query = Capsule::table('tblorders')
            ->where('userid', $this->clientId);

        if ($this->status) {
            $query->where('status', $this->status);
        }

        $records = $query
            ->orderBy('date', 'desc')
            ->limit($this->limit)
            ->get();

        return array_map(
            fn($r) => OrderDTO::fromArray((array) $r),
            $records
        );
    }
}
```

## Command Handler

```php
<?php
/**
 * Command handler
 */
class CommandHandler
{
    private array $handlers = [];

    public function register(string $commandClass, callable $handler): void
    {
        $this->handlers[$commandClass] = $handler;
    }

    public function handle(Command $command): mixed
    {
        $commandClass = get_class($command);

        if (!isset($this->handlers[$commandClass])) {
            throw new HandlerNotFoundException("No handler for $commandClass");
        }

        return call_user_func($this->handlers[$commandClass], $command);
    }
}

/**
 * Query handler
 */
class QueryHandler
{
    private array $handlers = [];

    public function register(string $queryClass, callable $handler): void
    {
        $this->handlers[$queryClass] = $handler;
    }

    public function handle(Query $query): mixed
    {
        $queryClass = get_class($query);

        if (!isset($this->handlers[$queryClass])) {
            throw new HandlerNotFoundException("No handler for $queryClass");
        }

        return call_user_func($this->handlers[$queryClass], $query);
    }
}
```

## CQRS Bus

### Command/Query Bus

```php
<?php
/**
 * CQRS bus for commands and queries
 */
class CQRSBus
{
    private CommandHandler $commandHandler;
    private QueryHandler $queryHandler;

    public function __construct(
        CommandHandler $commandHandler,
        QueryHandler $queryHandler
    ) {
        $this->commandHandler = $commandHandler;
        $this->queryHandler = $queryHandler;
    }

    /**
     * Dispatch a command
     */
    public function dispatch(Command $command): mixed
    {
        $command->validate();

        if ($command->hasErrors()) {
            throw new ValidationException($command->getErrors());
        }

        return $this->commandHandler->handle($command);
    }

    /**
     * Execute a query
     */
    public function query(Query $query): mixed
    {
        return $this->queryHandler->handle($query);
    }
}
```

### Registration

```php
<?php
/**
 * Register command and query handlers
 */
class CQRSSetup
{
    private CommandHandler $commandHandler;
    private QueryHandler $queryHandler;

    public function __construct()
    {
        $this->commandHandler = new CommandHandler();
        $this->queryHandler = new QueryHandler();
    }

    public function registerCommands(): void
    {
        $this->commandHandler->register(
            CreateClientCommand::class,
            fn($cmd) => $cmd->execute()
        );

        $this->commandHandler->register(
            UpdateClientCommand::class,
            fn($cmd) => $cmd->execute()
        );

        $this->commandHandler->register(
            CreateOrderCommand::class,
            fn($cmd) => $cmd->execute()
        );

        $this->commandHandler->register(
            UpdateOrderCommand::class,
            fn($cmd) => $cmd->execute()
        );
    }

    public function registerQueries(): void
    {
        $this->queryHandler->register(
            GetClientQuery::class,
            fn($q) => $q->execute()
        );

        $this->queryHandler->register(
            SearchClientsQuery::class,
            fn($q) => $q->execute()
        );

        $this->queryHandler->register(
            GetClientOrdersQuery::class,
            fn($q) => $q->execute()
        );
    }

    public function getBus(): CQRSBus
    {
        $this->registerCommands();
        $this->registerQueries();

        return new CQRSBus($this->commandHandler, $this->queryHandler);
    }
}
```

## Read Model (Projection)

### Projector

```php
<?php
/**
 * Read model projector
 */
class ClientProjector
{
    public function projectClientCreated(ClientCreatedEvent $event): void
    {
        $clientData = $event->getPayload();

        Capsule::table('mod_client_read_model')->insert([
            'client_id' => $clientData['client_id'],
            'full_name' => $clientData['firstname'] . ' ' . $clientData['lastname'],
            'email' => $clientData['email'],
            'created_at' => date('Y-m-d H:i:s'),
        ]);
    }

    public function projectClientUpdated(ClientUpdatedEvent $event): void
    {
        $payload = $event->getPayload();
        $changes = $payload['changes'];

        $update = [];
        if (isset($changes['firstname']) || isset($changes['lastname'])) {
            $update['full_name'] = ($changes['firstname'] ?? '') . ' ' . ($changes['lastname'] ?? '');
        }

        if (!empty($update)) {
            Capsule::table('mod_client_read_model')
                ->where('client_id', $payload['client_id'])
                ->update($update);
        }
    }
}
```

## DTOs for Read Side

### Client DTO

```php
<?php
/**
 * Client DTO for read operations
 */
class ClientDTO
{
    public int $id;
    public string $firstname;
    public string $lastname;
    public string $email;
    public ?string $company;
    public string $status;
    public string $createdAt;

    public static function fromArray(array $data): self
    {
        $dto = new self();
        $dto->id = (int) $data['id'];
        $dto->firstname = $data['firstname'];
        $dto->lastname = $data['lastname'];
        $dto->email = $data['email'];
        $dto->company = $data['companyname'] ?? null;
        $dto->status = $data['status'] ?? 'Active';
        $dto->createdAt = $data['datecreated'] ?? '';

        return $dto;
    }

    public function getFullName(): string
    {
        return trim($this->firstname . ' ' . $this->lastname);
    }
}
```

## Usage in Controllers

```php
<?php
/**
 * Client controller using CQRS
 */
class ClientController
{
    private CQRSBus $bus;

    public function __construct(CQRSBus $bus)
    {
        $this->bus = $bus;
    }

    public function create(array $data): ApiResponse
    {
        try {
            $command = new CreateClientCommand($data);
            $client = $this->bus->dispatch($command);

            return ApiResponse::success([
                'client_id' => $client->getId(),
                'message' => 'Client created successfully',
            ]);
        } catch (ValidationException $e) {
            return ApiResponse::error($e->getErrors());
        }
    }

    public function show(int $clientId): ApiResponse
    {
        $query = new GetClientQuery(['client_id' => $clientId]);
        $client = $this->bus->query($query);

        if (!$client) {
            return ApiResponse::error('Client not found', 404);
        }

        return ApiResponse::success($client->toArray());
    }

    public function search(array $params): ApiResponse
    {
        $query = new SearchClientsQuery($params);
        $result = $this->bus->query($query);

        return ApiResponse::success($result->toArray());
    }
}
```

## Best Practices

1. **Separate models** - Don't share domain and read models
2. **Validate commands** - Validate before execution
3. **Return DTOs from queries** - Never expose domain entities in read side
4. **Use event sourcing** - For complex write models
5. **Consider eventual consistency** - Read model may lag behind
6. **Optimize read model** - Denormalize for query performance
7. **Keep commands idempotent** - Safe to retry

## Related Patterns

- [Command Pattern](./command-pattern.md) - Command implementation
- [Repository Pattern](./repository-pattern.md) - Data access
- [Event Sourcing](./event-sourcing.md) - Event-driven updates
- [Service Layer](./service-layer.md) - Business operations