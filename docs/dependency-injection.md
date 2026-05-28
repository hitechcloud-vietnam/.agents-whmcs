# Dependency Injection in WHMCS

Dependency injection (DI) is a design pattern that helps manage dependencies and promotes loose coupling. This guide covers implementing DI patterns in WHMCS modules.

## DI Container

### Simple Container

```php
<?php
/**
 * Simple dependency injection container
 */
class Container
{
    private array $bindings = [];
    private array $instances = [];
    private array $aliases = [];

    /**
     * Bind an interface to an implementation
     */
    public function bind(string $abstract, string $concrete, bool $shared = false): void
    {
        $this->bindings[$abstract] = [
            'concrete' => $concrete,
            'shared' => $shared,
        ];
    }

    /**
     * Bind a singleton instance
     */
    public function singleton(string $abstract, $instance): void
    {
        $this->instances[$abstract] = $instance;
    }

    /**
     * Register an alias for an abstract
     */
    public function alias(string $abstract, string $alias): void
    {
        $this->aliases[$alias] = $abstract;
    }

    /**
     * Resolve a dependency
     */
    public function make(string $abstract): mixed
    {
        // Resolve alias
        $abstract = $this->aliases[$abstract] ?? $abstract;

        // Return existing instance if singleton
        if (isset($this->instances[$abstract])) {
            return $this->instances[$abstract];
        }

        // Get binding
        if (!isset($this->bindings[$abstract])) {
            // Try to auto-resolve
            return $this->autoResolve($abstract);
        }

        $binding = $this->bindings[$abstract];
        $concrete = $binding['concrete'];

        // Build instance
        $instance = $this->build($concrete);

        // Store if shared
        if ($binding['shared']) {
            $this->instances[$abstract] = $instance;
        }

        return $instance;
    }

    /**
     * Build a class instance
     */
    private function build(string $concrete): object
    {
        // Check if concrete is a closure
        if ($concrete instanceof \Closure) {
            return $concrete($this);
        }

        // Use reflection to build
        $reflector = new ReflectionClass($concrete);

        if (!$reflector->isInstantiable()) {
            throw new ContainerException("Target [$concrete] is not instantiable");
        }

        $constructor = $reflector->getConstructor();

        // No constructor - just instantiate
        if (!$constructor) {
            return new $concrete();
        }

        // Get constructor parameters
        $params = $constructor->getParameters();
        $dependencies = $this->resolveDependencies($params);

        return $reflector->newInstanceArgs($dependencies);
    }

    /**
     * Resolve dependencies
     */
    private function resolveDependencies(array $params): array
    {
        $dependencies = [];

        foreach ($params as $param) {
            $type = $param->getType();
            $name = $param->getName();

            if ($type && !$type->isBuiltin()) {
                // Class or interface type hint
                $typeName = $type->getName();
                $dependencies[] = $this->make($typeName);
            } elseif ($param->isDefaultValueAvailable()) {
                // Has default value
                $dependencies[] = $param->getDefaultValue();
            } else {
                throw new ContainerException(
                    "Unresolvable dependency [$name] in {$param->getDeclaringClass()->getName()}"
                );
            }
        }

        return $dependencies;
    }

    /**
     * Auto-resolve a concrete class
     */
    private function autoResolve(string $abstract): object
    {
        if (!class_exists($abstract)) {
            throw new ContainerException("Class [$abstract] does not exist");
        }

        return $this->build($abstract);
    }

    /**
     * Check if abstract is bound
     */
    public function bound(string $abstract): bool
    {
        $abstract = $this->aliases[$abstract] ?? $abstract;
        return isset($this->bindings[$abstract]) || isset($this->instances[$abstract]);
    }
}
```

## Service Registration

### Service Provider Pattern

```php
<?php
/**
 * Service provider base class
 */
abstract class ServiceProvider
{
    protected Container $container;
    protected array $deferred = [];

    public function __construct(Container $container)
    {
        $this->container = $container;
    }

    /**
     * Register services
     */
    abstract public function register(): void;

    /**
     * Boot services (after all registered)
     */
    public function boot(): void
    {
        // Override in subclasses
    }

    /**
     * Get deferred services
     */
    public function provides(): array
    {
        return $this->deferred;
    }

    /**
     * Check if provider is deferred
     */
    public function isDeferred(): bool
    {
        return !empty($this->deferred);
    }
}

/**
 * Application service provider
 */
class ApplicationServiceProvider extends ServiceProvider
{
    public function register(): void
    {
        // Bind services
        $this->container->singleton(CacheManagerInterface::class, RedisCacheStore::class);
        $this->container->singleton(DatabaseQueue::class);
        $this->container->singleton(EventBusInterface::class, SimpleEventBus::class);

        // Bind repositories
        $this->container->bind(ClientRepositoryInterface::class, ClientRepository::class);
        $this->container->bind(OrderRepositoryInterface::class, OrderRepository::class);

        // Bind services
        $this->container->bind(EmailServiceInterface::class, EmailService::class);
        $this->container->bind(PaymentServiceInterface::class, PaymentService::class);

        $this->deferred = [
            CacheManagerInterface::class,
            DatabaseQueue::class,
            EventBusInterface::class,
            ClientRepositoryInterface::class,
            OrderRepositoryInterface::class,
            EmailServiceInterface::class,
            PaymentServiceInterface::class,
        ];
    }

    public function boot(): void
    {
        // Initialize event listeners
        $eventBus = $this->container->make(EventBusInterface::class);
        $eventBus->subscribe(ClientCreatedEvent::class, new SendWelcomeEmailHandler());

        // Initialize cache
        $cache = $this->container->make(CacheManagerInterface::class);
        $cache->setPrefix('app_');
    }
}
```

### Module Service Provider

```php
<?php
/**
 * Module service provider
 */
class ModuleServiceProvider extends ServiceProvider
{
    public function register(): void
    {
        // Register module-specific services
        $this->container->bind(ModuleServiceInterface::class, ModuleService::class);
        $this->container->singleton(ModuleConfigRepository::class);
        $this->container->bind(ModuleWebhookHandler::class);

        // Register with deferred loading
        $this->deferred = [
            ModuleServiceInterface::class,
            ModuleConfigRepository::class,
            ModuleWebhookHandler::class,
        ];
    }

    public function boot(): void
    {
        // Register hooks
        add_hook('ClientAreaPage', 1, function ($vars) {
            $service = $this->container->make(ModuleServiceInterface::class);
            return $service->handleClientPage($vars);
        });
    }
}
```

## Constructor Injection

### Example Service Classes

```php
<?php
/**
 * Example service with dependencies
 */
class OrderService
{
    private ClientRepositoryInterface $clientRepository;
    private OrderRepositoryInterface $orderRepository;
    private EventBusInterface $eventBus;
    private CacheManagerInterface $cache;

    public function __construct(
        ClientRepositoryInterface $clientRepository,
        OrderRepositoryInterface $orderRepository,
        EventBusInterface $eventBus,
        CacheManagerInterface $cache
    ) {
        $this->clientRepository = $clientRepository;
        $this->orderRepository = $orderRepository;
        $this->eventBus = $eventBus;
        $this->cache = $cache;
    }

    public function createOrder(int $clientId, array $items): Order
    {
        $client = $this->clientRepository->find($clientId);

        if (!$client) {
            throw new ClientNotFoundException("Client $clientId not found");
        }

        $order = Order::create($clientId, $items);

        $this->orderRepository->save($order);
        $this->invalidateClientCache($clientId);

        $this->eventBus->publish(new OrderCreatedEvent($order->getId(), $clientId));

        return $order;
    }

    private function invalidateClientCache(int $clientId): void
    {
        $this->cache->invalidateTags(['client_' . $clientId, 'orders']);
    }
}

/**
 * Repository interface
 */
interface OrderRepositoryInterface
{
    public function find(int $id): ?Order;
    public function save(Order $order): void;
    public function delete(int $id): void;
}

/**
 * Repository implementation
 */
class OrderRepository implements OrderRepositoryInterface
{
    private string $table = 'tblorders';

    public function find(int $id): ?Order
    {
        $data = Capsule::table($this->table)->find($id);

        if (!$data) {
            return null;
        }

        return Order::fromArray((array) $data);
    }

    public function save(Order $order): void
    {
        $data = $order->toArray();

        Capsule::table($this->table)->updateOrInsert(
            ['id' => $data['id'] ?? null],
            $data
        );
    }

    public function delete(int $id): void
    {
        Capsule::table($this->table)->where('id', $id)->delete();
    }
}
```

## Method Injection

### Method-Based DI

```php
<?php
/**
 * Example with method injection
 */
class ReportGenerator
{
    /**
     * Generate report with injectable formatter
     */
    public function generate(ReportData $data, ?ReportFormatterInterface $formatter = null): string
    {
        $formatter = $formatter ?? new HtmlReportFormatter();

        return $formatter->format($data);
    }

    /**
     * Generate with cache
     */
    public function generateCached(
        ReportData $data,
        CacheManagerInterface $cache
    ): string {
        $key = 'report_' . md5(json_encode($data->toArray()));

        return $cache->remember($key, 3600, function () use ($data, $formatter) {
            return $this->generate($data);
        });
    }
}

// Usage
$generator = $container->make(ReportGenerator::class);
$report = $generator->generate($data);
```

## Factory Pattern

### Service Factory

```php
<?php
/**
 * Factory for creating services
 */
class ServiceFactory
{
    private Container $container;

    public function __construct(Container $container)
    {
        $this->container = $container;
    }

    /**
     * Create service with runtime parameters
     */
    public function make(string $serviceClass, array $parameters = []): object
    {
        $reflector = new ReflectionClass($serviceClass);

        if (!$reflector->isInstantiable()) {
            throw new ContainerException("Class $serviceClass is not instantiable");
        }

        $constructor = $reflector->getConstructor();

        if (!$constructor) {
            return new $serviceClass();
        }

        $params = $constructor->getParameters();
        $dependencies = [];

        foreach ($params as $param) {
            $type = $param->getType();

            if ($type && !$type->isBuiltin() && class_exists($type->getName())) {
                // Resolve from container
                $dependencies[] = $this->container->make($type->getName());
            } elseif (isset($parameters[$param->getName()])) {
                // Use provided parameter
                $dependencies[] = $parameters[$param->getName()];
            } elseif ($param->isDefaultValueAvailable()) {
                $dependencies[] = $param->getDefaultValue();
            } else {
                throw new ContainerException(
                    "Cannot resolve parameter {$param->getName()}"
                );
            }
        }

        return $reflector->newInstanceArgs($dependencies);
    }
}
```

## Integration with WHMCS

### WHMCS Module DI Setup

```php
<?php
/**
 * Initialize DI in module
 */
if (!defined("WHMCS")) {
    die("Access denied");
}

// Create container
$container = new Container();

// Register providers
$container->singleton(Container::class, $container);
$container->singleton(ServiceFactory::class, new ServiceFactory($container));

// Register service providers
$providers = [
    ApplicationServiceProvider::class,
    ModuleServiceProvider::class,
];

foreach ($providers as $providerClass) {
    $provider = new $providerClass($container);
    $provider->register();
    $provider->boot();
}

// Store in app
$_SESSION['module_container'] = $container;

// Or use a global helper
function app(): Container
{
    return $_SESSION['module_container'] ?? new Container();
}

// Example usage in hooks
add_hook('ClientAreaPage', 1, function ($vars) {
    $orderService = app()->make(OrderService::class);
    $orders = $orderService->getRecentOrders($vars['userid']);
    return ['orders' => $orders];
});
```

## Best Practices

1. **Program to interfaces** - Depend on abstractions, not concretions
2. **Use constructor injection** - Make dependencies explicit
3. **Single responsibility** - Services should do one thing well
4. **Avoid service locator** - Prefer injection over global access
5. **Manage lifetimes** - Use singletons for shared state
6. **Resolve at last moment** - Don't resolve until needed
7. **Handle exceptions** - Provide clear error messages

## Related Patterns

- [Service Layer](./service-layer.md) - Service architecture
- [Repository Pattern](./repository-pattern.md) - Data access pattern
- [Observer Pattern](./observer-pattern.md) - Event handling