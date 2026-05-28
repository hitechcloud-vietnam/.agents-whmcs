# Service Layer Architecture

The Service Layer pattern provides a centralized place to orchestrate business operations, keeping controllers thin and business logic reusable.

## Service Interface

```php
<?php
/**
 * Service interface
 */
interface ServiceInterface
{
    /**
     * Get the service name
     */
    public function getName(): string;

    /**
     * Initialize the service
     */
    public function boot(): void;
}
```

## Base Service

```php
<?php
/**
 * Abstract base service
 */
abstract class AbstractService implements ServiceInterface
{
    protected Container $container;
    protected EventBusInterface $eventBus;
    protected LoggerInterface $logger;

    public function __construct(
        Container $container,
        EventBusInterface $eventBus,
        LoggerInterface $logger
    ) {
        $this->container = $container;
        $this->eventBus = $eventBus;
        $this->logger = $logger;
    }

    public function boot(): void
    {
        // Override in subclasses
    }

    /**
     * Get a repository
     */
    protected function repository(string $name): RepositoryInterface
    {
        return $this->container->make($name);
    }

    /**
     * Get a sub-service
     */
    protected function service(string $name): ServiceInterface
    {
        return $this->container->make($name);
    }

    /**
     * Dispatch an event
     */
    protected function dispatch(Event $event): void
    {
        $this->eventBus->publish($event);
    }

    /**
     * Log info message
     */
    protected function logInfo(string $message, array $context = []): void
    {
        $this->logger->info($message, $context);
    }
}
```

## Client Service

```php
<?php
/**
 * Client service for business operations
 */
class ClientService extends AbstractService
{
    private ClientRepositoryInterface $clientRepo;
    private EmailServiceInterface $emailService;
    private AuditLogger $auditLogger;

    public function __construct(
        Container $container,
        EventBusInterface $eventBus,
        LoggerInterface $logger,
        ClientRepositoryInterface $clientRepo,
        EmailServiceInterface $emailService,
        AuditLogger $auditLogger
    ) {
        parent::__construct($container, $eventBus, $logger);

        $this->clientRepo = $clientRepo;
        $this->emailService = $emailService;
        $this->auditLogger = $auditLogger;
    }

    public function getName(): string
    {
        return 'client_service';
    }

    /**
     * Create a new client
     */
    public function createClient(array $data): Client
    {
        // Validate data
        $this->validateClientData($data);

        // Check for duplicate email
        if ($this->clientRepo->findByEmail($data['email'])) {
            throw new ClientException('A client with this email already exists');
        }

        // Create client entity
        $client = Client::create($data);

        // Persist
        $this->clientRepo->save($client);

        // Audit
        $this->auditLogger->log('client_created', [
            'client_id' => $client->getId(),
            'by' => $this->getCurrentAdminId(),
        ]);

        // Dispatch event
        $this->dispatch(new ClientCreatedEvent($client->getId(), $data));

        // Send welcome email
        $this->emailService->sendWelcome($client);

        $this->logInfo('Client created', ['client_id' => $client->getId()]);

        return $client;
    }

    /**
     * Update client
     */
    public function updateClient(int $clientId, array $updates): Client
    {
        $client = $this->clientRepo->find($clientId);

        if (!$client) {
            throw new ClientNotFoundException("Client $clientId not found");
        }

        $previousData = $client->toArray();

        // Apply updates
        $client->update($updates);

        // Validate
        $this->validateClientData($client->toArray(), $clientId);

        // Persist
        $this->clientRepo->save($client);

        // Audit
        $this->auditLogger->log('client_updated', [
            'client_id' => $clientId,
            'changes' => $this->getChanges($previousData, $updates),
            'by' => $this->getCurrentAdminId(),
        ]);

        // Dispatch event
        $this->dispatch(new ClientUpdatedEvent($clientId, $updates, $previousData));

        $this->logInfo('Client updated', ['client_id' => $clientId]);

        return $client;
    }

    /**
     * Search clients
     */
    public function searchClients(string $term, int $limit = 20): array
    {
        return $this->clientRepo->search($term);
    }

    /**
     * Get client statistics
     */
    public function getClientStats(int $clientId): array
    {
        $client = $this->clientRepo->find($clientId);

        if (!$client) {
            throw new ClientNotFoundException("Client $clientId not found");
        }

        return [
            'orders' => $this->getOrderStats($clientId),
            'services' => $this->getServiceStats($clientId),
            'invoices' => $this->getInvoiceStats($clientId),
            'total_spent' => $this->getTotalSpent($clientId),
        ];
    }

    /**
     * Archive client
     */
    public function archiveClient(int $clientId, string $reason): void
    {
        $client = $this->clientRepo->find($clientId);

        if (!$client) {
            throw new ClientNotFoundException("Client $clientId not found");
        }

        $client->archive($reason);
        $this->clientRepo->save($client);

        $this->auditLogger->log('client_archived', [
            'client_id' => $clientId,
            'reason' => $reason,
            'by' => $this->getCurrentAdminId(),
        ]);

        $this->dispatch(new ClientArchivedEvent($clientId, $reason));
    }

    private function validateClientData(array $data, ?int $excludeId = null): void
    {
        $required = ['firstname', 'lastname', 'email'];

        foreach ($required as $field) {
            if (empty($data[$field])) {
                throw new ValidationException("Field '{$field}' is required");
            }
        }

        if (!filter_var($data['email'], FILTER_VALIDATE_EMAIL)) {
            throw new ValidationException('Invalid email address');
        }
    }

    private function getChanges(array $previous, array $current): array
    {
        $changes = [];

        foreach ($current as $key => $value) {
            if (isset($previous[$key]) && $previous[$key] !== $value) {
                $changes[$key] = [
                    'from' => $previous[$key],
                    'to' => $value,
                ];
            }
        }

        return $changes;
    }

    private function getCurrentAdminId(): ?int
    {
        return $_SESSION['adminid'] ?? null;
    }

    private function getOrderStats(int $clientId): array
    {
        $repo = $this->repository(OrderRepositoryInterface::class);
        return $repo->getOrderStats($clientId);
    }

    private function getServiceStats(int $clientId): array
    {
        return Capsule::table('tblhosting')
            ->where('userid', $clientId)
            ->selectRaw("COUNT(*) as total, SUM(domainstatus = 'Active') as active")
            ->first();
    }

    private function getInvoiceStats(int $clientId): array
    {
        return Capsule::table('tblinvoices')
            ->where('userid', $clientId)
            ->selectRaw("
                COUNT(*) as total,
                SUM(status = 'Paid') as paid,
                SUM(status = 'Unpaid') as unpaid,
                SUM(total) as total_amount
            ")
            ->first();
    }

    private function getTotalSpent(int $clientId): float
    {
        return Capsule::table('tblinvoices')
            ->where('userid', $clientId)
            ->where('status', 'Paid')
            ->sum('total') ?? 0.0;
    }
}
```

## Order Service

```php
<?php
/**
 * Order processing service
 */
class OrderService extends AbstractService
{
    private OrderRepositoryInterface $orderRepo;
    private ClientRepositoryInterface $clientRepo;
    private ProductServiceInterface $productService;
    private PaymentServiceInterface $paymentService;
    private InventoryService $inventoryService;

    public function __construct(
        Container $container,
        EventBusInterface $eventBus,
        LoggerInterface $logger,
        OrderRepositoryInterface $orderRepo,
        ClientRepositoryInterface $clientRepo,
        ProductServiceInterface $productService,
        PaymentServiceInterface $paymentService,
        InventoryService $inventoryService
    ) {
        parent::__construct($container, $eventBus, $logger);

        $this->orderRepo = $orderRepo;
        $this->clientRepo = $clientRepo;
        $this->productService = $productService;
        $this->paymentService = $paymentService;
        $this->inventoryService = $inventoryService;
    }

    public function getName(): string
    {
        return 'order_service';
    }

    /**
     * Create a new order
     */
    public function createOrder(int $clientId, array $items, array $paymentMethod = []): Order
    {
        // Validate client
        $client = $this->clientRepo->find($clientId);
        if (!$client) {
            throw new ClientNotFoundException("Client $clientId not found");
        }

        // Validate items
        $this->validateOrderItems($items);

        // Calculate totals
        $totals = $this->calculateTotals($items);

        // Check inventory
        $this->checkInventory($items);

        // Create order
        $order = Order::create($clientId, $items, $totals);

        // Save order
        $this->orderRepo->save($order);

        // Reserve inventory
        foreach ($items as $item) {
            $this->inventoryService->reserve(
                $item['product_id'],
                $item['quantity'] ?? 1
            );
        }

        // Process payment if applicable
        if ($totals['total'] > 0 && !empty($paymentMethod)) {
            $this->processPayment($order, $paymentMethod);
        }

        // Audit
        $this->auditOrderCreation($order);

        // Dispatch event
        $this->dispatch(new OrderPlacedEvent(
            $order->getId(),
            $clientId,
            $items,
            $totals['total']
        ));

        $this->logInfo('Order created', [
            'order_id' => $order->getId(),
            'client_id' => $clientId,
            'total' => $totals['total'],
        ]);

        return $order;
    }

    /**
     * Cancel an order
     */
    public function cancelOrder(int $orderId, string $reason): void
    {
        $order = $this->orderRepo->find($orderId);

        if (!$order) {
            throw new OrderNotFoundException("Order $orderId not found");
        }

        if ($order->isCancelled()) {
            throw new OrderException('Order is already cancelled');
        }

        $previousStatus = $order->getStatus();

        // Cancel order
        $order->cancel($reason);
        $this->orderRepo->save($order);

        // Release inventory
        foreach ($order->getItems() as $item) {
            $this->inventoryService->release(
                $item['product_id'],
                $item['quantity'] ?? 1
            );
        }

        // Refund if applicable
        if ($order->hasBeenPaid()) {
            $this->paymentService->refund($order);
        }

        // Dispatch event
        $this->dispatch(new OrderCancelledEvent($orderId, $reason, $previousStatus));

        $this->logInfo('Order cancelled', [
            'order_id' => $orderId,
            'reason' => $reason,
        ]);
    }

    /**
     * Get order with related data
     */
    public function getOrderDetails(int $orderId): array
    {
        $order = $this->orderRepo->find($orderId);

        if (!$order) {
            throw new OrderNotFoundException("Order $orderId not found");
        }

        return [
            'order' => $order,
            'client' => $this->clientRepo->find($order->getClientId()),
            'items' => $this->getOrderItems($orderId),
            'invoices' => $this->getOrderInvoices($orderId),
            'transactions' => $this->getOrderTransactions($orderId),
        ];
    }

    private function validateOrderItems(array $items): void
    {
        foreach ($items as $index => $item) {
            if (empty($item['product_id'])) {
                throw new ValidationException("Product ID is required for item $index");
            }

            if (!isset($item['quantity']) || $item['quantity'] < 1) {
                $items[$index]['quantity'] = 1;
            }
        }
    }

    private function calculateTotals(array $items): array
    {
        $subtotal = 0;
        $tax = 0;

        foreach ($items as $item) {
            $price = $this->productService->getPrice($item['product_id']);
            $quantity = $item['quantity'] ?? 1;
            $subtotal += $price * $quantity;
        }

        $taxRate = 0.0; // Get from config
        $tax = $subtotal * $taxRate;

        return [
            'subtotal' => $subtotal,
            'tax' => $tax,
            'total' => $subtotal + $tax,
        ];
    }

    private function checkInventory(array $items): void
    {
        foreach ($items as $item) {
            if (!$this->inventoryService->isAvailable(
                $item['product_id'],
                $item['quantity'] ?? 1
            )) {
                throw new InventoryException(
                    "Product {$item['product_id']} is not available"
                );
            }
        }
    }

    private function processPayment(Order $order, array $paymentMethod): void
    {
        // Process payment through payment service
    }

    private function auditOrderCreation(Order $order): void
    {
        $this->auditLogger->log('order_created', [
            'order_id' => $order->getId(),
            'client_id' => $order->getClientId(),
            'total' => $order->getTotal(),
            'by' => $this->getCurrentAdminId(),
        ]);
    }

    private function getOrderItems(int $orderId): array
    {
        return Capsule::table('tblorderitems')
            ->where('orderid', $orderId)
            ->get();
    }

    private function getOrderInvoices(int $orderId): array
    {
        return Capsule::table('tblinvoices')
            ->where('order_id', $orderId)
            ->get();
    }

    private function getOrderTransactions(int $orderId): array
    {
        return Capsule::table('tblaccounts')
            ->where('invoice_id', $orderId)
            ->get();
    }
}
```

## Transactional Service

```php
<?php
/**
 * Service with transaction support
 */
class TransactionalService extends AbstractService
{
    /**
     * Execute operation in transaction
     */
    protected function transaction(callable $operation): mixed
    {
        return Capsule::connection()->transaction($operation);
    }

    /**
     * Execute with retry on deadlock
     */
    protected function withRetry(callable $operation, int $maxRetries = 3): mixed
    {
        $attempts = 0;

        while ($attempts < $maxRetries) {
            try {
                return $this->transaction($operation);
            } catch (\PDOException $e) {
                if ($e->getCode() === '40001' && $attempts < $maxRetries - 1) {
                    // Deadlock detected, retry
                    $attempts++;
                    usleep(rand(10000, 50000));
                    continue;
                }

                throw $e;
            }
        }
    }
}
```

## Service Factory

```php
<?php
/**
 * Service factory
 */
class ServiceFactory
{
    private Container $container;
    private array $services = [];

    public function __construct(Container $container)
    {
        $this->container = $container;
    }

    /**
     * Get or create a service
     */
    public function make(string $serviceClass): ServiceInterface
    {
        if (!isset($this->services[$serviceClass])) {
            $this->services[$serviceClass] = $this->container->make($serviceClass);
        }

        return $this->services[$serviceClass];
    }

    /**
     * Register a service
     */
    public function register(string $name, string $serviceClass): void
    {
        $this->services[$name] = $this->container->make($serviceClass);
    }
}
```

## Best Practices

1. **Single responsibility** - Each service handles one domain
2. **Use interfaces** - Define service contracts
3. **Dependency injection** - Inject repositories and other services
4. **Transaction management** - Wrap complex operations in transactions
5. **Event dispatching** - Decouple with events
6. **Logging** - Log all business operations
7. **Validation** - Validate inputs before processing
8. **Exception handling** - Throw meaningful exceptions

## Related Patterns

- [Repository Pattern](./repository-pattern.md) - Data access
- [Dependency Injection](./dependency-injection.md) - Service creation
- [Event Sourcing](./event-sourcing.md) - Event dispatching
- [Service Layer](./service-layer.md) - Architecture guidance