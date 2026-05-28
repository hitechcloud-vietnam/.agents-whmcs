# Event-Driven Architecture

Event-driven architecture enables loosely coupled systems where components communicate through events rather than direct dependencies.

## Event System

### Event Bus Interface

```php
<?php
/**
 * Event bus interface
 */
interface EventBusInterface
{
    /**
     * Publish an event
     */
    public function publish(Event $event): void;

    /**
     * Subscribe to an event
     */
    public function subscribe(string $eventType, callable $handler, int $priority = 0): Subscription;

    /**
     * Unsubscribe from an event
     */
    public function unsubscribe(Subscription $subscription): void;
}
```

### Event Base Class

```php
<?php
/**
 * Base event class
 */
abstract class Event
{
    protected string $eventType;
    protected array $payload;
    protected DateTimeImmutable $occurredAt;
    protected ?DateTimeImmutable $recordedAt = null;

    public function __construct(array $payload = [])
    {
        $this->eventType = static::class;
        $this->payload = $payload;
        $this->occurredAt = new DateTimeImmutable();
    }

    public function getEventType(): string
    {
        return $this->eventType;
    }

    public function getPayload(): array
    {
        return $this->payload;
    }

    public function occurredAt(): DateTimeImmutable
    {
        return $this->occurredAt;
    }

    public function withPayload(array $payload): self
    {
        $clone = clone $this;
        $clone->payload = array_merge($this->payload, $payload);
        return $clone;
    }

    public function toArray(): array
    {
        return [
            'event_type' => $this->eventType,
            'payload' => $this->payload,
            'occurred_at' => $this->occurredAt->format('Y-m-d H:i:s.u'),
        ];
    }
}
```

### Domain Events

```php
<?php
/**
 * Client-related events
 */
class ClientCreatedEvent extends Event
{
    public const NAME = 'client.created';

    public function __construct(int $clientId, array $clientData)
    {
        parent::__construct([
            'client_id' => $clientId,
            'client_data' => $clientData,
        ]);
    }

    public function getClientId(): int
    {
        return $this->payload['client_id'];
    }
}

class ClientUpdatedEvent extends Event
{
    public const NAME = 'client.updated';

    public function __construct(int $clientId, array $changes, array $previousData)
    {
        parent::__construct([
            'client_id' => $clientId,
            'changes' => $changes,
            'previous_data' => $previousData,
        ]);
    }
}

class OrderPlacedEvent extends Event
{
    public const NAME = 'order.placed';

    public function __construct(int $orderId, int $clientId, array $items, float $total)
    {
        parent::__construct([
            'order_id' => $orderId,
            'client_id' => $clientId,
            'items' => $items,
            'total' => $total,
        ]);
    }
}

class PaymentReceivedEvent extends Event
{
    public const NAME = 'payment.received';

    public function __construct(int $invoiceId, int $clientId, float $amount, string $method)
    {
        parent::__construct([
            'invoice_id' => $invoiceId,
            'client_id' => $clientId,
            'amount' => $amount,
            'payment_method' => $method,
        ]);
    }
}

class ServiceProvisionedEvent extends Event
{
    public const NAME = 'service.provisioned';

    public function __construct(int $serviceId, int $clientId, string $server)
    {
        parent::__construct([
            'service_id' => $serviceId,
            'client_id' => $clientId,
            'server' => $server,
        ]);
    }
}
```

## Event Bus Implementation

### Simple Event Bus

```php
<?php
/**
 * Simple in-memory event bus
 */
class SimpleEventBus implements EventBusInterface
{
    private array $handlers = [];
    private array $subscriptions = [];

    public function publish(Event $event): void
    {
        $eventType = $event->getEventType();

        if (!isset($this->handlers[$eventType])) {
            return;
        }

        // Sort handlers by priority
        $handlers = $this->handlers[$eventType];
        usort($handlers, fn($a, $b) => $b['priority'] - $a['priority']);

        foreach ($handlers as $handler) {
            try {
                $handler['callback']($event);

                // Allow handler to stop propagation
                if (isset($handler['stopped']) && $handler['stopped']) {
                    break;
                }
            } catch (\Throwable $e) {
                logActivity("Event handler error: " . $e->getMessage());
                // Continue to next handler
            }
        }
    }

    public function subscribe(string $eventType, callable $handler, int $priority = 0): Subscription
    {
        $subscriptionId = uniqid('sub_', true);

        $this->handlers[$eventType][] = [
            'callback' => $handler,
            'priority' => $priority,
            'subscription_id' => $subscriptionId,
        ];

        // Store subscription reference
        $subscription = new Subscription($subscriptionId, $eventType, $handler);

        return $subscription;
    }

    public function unsubscribe(Subscription $subscription): void
    {
        $eventType = $subscription->getEventType();
        $subscriptionId = $subscription->getId();

        if (!isset($this->handlers[$eventType])) {
            return;
        }

        $this->handlers[$eventType] = array_values(
            array_filter(
                $this->handlers[$eventType],
                fn($h) => $h['subscription_id'] !== $subscriptionId
            )
        );
    }

    public function hasHandlers(string $eventType): bool
    {
        return !empty($this->handlers[$eventType]);
    }
}
```

### Async Event Bus

```php
<?php
/**
 * Event bus with async processing
 */
class AsyncEventBus implements EventBusInterface
{
    private EventBusInterface $syncBus;
    private DatabaseQueue $queue;

    public function __construct(EventBusInterface $syncBus, DatabaseQueue $queue)
    {
        $this->syncBus = $syncBus;
        $this->queue = $queue;
    }

    public function publish(Event $event): void
    {
        // Always publish to sync bus for immediate handlers
        $this->syncBus->publish($event);

        // Queue for async handlers
        $this->queue->later(
            new ProcessEventJob($event),
            0 // Process ASAP
        );
    }

    public function subscribe(string $eventType, callable $handler, int $priority = 0): Subscription
    {
        return $this->syncBus->subscribe($eventType, $handler, $priority);
    }

    public function unsubscribe(Subscription $subscription): void
    {
        $this->syncBus->unsubscribe($subscription);
    }
}

/**
 * Async event processing job
 */
class ProcessEventJob extends AbstractJob
{
    private array $eventData;

    public function __construct(Event $event)
    {
        $this->eventData = $event->toArray();
        parent::__construct();
    }

    public function handle(): void
    {
        // Reconstruct event and publish to async handlers
    }
}
```

## Event Handlers

### Client Event Handlers

```php
<?php
/**
 * Welcome email handler
 */
class SendWelcomeEmailHandler
{
    public function handle(Event $event): void
    {
        if (!$event instanceof ClientCreatedEvent) {
            return;
        }

        $clientData = $event->getPayload()['client_data'];

        sendEmail('Welcome Email', $event->getClientId(), [
            'client_name' => $clientData['firstname'] . ' ' . $clientData['lastname'],
        ]);
    }
}

/**
 * Audit logging handler
 */
class AuditLogHandler
{
    public function handle(Event $event): void
    {
        Capsule::table('mod_audit_log')->insert([
            'event_type' => $event->getEventType(),
            'payload' => json_encode($event->getPayload()),
            'occurred_at' => $event->occurredAt()->format('Y-m-d H:i:s'),
            'ip_address' => $_SERVER['REMOTE_ADDR'] ?? 'unknown',
        ]);
    }
}

/**
 * Cache invalidation handler
 */
class CacheInvalidationHandler
{
    private CacheManagerInterface $cache;

    public function __construct(CacheManagerInterface $cache)
    {
        $this->cache = $cache;
    }

    public function handle(Event $event): void
    {
        $tags = $this->getInvalidationTags($event);

        foreach ($tags as $tag) {
            $this->cache->invalidateTags([$tag]);
        }
    }

    private function getInvalidationTags(Event $event): array
    {
        if ($event instanceof ClientCreatedEvent || $event instanceof ClientUpdatedEvent) {
            return ['clients', 'client_' . $event->getClientId()];
        }

        if ($event instanceof OrderPlacedEvent) {
            return ['orders', 'client_' . $event->getPayload()['client_id']];
        }

        return [];
    }
}
```

## Event Sourcing

### Event Store

```php
<?php
/**
 * Event store for event sourcing
 */
class EventStore
{
    private string $table = 'mod_event_store';

    /**
     * Append events to store
     */
    public function append(Event $event, ?string $aggregateId = null): void
    {
        Capsule::table($this->table)->insert([
            'aggregate_id' => $aggregateId,
            'event_type' => $event->getEventType(),
            'payload' => json_encode($event->getPayload()),
            'occurred_at' => $event->occurredAt()->format('Y-m-d H:i:s.u'),
            'metadata' => json_encode([
                'correlation_id' => $this->getCorrelationId(),
                'causation_id' => $this->getCausationId(),
            ]),
        ]);
    }

    /**
     * Get events for aggregate
     */
    public function getEventsForAggregate(string $aggregateId): array
    {
        $records = Capsule::table($this->table)
            ->where('aggregate_id', $aggregateId)
            ->orderBy('occurred_at', 'asc')
            ->get();

        return array_map(fn($r) => $this->reconstructEvent($r), $records);
    }

    /**
     * Get events by type
     */
    public function getEventsByType(string $eventType, int $limit = 100): array
    {
        $records = Capsule::table($this->table)
            ->where('event_type', $eventType)
            ->orderBy('occurred_at', 'desc')
            ->limit($limit)
            ->get();

        return array_map(fn($r) => $this->reconstructEvent($r), $records);
    }

    /**
     * Reconstruct event from storage
     */
    private function reconstructEvent(stdClass $record): Event
    {
        $class = $record->event_type;
        $payload = json_decode($record->payload, true);

        $event = new $class($payload);

        return $event;
    }

    private function getCorrelationId(): ?string
    {
        return $_SESSION['correlation_id'] ?? null;
    }

    private function getCausationId(): ?string
    {
        return $_SESSION['causation_id'] ?? null;
    }
}
```

### Aggregate Root with Event Sourcing

```php
<?php
/**
 * Aggregate root with event sourcing
 */
abstract class EventSourcedAggregate
{
    protected string $aggregateId;
    private array $pendingEvents = [];

    abstract public function getAggregateId(): string;

    /**
     * Apply event to aggregate
     */
    protected function apply(Event $event): void
    {
        $this->pendingEvents[] = $event;
        $this->handleEvent($event);
    }

    /**
     * Handle specific event
     */
    abstract protected function handleEvent(Event $event): void;

    /**
     * Get pending events
     */
    public function getPendingEvents(): array
    {
        return $this->pendingEvents;
    }

    /**
     * Clear pending events after persistence
     */
    public function clearPendingEvents(): void
    {
        $this->pendingEvents = [];
    }

    /**
     * Reconstitute from events
     */
    public static function reconstitute(string $aggregateId, array $events): self
    {
        $instance = new static();
        $instance->aggregateId = $aggregateId;

        foreach ($events as $event) {
            $instance->handleEvent($event);
        }

        return $instance;
    }
}

/**
 * Order aggregate with event sourcing
 */
class OrderAggregate extends EventSourcedAggregate
{
    private int $clientId;
    private array $items = [];
    private string $status = 'draft';
    private float $total = 0;

    public function getAggregateId(): string
    {
        return $this->aggregateId;
    }

    public static function create(int $clientId): self
    {
        $order = new self();
        $order->aggregateId = 'order_' . uniqid();
        $order->clientId = $clientId;
        $order->apply(new OrderCreatedEvent($order->aggregateId, $clientId));
        return $order;
    }

    public function addItem(array $item): void
    {
        if ($this->status !== 'draft') {
            throw new OrderException('Cannot add items to non-draft order');
        }

        $this->items[] = $item;
        $this->recalculateTotal();
        $this->apply(new OrderItemAddedEvent($this->aggregateId, $item));
    }

    public function place(): void
    {
        if (empty($this->items)) {
            throw new OrderException('Cannot place empty order');
        }

        $this->status = 'placed';
        $this->apply(new OrderPlacedEvent(
            $this->aggregateId,
            $this->clientId,
            $this->items,
            $this->total
        ));
    }

    protected function handleEvent(Event $event): void
    {
        if ($event instanceof OrderCreatedEvent) {
            $this->aggregateId = $event->getPayload()['order_id'];
            $this->clientId = $event->getPayload()['client_id'];
            $this->status = 'draft';
        } elseif ($event instanceof OrderItemAddedEvent) {
            $this->items[] = $event->getPayload()['item'];
            $this->recalculateTotal();
        } elseif ($event instanceof OrderPlacedEvent) {
            $this->status = 'placed';
        }
    }

    private function recalculateTotal(): void
    {
        $this->total = array_sum(array_column($this->items, 'price'));
    }
}
```

## Event Projection

### Read Model Projector

```php
<?php
/**
 * Projector for building read models
 */
class OrderReadModelProjector
{
    private string $readTable = 'mod_order_read_model';

    public function project(ClientCreatedEvent $event): void
    {
        // Project client data
    }

    public function project(OrderPlacedEvent $event): void
    {
        $payload = $event->getPayload();

        Capsule::table($this->readTable)->updateOrInsert(
            ['order_id' => $payload['order_id']],
            [
                'client_id' => $payload['client_id'],
                'total' => $payload['total'],
                'item_count' => count($payload['items']),
                'status' => 'placed',
                'placed_at' => date('Y-m-d H:i:s'),
            ]
        );
    }

    public function project(OrderPaidEvent $event): void
    {
        Capsule::table($this->readTable)
            ->where('order_id', $event->getPayload()['order_id'])
            ->update(['status' => 'paid', 'paid_at' => date('Y-m-d H:i:s')]);
    }
}
```

## Best Practices

1. **Use past tense for events** - `OrderPlacedEvent` not `PlaceOrderEvent`
2. **Make events immutable** - Once occurred, don't modify
3. **Include correlation IDs** - Track event chains
4. **Version event schemas** - Handle evolution gracefully
5. **Use async for side effects** - Prevent slow handlers
6. **Project for reads** - Event store + read models
7. **Handle failures** - Implement dead letter queues

## Related Patterns

- [Queue Processing](./queue-processing.md) - Async event handling
- [Observer Pattern](./observer-pattern.md) - Event subscriptions
- [Service Layer](./service-layer.md) - Event-driven services