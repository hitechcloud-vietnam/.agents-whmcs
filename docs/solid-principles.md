# SOLID Principles in Module Development

SOLID principles provide guidelines for creating maintainable, flexible, and robust code in WHMCS module development.

## S - Single Responsibility Principle (SRP)

A class should have only one reason to change.

### Violation Example

```php
<?php
/**
 * BAD: Multiple responsibilities
 */
class ClientManager
{
    public function createClient(array $data)
    {
        // Validation
        // Database operations
        // Send email
        // Log activity
        // Update cache
        // Send webhook
    }

    public function updateClient(int $id, array $data)
    {
        // Too many responsibilities
    }
}
```

### Refactored Example

```php
<?php
/**
 * GOOD: Single responsibility - Client creation
 */
class ClientCreator
{
    private ClientValidator $validator;
    private ClientRepositoryInterface $repository;

    public function __construct(
        ClientValidator $validator,
        ClientRepositoryInterface $repository
    ) {
        $this->validator = $validator;
        $this->repository = $repository;
    }

    public function create(array $data): Client
    {
        $this->validator->validateForCreation($data);
        $client = Client::create($data);
        $this->repository->save($client);
        return $client;
    }
}

/**
 * Client validator - handles validation only
 */
class ClientValidator
{
    public function validateForCreation(array $data): void
    {
        $required = ['firstname', 'lastname', 'email'];

        foreach ($required as $field) {
            if (empty($data[$field])) {
                throw new ValidationException("Field '$field' is required");
            }
        }

        if (!filter_var($data['email'], FILTER_VALIDATE_EMAIL)) {
            throw new ValidationException('Invalid email format');
        }
    }

    public function validateForUpdate(array $data): void
    {
        // Update-specific validation
    }
}

/**
 * Client notifier - handles notifications only
 */
class ClientNotifier
{
    private EmailServiceInterface $emailService;

    public function sendWelcome(Client $client): void
    {
        $this->emailService->sendWelcome($client);
    }

    public function sendSuspensionNotice(Client $client): void
    {
        $this->emailService->sendSuspensionNotice($client);
    }
}

/**
 * Client auditor - handles audit logging only
 */
class ClientAuditor
{
    private LoggerInterface $logger;

    public function logCreation(int $clientId): void
    {
        $this->logger->info('Client created', ['client_id' => $clientId]);
    }

    public function logUpdate(int $clientId, array $changes): void
    {
        $this->logger->info('Client updated', [
            'client_id' => $clientId,
            'changes' => $changes,
        ]);
    }
}
```

## O - Open/Closed Principle (OCP)

Software entities should be open for extension but closed for modification.

### Violation Example

```php
<?php
/**
 * BAD: Must modify to add new notification types
 */
class NotificationService
{
    public function send(int $clientId, string $type, string $message)
    {
        if ($type === 'email') {
            // Send email
        } elseif ($type === 'sms') {
            // Send SMS
        } elseif ($type === 'push') {
            // Send push notification
        }
        // Must add new condition for each type
    }
}
```

### Refactored Example

```php
<?php
/**
 * GOOD: Open for extension, closed for modification
 */
interface NotificationChannelInterface
{
    public function send(int $userId, string $message): void;
    public function supports(string $type): bool;
}

class EmailChannel implements NotificationChannelInterface
{
    public function send(int $userId, string $message): void
    {
        // Send email
    }

    public function supports(string $type): bool
    {
        return $type === 'email';
    }
}

class SmsChannel implements NotificationChannelInterface
{
    public function send(int $userId, string $message): void
    {
        // Send SMS
    }

    public function supports(string $type): bool
    {
        return $type === 'sms';
    }
}

class PushChannel implements NotificationChannelInterface
{
    public function send(int $userId, string $message): void
    {
        // Send push notification
    }

    public function supports(string $type): bool
    {
        return $type === 'push';
    }
}

class NotificationService
{
    private array $channels = [];

    public function addChannel(NotificationChannelInterface $channel): void
    {
        $this->channels[] = $channel;
    }

    public function send(int $userId, string $type, string $message): void
    {
        foreach ($this->channels as $channel) {
            if ($channel->supports($type)) {
                $channel->send($userId, $message);
                return;
            }
        }

        throw new NotificationException("Unsupported notification type: $type");
    }
}

// Usage
$service = new NotificationService();
$service->addChannel(new EmailChannel());
$service->addChannel(new SmsChannel());
$service->addChannel(new PushChannel());

// To add WhatsApp: just create WhatsAppChannel and add it
// No modification to NotificationService needed
```

### Strategy Pattern Application

```php
<?php
/**
 * Payment processor - open for new methods, closed for modification
 */
interface PaymentStrategyInterface
{
    public function processPayment(float $amount, array $metadata): PaymentResult;
    public function getMethodName(): string;
    public function supports(float $amount): bool;
}

class PayPalStrategy implements PaymentStrategyInterface
{
    public function processPayment(float $amount, array $metadata): PaymentResult
    {
        // PayPal processing
    }

    public function getMethodName(): string
    {
        return 'paypal';
    }

    public function supports(float $amount): bool
    {
        return $amount <= 10000;
    }
}

class StripeStrategy implements PaymentStrategyInterface
{
    public function processPayment(float $amount, array $metadata): PaymentResult
    {
        // Stripe processing
    }

    public function getMethodName(): string
    {
        return 'stripe';
    }

    public function supports(float $amount): bool
    {
        return true;
    }
}

class PaymentProcessor
{
    private array $strategies = [];

    public function addStrategy(PaymentStrategyInterface $strategy): void
    {
        $this->strategies[] = $strategy;
    }

    public function process(string $method, float $amount, array $metadata): PaymentResult
    {
        foreach ($this->strategies as $strategy) {
            if ($strategy->getMethodName() === $method && $strategy->supports($amount)) {
                return $strategy->processPayment($amount, $metadata);
            }
        }

        throw new PaymentException("No suitable payment method for $method");
    }
}
```

## L - Liskov Substitution Principle (LSP)

Objects of a superclass should be replaceable with objects of subclasses without affecting correctness.

### Violation Example

```php
<?php
/**
 * BAD: Square is not a proper rectangle
 */
class Rectangle
{
    protected int $width;
    protected int $height;

    public function setWidth(int $width): void
    {
        $this->width = $width;
    }

    public function setHeight(int $height): void
    {
        $this->height = $height;
    }

    public function getArea(): int
    {
        return $this->width * $this->height;
    }
}

class Square extends Rectangle
{
    public function setWidth(int $width): void
    {
        $this->width = $width;
        $this->height = $width; // Breaks Rectangle contract
    }

    public function setHeight(int $height): void
    {
        $this->height = $height;
        $this->width = $height;
    }
}

// Usage that breaks LSP
function calculateArea(Rectangle $rect)
{
    $rect->setWidth(5);
    $rect->setHeight(4);
    return $rect->getArea(); // Expects 20, but Square returns 16
}
```

### Refactored Example

```php
<?php
/**
 * GOOD: Proper abstraction for shapes
 */
interface ShapeInterface
{
    public function getArea(): int;
}

class Rectangle implements ShapeInterface
{
    private int $width;
    private int $height;

    public function __construct(int $width, int $height)
    {
        $this->width = $width;
        $this->height = $height;
    }

    public function getArea(): int
    {
        return $this->width * $this->height;
    }
}

class Square implements ShapeInterface
{
    private int $side;

    public function __construct(int $side)
    {
        $this->side = $side;
    }

    public function getArea(): int
    {
        return $this->side * $this->side;
    }
}

// Now both can be used interchangeably
function printArea(ShapeInterface $shape)
{
    echo $shape->getArea();
}
```

### Proper Inheritance for WHMCS

```php
<?php
/**
 * GOOD: Proper inheritance - all implementations can be substituted
 */
interface OrderStateInterface
{
    public function canBeCancelled(): bool;
    public function canBeSuspended(): bool;
    public function getStatusCode(): string;
}

class PendingOrderState implements OrderStateInterface
{
    public function canBeCancelled(): bool
    {
        return true;
    }

    public function canBeSuspended(): bool
    {
        return true;
    }

    public function getStatusCode(): string
    {
        return 'Pending';
    }
}

class ActiveOrderState implements OrderStateInterface
{
    public function canBeCancelled(): bool
    {
        return false; // Can't cancel active orders easily
    }

    public function canBeSuspended(): bool
    {
        return true;
    }

    public function getStatusCode(): string
    {
        return 'Active';
    }
}

class CancelledOrderState implements OrderStateInterface
{
    public function canBeCancelled(): bool
    {
        return false;
    }

    public function canBeSuspended(): bool
    {
        return false;
    }

    public function getStatusCode(): string
    {
        return 'Cancelled';
    }
}

// Order class works with any state implementation
class Order
{
    private OrderStateInterface $state;

    public function setState(OrderStateInterface $state): void
    {
        $this->state = $state;
    }

    public function cancel(): bool
    {
        if ($this->state->canBeCancelled()) {
            // Perform cancellation
            return true;
        }
        return false;
    }

    public function suspend(): bool
    {
        if ($this->state->canBeSuspended()) {
            // Perform suspension
            return true;
        }
        return false;
    }
}
```

## I - Interface Segregation Principle (ISP)

Clients should not be forced to depend on interfaces they do not use.

### Violation Example

```php
<?php
/**
 * BAD: Fat interface - clients forced to implement unused methods
 */
interface EntityOperationsInterface
{
    public function create(array $data);
    public function read(int $id);
    public function update(int $id, array $data);
    public function delete(int $id);
    public function list(int $limit);
    public function search(string $term);
    public function export(string $format);
    public function import(array $data);
    public function validate();
    public function archive();
}

class ClientRepository implements EntityOperationsInterface
{
    // Must implement ALL methods, even if some don't make sense
    public function export(string $format): void
    {
        // What if clients can't be exported in CSV?
    }

    public function import(array $data): void
    {
        // This method might not belong here
    }
}
```

### Refactored Example

```php
<?php
/**
 * GOOD: Segregated interfaces
 */
interface CreateOperationInterface
{
    public function create(array $data): object;
}

interface ReadOperationInterface
{
    public function read(int $id): ?object;
    public function list(int $limit): array;
    public function search(string $term): array;
}

interface UpdateOperationInterface
{
    public function update(int $id, array $data): object;
}

interface DeleteOperationInterface
{
    public function delete(int $id): bool;
}

interface ExportOperationInterface
{
    public function export(string $format): string;
}

interface ImportOperationInterface
{
    public function import(array $data): array;
}

// Clients only implement what they need
interface ClientRepositoryInterface extends
    CreateOperationInterface,
    ReadOperationInterface,
    UpdateOperationInterface,
    DeleteOperationInterface
{
    // Combines only the operations that make sense for clients
}

// Export is separate for entities that support it
interface ExportableInterface
{
    public function export(string $format): string;
}
```

### WHMCS-Specific Application

```php
<?php
/**
 * Segregated interfaces for different module types
 */
interface ModuleConfigInterface
{
    public function getConfig(): array;
    public function saveConfig(array $config): void;
}

interface ModuleActionsInterface
{
    public function getActions(): array;
    public function executeAction(string $action, array $params): mixed;
}

interface ModuleCronInterface
{
    public function getCronFrequency(): string;
    public function executeCron(): CronResult;
}

interface ModuleWidgetInterface
{
    public function getWidgetTitle(): string;
    public function getWidgetData(): array;
}

// Modules implement only what they need
class SimpleModule implements ModuleConfigInterface, ModuleActionsInterface
{
    // Not forced to implement cron or widget interfaces
    public function getConfig(): array
    {
        return [];
    }

    public function saveConfig(array $config): void
    {
        // Save config
    }

    public function getActions(): array
    {
        return [];
    }

    public function executeAction(string $action, array $params): mixed
    {
        // Execute action
    }
}

class DashboardModule implements ModuleConfigInterface, ModuleWidgetInterface
{
    public function getConfig(): array
    {
        return [];
    }

    public function saveConfig(array $config): void
    {
        // Save config
    }

    public function getWidgetTitle(): string
    {
        return 'Dashboard Stats';
    }

    public function getWidgetData(): array
    {
        return [];
    }
}
```

## D - Dependency Inversion Principle (DIP)

High-level modules should not depend on low-level modules. Both should depend on abstractions.

### Violation Example

```php
<?php
/**
 * BAD: High-level module depends on low-level implementation
 */
class ClientService
{
    private DatabaseClientRepository $repository; // Direct dependency

    public function __construct()
    {
        $this->repository = new DatabaseClientRepository(); // Tight coupling
    }

    public function createClient(array $data): Client
    {
        // Uses concrete implementation
        $this->repository->save($data);
    }
}
```

### Refactored Example

```php
<?php
/**
 * GOOD: Depend on abstractions, not concretions
 */
interface ClientRepositoryInterface
{
    public function find(int $id): ?Client;
    public function save(Client $client): void;
    public function delete(int $id): bool;
}

class ClientService
{
    private ClientRepositoryInterface $repository; // Depend on interface

    public function __construct(ClientRepositoryInterface $repository)
    {
        $this->repository = $repository; // Injected, not created
    }

    public function createClient(array $data): Client
    {
        $client = Client::create($data);
        $this->repository->save($client);
        return $client;
    }
}

/**
 * Low-level implementation depends on abstraction
 */
class DatabaseClientRepository implements ClientRepositoryInterface
{
    public function find(int $id): ?Client
    {
        $data = Capsule::table('tblclients')->find($id);
        return $data ? Client::fromArray((array) $data) : null;
    }

    public function save(Client $client): void
    {
        Capsule::table('tblclients')->updateOrInsert(
            ['id' => $client->getId()],
            $client->toArray()
        );
    }

    public function delete(int $id): bool
    {
        return Capsule::table('tblclients')->where('id', $id)->delete() > 0;
    }
}

/**
 * Alternative: Cache repository (can be swapped without changing ClientService)
 */
class CachedClientRepository implements ClientRepositoryInterface
{
    private ClientRepositoryInterface $repository;
    private CacheManagerInterface $cache;

    public function __construct(
        ClientRepositoryInterface $repository,
        CacheManagerInterface $cache
    ) {
        $this->repository = $repository;
        $this->cache = $cache;
    }

    public function find(int $id): ?Client
    {
        $key = "client_$id";
        return $this->cache->remember($key, 3600, function () use ($id) {
            return $this->repository->find($id);
        });
    }

    public function save(Client $client): void
    {
        $this->repository->save($client);
        $this->cache->delete("client_" . $client->getId());
    }

    public function delete(int $id): bool
    {
        $this->cache->delete("client_$id");
        return $this->repository->delete($id);
    }
}
```

### Dependency Injection Container

```php
<?php
/**
 * DI container applying DIP
 */
class Container
{
    private array $bindings = [];

    public function bind(string $abstract, string $concrete): void
    {
        $this->bindings[$abstract] = $concrete;
    }

    public function make(string $abstract): object
    {
        if (!isset($this->bindings[$abstract])) {
            throw new ContainerException("No binding for $abstract");
        }

        $concrete = $this->bindings[$abstract];

        // For interfaces, instantiate the concrete class
        $instance = new $concrete();

        // If concrete has dependencies, resolve them
        $reflector = new ReflectionClass($concrete);
        $constructor = $reflector->getConstructor();

        if ($constructor) {
            $params = $constructor->getParameters();
            $dependencies = [];

            foreach ($params as $param) {
                $type = $param->getType();
                if ($type && !$type->isBuiltin()) {
                    $dependencies[] = $this->make($type->getName());
                }
            }

            return $reflector->newInstanceArgs($dependencies);
        }

        return $instance;
    }
}

// Setup
$container = new Container();

// Bind interface to implementation
$container->bind(ClientRepositoryInterface::class, DatabaseClientRepository::class);

// High-level module gets interface, implementation is injected
$container->bind(ClientService::class, function ($c) {
    return new ClientService($c->make(ClientRepositoryInterface::class));
});

// Usage
$clientService = $container->make(ClientService::class);
$client = $clientService->createClient(['firstname' => 'John', 'lastname' => 'Doe']);
```

## Applying SOLID in WHMCS Modules

### Complete Example

```php
<?php
/**
 * WHMCS Module following SOLID principles
 */

// Single Responsibility: Separate classes for each concern
class ClientValidator
{
    public function validate(array $data): void { /* ... */ }
}

class ClientRepository implements ClientRepositoryInterface
{
    public function find(int $id): ?Client { /* ... */ }
    public function save(Client $client): void { /* ... */ }
}

class ClientNotifier
{
    private EmailServiceInterface $email;
    public function sendWelcome(Client $client): void { /* ... */ }
}

// Open/Closed: Extend through strategy pattern
interface PricingStrategyInterface
{
    public function calculatePrice(Product $product, int $quantity): Money;
}

class StandardPricingStrategy implements PricingStrategyInterface { /* ... */ }
class BulkDiscountStrategy implements PricingStrategyInterface { /* ... */ }

// Liskov: Proper inheritance hierarchy
interface OrderStateHandler
{
    public function canTransitionTo(string $newState): bool;
    public function onEnter(): void;
    public function onExit(): void;
}

// Interface Segregation: Small, focused interfaces
interface ConfigurableInterface { public function getConfig(): array; }
interface ExecutableInterface { public function execute(): mixed; }

// Dependency Inversion: Depend on abstractions
class OrderService
{
    private ClientRepositoryInterface $clients;
    private PricingStrategyInterface $pricing;
    private NotificationPort $notifications;

    public function __construct(
        ClientRepositoryInterface $clients,
        PricingStrategyInterface $pricing,
        NotificationPort $notifications
    ) {
        $this->clients = $clients;
        $this->pricing = $pricing;
        $this->notifications = $notifications;
    }
}
```

## Best Practices Summary

| Principle | Application in WHMCS |
|-----------|---------------------|
| **S** - SRP | Separate validators, repositories, services, notifiers |
| **O** - OCP | Use strategy pattern for payment methods, notification channels |
| **L** - LSP | Ensure subclasses can substitute parent classes properly |
| **I** - ISP | Create small interfaces like `ConfigurableInterface`, `ExecutableInterface` |
| **D** - DIP | Depend on `ClientRepositoryInterface`, not `DatabaseClientRepository` |

## Related Patterns

- [Service Layer](./service-layer.md) - Organizing business logic
- [Repository Pattern](./repository-pattern.md) - Data access abstraction
- [Dependency Injection](./dependency-injection.md) - Dependency management
- [Strategy Pattern](./command-pattern.md) - Open/closed implementation