# Hexagonal Architecture Introduction

Hexagonal Architecture (Ports and Adapters) creates a clean separation between business logic and external concerns.

## Architecture Overview

```
                    ┌─────────────────────────────┐
                    │         Application         │
                    │          Core              │
                    │    ┌───────────────┐      │
                    │    │   Domain      │      │
                    │    │   Entities    │      │
                    │    │   Services   │      │
                    │    │   Value Obj   │      │
                    │    └───────────────┘      │
                    │    ┌───────────────┐      │
                    │    │   Use Cases   │      │
                    │    │   (Ports)     │      │
                    │    └───────────────┘      │
                    └─────────────────────────────┘
                              │
          ┌───────────────────┼───────────────────┐
          │                   │                   │
    ┌─────▼─────┐       ┌─────▼─────┐       ┌─────▼─────┐
    │  Primary  │       │ Secondary  │       │ Secondary │
    │  Adapters │       │  Adapters  │       │  Adapters │
    │  (API)    │       │   (DB)     │       │  (Email)  │
    └───────────┘       └───────────┘       └───────────┘
```

## Domain Layer

### Domain Entities

```php
<?php
/**
 * Domain entity - Client
 */
class Client
{
    private ?int $id;
    private string $firstname;
    private string $lastname;
    private Email $email;
    private ClientStatus $status;
    private DateTimeImmutable $createdAt;

    private function __construct(
        string $firstname,
        string $lastname,
        Email $email
    ) {
        $this->id = null;
        $this->firstname = $firstname;
        $this->lastname = $lastname;
        $this->email = $email;
        $this->status = ClientStatus::active();
        $this->createdAt = new DateTimeImmutable();
    }

    public static function create(
        string $firstname,
        string $lastname,
        Email $email
    ): self {
        return new self($firstname, $lastname, $email);
    }

    public function getId(): ?int
    {
        return $this->id;
    }

    public function getEmail(): Email
    {
        return $this->email;
    }

    public function getFullName(): string
    {
        return trim($this->firstname . ' ' . $this->lastname);
    }

    public function getStatus(): ClientStatus
    {
        return $this->status;
    }

    public function suspend(): void
    {
        if ($this->status->isActive()) {
            $this->status = ClientStatus::suspended();
        }
    }

    public function activate(): void
    {
        $this->status = ClientStatus::active();
    }

    public function toArray(): array
    {
        return [
            'id' => $this->id,
            'firstname' => $this->firstname,
            'lastname' => $this->lastname,
            'email' => (string) $this->email,
            'status' => $this->status->getValue(),
            'created_at' => $this->createdAt->format('Y-m-d H:i:s'),
        ];
    }
}

/**
 * Client status value object
 */
class ClientStatus
{
    private const ACTIVE = 'Active';
    private const INACTIVE = 'Inactive';
    private const SUSPENDED = 'Suspended';

    private string $value;

    private function __construct(string $value)
    {
        $this->value = $value;
    }

    public static function active(): self
    {
        return new self(self::ACTIVE);
    }

    public static function inactive(): self
    {
        return new self(self::INACTIVE);
    }

    public static function suspended(): self
    {
        return new self(self::SUSPENDED);
    }

    public function getValue(): string
    {
        return $this->value;
    }

    public function isActive(): bool
    {
        return $this->value === self::ACTIVE;
    }

    public function equals(ClientStatus $other): bool
    {
        return $this->value === $other->value;
    }
}
```

## Ports (Interfaces)

### Primary Ports (Driving)

```php
<?php
/**
 * Client service port (use case interface)
 */
interface ClientServicePort
{
    public function createClient(CreateClientInput $input): ClientOutput;
    public function getClient(int $id): ?ClientOutput;
    public function updateClient(int $id, UpdateClientInput $input): ClientOutput;
    public function searchClients(string $term): array;
}

/**
 * Order service port
 */
interface OrderServicePort
{
    public function createOrder(CreateOrderInput $input): OrderOutput;
    public function getOrder(int $id): ?OrderOutput;
    public function cancelOrder(int $id, string $reason): void;
    public function getClientOrders(int $clientId): array;
}
```

### Secondary Ports (Driven)

```php
<?php
/**
 * Client repository port
 */
interface ClientRepositoryPort
{
    public function findById(int $id): ?Client;
    public function findByEmail(Email $email): ?Client;
    public function save(Client $client): void;
    public function delete(int $id): bool;
}

/**
 * Notification port
 */
interface NotificationPort
{
    public function sendWelcomeEmail(Client $client): void;
    public function sendOrderConfirmation(Order $order): void;
    public function sendAlert(string $recipient, string $subject, string $message): void;
}

/**
 * Logger port
 */
interface LoggerPort
{
    public function info(string $message, array $context = []): void;
    public function error(string $message, array $context = []): void;
    public function debug(string $message, array $context = []): void;
}
```

## Use Cases

### Create Client Use Case

```php
<?php
/**
 * Create client use case
 */
class CreateClientUseCase
{
    private ClientRepositoryPort $repository;
    private NotificationPort $notifications;
    private LoggerPort $logger;

    public function __construct(
        ClientRepositoryPort $repository,
        NotificationPort $notifications,
        LoggerPort $logger
    ) {
        $this->repository = $repository;
        $this->notifications = $notifications;
        $this->logger = $logger;
    }

    public function execute(CreateClientInput $input): ClientOutput
    {
        // Check for existing client
        $existingClient = $this->repository->findByEmail($input->email);
        if ($existingClient) {
            throw new ClientAlreadyExistsException($input->email);
        }

        // Create domain entity
        $client = Client::create(
            $input->firstname,
            $input->lastname,
            $input->email
        );

        // Persist
        $this->repository->save($client);

        // Send notification
        $this->notifications->sendWelcomeEmail($client);

        // Log
        $this->logger->info('Client created', [
            'client_id' => $client->getId(),
            'email' => (string) $client->getEmail(),
        ]);

        return ClientOutput::fromEntity($client);
    }
}

/**
 * Input DTO
 */
class CreateClientInput
{
    public string $firstname;
    public string $lastname;
    public Email $email;
    public ?string $company;
    public ?string $phone;

    public static function fromArray(array $data): self
    {
        $input = new self();
        $input->firstname = $data['firstname'];
        $input->lastname = $data['lastname'];
        $input->email = new Email($data['email']);
        $input->company = $data['companyname'] ?? null;
        $input->phone = $data['phonenumber'] ?? null;

        return $input;
    }
}

/**
 * Output DTO
 */
class ClientOutput
{
    public int $id;
    public string $firstname;
    public string $lastname;
    public string $email;
    public string $fullName;
    public string $status;
    public string $createdAt;

    public static function fromEntity(Client $client): self
    {
        $output = new self();
        $output->id = $client->getId();
        $output->firstname = $client->firstname;
        $output->lastname = $client->lastname;
        $output->email = (string) $client->getEmail();
        $output->fullName = $client->getFullName();
        $output->status = $client->getStatus()->getValue();
        $output->createdAt = $client->createdAt->format('Y-m-d H:i:s');

        return $output;
    }

    public function toArray(): array
    {
        return [
            'id' => $this->id,
            'firstname' => $this->firstname,
            'lastname' => $this->lastname,
            'email' => $this->email,
            'full_name' => $this->fullName,
            'status' => $this->status,
            'created_at' => $this->createdAt,
        ];
    }
}
```

## Adapters (Implementations)

### Primary Adapters

```php
<?php
/**
 * HTTP controller adapter
 */
class ClientControllerAdapter
{
    private ClientServicePort $clientService;

    public function __construct(ClientServicePort $clientService)
    {
        $this->clientService = $clientService;
    }

    public function create(array $requestData): array
    {
        try {
            $input = CreateClientInput::fromArray($requestData);
            $output = $this->clientService->createClient($input);

            return [
                'status' => 201,
                'body' => ['success' => true, 'data' => $output->toArray()],
            ];
        } catch (ValidationException $e) {
            return [
                'status' => 422,
                'body' => ['success' => false, 'errors' => $e->getErrors()],
            ];
        } catch (ClientAlreadyExistsException $e) {
            return [
                'status' => 409,
                'body' => ['success' => false, 'message' => $e->getMessage()],
            ];
        }
    }

    public function show(int $clientId): array
    {
        $client = $this->clientService->getClient($clientId);

        if (!$client) {
            return ['status' => 404, 'body' => ['success' => false, 'message' => 'Not found']];
        }

        return [
            'status' => 200,
            'body' => ['success' => true, 'data' => $client->toArray()],
        ];
    }
}

/**
 * WHMCS hook adapter
 */
class WHMCSHookAdapter
{
    private ClientServicePort $clientService;

    public function __construct(ClientServicePort $clientService)
    {
        $this->clientService = $clientService;
    }

    public function handleClientAdd(array $vars): array
    {
        $input = CreateClientInput::fromArray($vars);
        return $this->clientService->createClient($input);
    }
}
```

### Secondary Adapters

```php
<?php
/**
 * Database client repository adapter
 */
class DatabaseClientRepositoryAdapter implements ClientRepositoryPort
{
    private string $table = 'tblclients';

    public function findById(int $id): ?Client
    {
        $data = Capsule::table($this->table)->find($id);

        if (!$data) {
            return null;
        }

        return $this->hydrateClient((array) $data);
    }

    public function findByEmail(Email $email): ?Client
    {
        $data = Capsule::table($this->table)
            ->where('email', (string) $email)
            ->first();

        if (!$data) {
            return null;
        }

        return $this->hydrateClient((array) $data);
    }

    public function save(Client $client): void
    {
        $data = $client->toArray();

        if ($client->getId()) {
            Capsule::table($this->table)
                ->where('id', $client->getId())
                ->update($data);
        } else {
            $newId = Capsule::table($this->table)->insertGetId($data);
            // Update entity ID (this is a bit awkward - consider better approach)
        }
    }

    public function delete(int $id): bool
    {
        return Capsule::table($this->table)
            ->where('id', $id)
            ->delete() > 0;
    }

    private function hydrateClient(array $data): Client
    {
        $client = Client::create(
            $data['firstname'],
            $data['lastname'],
            new Email($data['email'])
        );

        // Set ID using reflection (in production, use a proper method)
        $reflection = new ReflectionClass($client);
        $property = $reflection->getProperty('id');
        $property->setAccessible(true);
        $property->setValue($client, (int) $data['id']);

        return $client;
    }
}

/**
 * Email notification adapter
 */
class EmailNotificationAdapter implements NotificationPort
{
    private EmailServiceInterface $emailService;

    public function __construct(EmailServiceInterface $emailService)
    {
        $this->emailService = $emailService;
    }

    public function sendWelcomeEmail(Client $client): void
    {
        $this->emailService->send(
            $client->getId(),
            'welcome_email',
            ['client_name' => $client->getFullName()]
        );
    }

    public function sendOrderConfirmation(Order $order): void
    {
        $this->emailService->send(
            $order->getClientId(),
            'order_confirmation',
            ['order_id' => $order->getId()]
        );
    }

    public function sendAlert(string $recipient, string $subject, string $message): void
    {
        $this->emailService->sendRaw($recipient, $subject, $message);
    }
}

/**
 * WHMCS logger adapter
 */
class WHMCSLoggerAdapter implements LoggerPort
{
    public function info(string $message, array $context = []): void
    {
        logActivity($message . ' ' . json_encode($context));
    }

    public function error(string $message, array $context = []): void
    {
        logActivity('ERROR: ' . $message . ' ' . json_encode($context));
    }

    public function debug(string $message, array $context = []): void
    {
        // Only log in debug mode
        if (App::get_config('debug_mode')) {
            logActivity('DEBUG: ' . $message . ' ' . json_encode($context));
        }
    }
}
```

## Service Implementation

### Service Using Ports

```php
<?php
/**
 * Client service implementation
 */
class ClientServiceImpl implements ClientServicePort
{
    private CreateClientUseCase $createClientUseCase;
    private GetClientUseCase $getClientUseCase;
    private UpdateClientUseCase $updateClientUseCase;
    private SearchClientsUseCase $searchClientsUseCase;

    public function __construct(
        CreateClientUseCase $createClientUseCase,
        GetClientUseCase $getClientUseCase,
        UpdateClientUseCase $updateClientUseCase,
        SearchClientsUseCase $searchClientsUseCase
    ) {
        $this->createClientUseCase = $createClientUseCase;
        $this->getClientUseCase = $getClientUseCase;
        $this->updateClientUseCase = $updateClientUseCase;
        $this->searchClientsUseCase = $searchClientsUseCase;
    }

    public function createClient(CreateClientInput $input): ClientOutput
    {
        return $this->createClientUseCase->execute($input);
    }

    public function getClient(int $id): ?ClientOutput
    {
        return $this->getClientUseCase->execute($id);
    }

    public function updateClient(int $id, UpdateClientInput $input): ClientOutput
    {
        return $this->updateClientUseCase->execute($id, $input);
    }

    public function searchClients(string $term): array
    {
        return $this->searchClientsUseCase->execute($term);
    }
}
```

## DI Setup

```php
<?php
/**
 * Hexagonal architecture DI setup
 */
class HexagonalBootstrap
{
    private Container $container;

    public function __construct(Container $container)
    {
        $this->container = $container;
    }

    public function register(): void
    {
        // Register adapters (driven)
        $this->container->singleton(
            ClientRepositoryPort::class,
            DatabaseClientRepositoryAdapter::class
        );

        $this->container->singleton(
            NotificationPort::class,
            EmailNotificationAdapter::class
        );

        $this->container->singleton(
            LoggerPort::class,
            WHMCSLoggerAdapter::class
        );

        // Register use cases
        $this->container->bind(
            CreateClientUseCase::class,
            fn($c) => new CreateClientUseCase(
                $c->make(ClientRepositoryPort::class),
                $c->make(NotificationPort::class),
                $c->make(LoggerPort::class)
            )
        );

        // Register services (primary port implementations)
        $this->container->bind(
            ClientServicePort::class,
            ClientServiceImpl::class
        );
    }
}
```

## Best Practices

1. **Keep domain pure** - No framework dependencies in domain
2. **Define clear ports** - Use interfaces for all external dependencies
3. **One direction dependencies** - Outer layers depend on inner, never reverse
4. **Use value objects** - Represent domain concepts as immutable objects
5. **Single responsibility** - Each adapter has one purpose
6. **Test the core** - Domain and use cases should be easily testable
7. **Decouple persistence** - Database adapters can be swapped

## Related Patterns

- [Service Layer](./service-layer.md) - Use case organization
- [Repository Pattern](./repository-pattern.md) - Data access ports
- [Dependency Injection](./dependency-injection.md) - Wiring adapters
- [CQRS Pattern](./cqrs-pattern.md) - Read/write separation