# Observer Pattern for Hooks

The Observer Pattern enables objects to be notified of changes in other objects. In WHMCS, this pattern is implemented through the hook system.

## Observer Interface

```php
<?php
/**
 * Observer interface
 */
interface ObserverInterface
{
    /**
     * Handle the event notification
     */
    public function update(Event $event): void;
}

/**
 * Subject interface
 */
interface SubjectInterface
{
    /**
     * Attach an observer
     */
    public function attach(ObserverInterface $observer): void;

    /**
     * Detach an observer
     */
    public function detach(ObserverInterface $observer): void;

    /**
     * Notify all observers
     */
    public function notify(): void;
}
```

## Event Subject

### Event Subject Implementation

```php
<?php
/**
 * Event subject for notification
 */
class EventSubject implements SubjectInterface
{
    private array $observers = [];
    private array $events = [];

    public function attach(ObserverInterface $observer): void
    {
        $this->observers[] = $observer;
    }

    public function detach(ObserverInterface $observer): void
    {
        $this->observers = array_filter(
            $this->observers,
            fn($o) => $o !== $observer
        );
    }

    public function notify(): void
    {
        foreach ($this->observers as $observer) {
            foreach ($this->events as $event) {
                $observer->update($event);
            }
        }
    }

    public function attachEvent(string $eventType, ObserverInterface $observer): void
    {
        $this->observers[$eventType][] = $observer;
    }

    public function notifyEvent(string $eventType, Event $event): void
    {
        $observers = $this->observers[$eventType] ?? [];

        foreach ($observers as $observer) {
            try {
                $observer->update($event);
            } catch (\Throwable $e) {
                logActivity("Observer error: " . $e->getMessage());
            }
        }
    }
}
```

## Hook Observers

### Custom Hook Observer

```php
<?php
/**
 * Hook observer for WHMCS hooks
 */
class HookObserver implements ObserverInterface
{
    private string $hookName;
    private int $priority;
    private callable $handler;

    public function __construct(string $hookName, callable $handler, int $priority = 0)
    {
        $this->hookName = $hookName;
        $this->handler = $handler;
        $this->priority = $priority;
    }

    public function update(Event $event): void
    {
        call_user_func($this->handler, $event);
    }

    public function getHookName(): string
    {
        return $this->hookName;
    }

    public function getPriority(): int
    {
        return $this->priority;
    }
}

/**
 * Hook registry
 */
class HookRegistry
{
    private array $hooks = [];

    /**
     * Register a hook observer
     */
    public function register(string $hookName, callable $handler, int $priority = 0): void
    {
        $this->hooks[$hookName][] = [
            'handler' => $handler,
            'priority' => $priority,
        ];

        // Sort by priority
        usort($this->hooks[$hookName], fn($a, $b) => $b['priority'] - $a['priority']);
    }

    /**
     * Execute hooks for an event
     */
    public function execute(string $hookName, array $params = []): array
    {
        $hooks = $this->hooks[$hookName] ?? [];
        $results = [];

        foreach ($hooks as $hook) {
            try {
                $result = call_user_func($hook['handler'], $params);

                if ($result !== null) {
                    $results[] = $result;
                }
            } catch (\Throwable $e) {
                logActivity("Hook error in {$hookName}: " . $e->getMessage());
            }
        }

        return $results;
    }

    /**
     * Check if hook has observers
     */
    public function hasHook(string $hookName): bool
    {
        return !empty($this->hooks[$hookName]);
    }
}
```

## Hook Implementations

### Client Observers

```php
<?php
/**
 * Client created observer
 */
class ClientCreatedObserver implements ObserverInterface
{
    private EmailService $emailService;
    private AuditLogger $auditLogger;
    private CacheManager $cache;

    public function __construct(
        EmailService $emailService,
        AuditLogger $auditLogger,
        CacheManager $cache
    ) {
        $this->emailService = $emailService;
        $this->auditLogger = $auditLogger;
        $this->cache = $cache;
    }

    public function update(Event $event): void
    {
        if (!$event instanceof ClientCreatedEvent) {
            return;
        }

        // Send welcome email
        $this->emailService->send(
            $event->getClientId(),
            'welcome_email',
            ['welcome_name' => $event->getPayload()['firstname']]
        );

        // Log audit
        $this->auditLogger->log('client_created', [
            'client_id' => $event->getClientId(),
        ]);

        // Invalidate cache
        $this->cache->invalidateTags(['clients', 'stats']);
    }
}

/**
 * Order placed observer
 */
class OrderPlacedObserver implements ObserverInterface
{
    private InventoryService $inventory;
    private NotificationService $notifications;
    private StatsService $stats;

    public function __construct(
        InventoryService $inventory,
        NotificationService $notifications,
        StatsService $stats
    ) {
        $this->inventory = $inventory;
        $this->notifications = $notifications;
        $this->stats = $stats;
    }

    public function update(Event $event): void
    {
        if (!$event instanceof OrderPlacedEvent) {
            return;
        }

        $payload = $event->getPayload();

        // Reserve inventory
        foreach ($payload['items'] as $item) {
            $this->inventory->reserve($item['product_id'], $item['quantity']);
        }

        // Send admin notification
        $this->notifications->sendAdminAlert(
            'New Order',
            "Order #{$event->getOrderId()} placed for client {$payload['client_id']}"
        );

        // Update stats
        $this->stats->increment('orders_today');
        $this->stats->addRevenue($payload['total']);
    }
}
```

### Observer Registrations

```php
<?php
/**
 * Observer registration service
 */
class ObserverRegistration
{
    private EventBus $eventBus;

    public function __construct(EventBus $eventBus)
    {
        $this->eventBus = $eventBus;
    }

    /**
     * Register all module observers
     */
    public function registerAll(): void
    {
        $this->registerClientObservers();
        $this->registerOrderObservers();
        $this->registerInvoiceObservers();
        $this->registerServiceObservers();
    }

    private function registerClientObservers(): void
    {
        $this->eventBus->subscribe(
            ClientCreatedEvent::class,
            new ClientCreatedObserver(
                App::make(EmailService::class),
                App::make(AuditLogger::class),
                App::make(CacheManager::class)
            )
        );

        $this->eventBus->subscribe(
            ClientUpdatedEvent::class,
            new ClientUpdatedObserver(
                App::make(CacheManager::class),
                App::make(AuditLogger::class)
            )
        );
    }

    private function registerOrderObservers(): void
    {
        $this->eventBus->subscribe(
            OrderPlacedEvent::class,
            new OrderPlacedObserver(
                App::make(InventoryService::class),
                App::make(NotificationService::class),
                App::make(StatsService::class)
            )
        );
    }

    private function registerInvoiceObservers(): void
    {
        $this->eventBus->subscribe(
            PaymentReceivedEvent::class,
            new PaymentReceivedObserver(
                App::make(InvoiceService::class),
                App::make(EmailService::class),
                App::make(AuditLogger::class)
            )
        );
    }

    private function registerServiceObservers(): void
    {
        $this->eventBus->subscribe(
            ServiceProvisionedEvent::class,
            new ServiceProvisionedObserver(
                App::make(EmailService::class),
                App::make(NotificationService::class)
            )
        );
    }
}
```

## WHMCS Hook Integration

### Hook Handler

```php
<?php
/**
 * Hook handler for WHMCS integration
 */
class WHMCSHookHandler
{
    private HookRegistry $registry;
    private EventBus $eventBus;

    public function __construct(HookRegistry $registry, EventBus $eventBus)
    {
        $this->registry = $registry;
        $this->eventBus = $eventBus;
    }

    /**
     * Register all WHMCS hooks
     */
    public function registerHooks(): void
    {
        // Client hooks
        add_hook('ClientAdd', 1, function ($vars) {
            $this->handleClientAdd($vars);
        });

        add_hook('ClientEdit', 1, function ($vars) {
            $this->handleClientEdit($vars);
        });

        // Order hooks
        add_hook('OrderAccepted', 1, function ($vars) {
            $this->handleOrderAccepted($vars);
        });

        // Invoice hooks
        add_hook('InvoicePaid', 1, function ($vars) {
            $this->handleInvoicePaid($vars);
        });

        // Service hooks
        add_hook('ServiceProvision', 1, function ($vars) {
            $this->handleServiceProvision($vars);
        });
    }

    private function handleClientAdd(array $vars): void
    {
        $event = new ClientCreatedEvent(
            $vars['client_id'],
            $vars
        );

        $this->eventBus->publish($event);
    }

    private function handleClientEdit(array $vars): void
    {
        $event = new ClientUpdatedEvent(
            $vars['client_id'],
            $vars['fields'] ?? [],
            $vars['previous'] ?? []
        );

        $this->eventBus->publish($event);
    }

    private function handleOrderAccepted(array $vars): void
    {
        $event = new OrderPlacedEvent(
            $vars['order_id'],
            $vars['userid'],
            $vars['products'] ?? [],
            $vars['total'] ?? 0
        );

        $this->eventBus->publish($event);
    }

    private function handleInvoicePaid(array $vars): void
    {
        $event = new PaymentReceivedEvent(
            $vars['invoice_id'],
            $vars['user_id'],
            $vars['amount_paid'],
            $vars['payment_method'] ?? 'unknown'
        );

        $this->eventBus->publish($event);
    }

    private function handleServiceProvision(array $vars): void
    {
        $event = new ServiceProvisionedEvent(
            $vars['service_id'],
            $vars['userid'],
            $vars['server'] ?? 'unknown'
        );

        $this->eventBus->publish($event);
    }
}
```

## Observer Priority

### Priority-Based Observer Management

```php
<?php
/**
 * Priority observer implementation
 */
class PriorityObserver implements ObserverInterface
{
    private int $priority;
    private string $name;
    private callable $handler;

    public const PRIORITY_HIGH = 100;
    public const PRIORITY_NORMAL = 0;
    public const PRIORITY_LOW = -100;

    public function __construct(
        string $name,
        callable $handler,
        int $priority = self::PRIORITY_NORMAL
    ) {
        $this->name = $name;
        $this->handler = $handler;
        $this->priority = $priority;
    }

    public function update(Event $event): void
    {
        call_user_func($this->handler, $event);
    }

    public function getPriority(): int
    {
        return $this->priority;
    }

    public function getName(): string
    {
        return $this->name;
    }
}

/**
 * Priority-based event dispatcher
 */
class PriorityEventDispatcher
{
    private array $observers = [];

    public function addObserver(ObserverInterface $observer, int $priority = 0): void
    {
        $this->observers[] = [
            'observer' => $observer,
            'priority' => $priority,
        ];

        usort($this->observers, fn($a, $b) => $b['priority'] - $a['priority']);
    }

    public function dispatch(Event $event): void
    {
        foreach ($this->observers as $entry) {
            $entry['observer']->update($event);
        }
    }
}
```

## Best Practices

1. **Single responsibility** - Each observer handles one concern
2. **Use interfaces** - Depend on abstractions
3. **Handle exceptions** - Don't let one observer break others
4. **Consider priority** - Order matters for some observers
5. **Unsubscribe properly** - Clean up observers when done
6. **Document events** - Make event types clear
7. **Avoid tight coupling** - Observer shouldn't know about subject internals

## Related Patterns

- [Event Sourcing](./event-sourcing.md) - Event-driven observer
- [Hook Reference](./module-hook-reference.md) - WHMCS hooks
- [Service Layer](./service-layer.md) - Observer services