# Message Queue Patterns

Message queue patterns enable asynchronous communication between system components, improving scalability and decoupling in WHMCS modules.

## Queue Architecture

### Message Interface

```php
<?php
/**
 * Message interface for queue operations
 */
interface MessageInterface
{
    public function getId(): string;
    public function getType(): string;
    public function getPayload(): array;
    public function getTimestamp(): DateTimeImmutable;
    public function getHeaders(): array;
    public function withHeader(string $key, $value): self;
    public function toArray(): array;
}
```

### Message Implementation

```php
<?php
<?php
/**
 * Base message implementation
 */
abstract class AbstractMessage implements MessageInterface
{
    protected string $id;
    protected array $payload;
    protected DateTimeImmutable $timestamp;
    protected array $headers = [];

    public function __construct(array $payload = [], array $headers = [])
    {
        $this->id = $this->generateId();
        $this->payload = $payload;
        $this->timestamp = new DateTimeImmutable();
        $this->headers = $headers;
    }

    public function getId(): string
    {
        return $this->id;
    }

    abstract public function getType(): string;

    public function getPayload(): array
    {
        return $this->payload;
    }

    public function getTimestamp(): DateTimeImmutable
    {
        return $this->timestamp;
    }

    public function getHeaders(): array
    {
        return $this->headers;
    }

    public function withHeader(string $key, $value): self
    {
        $clone = clone $this;
        $clone->headers[$key] = $value;
        return $clone;
    }

    public function getHeader(string $key, $default = null)
    {
        return $this->headers[$key] ?? $default;
    }

    public function toArray(): array
    {
        return [
            'id' => $this->id,
            'type' => $this->getType(),
            'payload' => $this->payload,
            'timestamp' => $this->timestamp->format('Y-m-d\TH:i:s.uP'),
            'headers' => $this->headers,
        ];
    }

    private function generateId(): string
    {
        return sprintf(
            '%s-%s',
            $this->timestamp->format('YmdHis'),
            bin2hex(random_bytes(8))
        );
    }
}

/**
 * Domain message implementations
 */
class ClientCreatedMessage extends AbstractMessage
{
    public function getType(): string
    {
        return 'client.created';
    }

    public static function fromPayload(int $clientId, array $data): self
    {
        return new self([
            'client_id' => $clientId,
            'client_data' => $data,
        ]);
    }
}

class OrderPlacedMessage extends AbstractMessage
{
    public function getType(): string
    {
        return 'order.placed';
    }

    public static function fromPayload(int $orderId, int $clientId, array $items, float $total): self
    {
        return new self([
            'order_id' => $orderId,
            'client_id' => $clientId,
            'items' => $items,
            'total' => $total,
        ]);
    }
}

class PaymentProcessedMessage extends AbstractMessage
{
    public function getType(): string
    {
        return 'payment.processed';
    }

    public static function fromPayload(int $invoiceId, float $amount, string $method): self
    {
        return new self([
            'invoice_id' => $invoiceId,
            'amount' => $amount,
            'payment_method' => $method,
        ]);
    }
}

class ServiceProvisionedMessage extends AbstractMessage
{
    public function getType(): string
    {
        return 'service.provisioned';
    }

    public static function fromPayload(int $serviceId, int $clientId, string $server): self
    {
        return new self([
            'service_id' => $serviceId,
            'client_id' => $clientId,
            'server' => $server,
        ]);
    }
}
```

## Queue Producer

### Message Producer

```php
<?php
/**
 * Message queue producer
 */
class MessageProducer
{
    private string $table = 'mod_message_queue';
    private string $exchange = 'default';
    private array $exchanges = [];

    /**
     * Publish a message
     */
    public function publish(MessageInterface $message, string $routingKey = ''): string
    {
        $messageData = $message->toArray();
        $messageData['exchange'] = $this->exchange;
        $messageData['routing_key'] = $routingKey;
        $messageData['status'] = 'pending';

        $id = Capsule::table($this->table)->insertGetId($messageData);

        return $message->getId();
    }

    /**
     * Publish multiple messages
     */
    public function publishBatch(array $messages, string $routingKey = ''): array
    {
        $ids = [];

        foreach ($messages as $message) {
            $ids[] = $this->publish($message, $routingKey);
        }

        return $ids;
    }

    /**
     * Publish with delay
     */
    public function publishDelayed(MessageInterface $message, int $delaySeconds, string $routingKey = ''): string
    {
        $messageData = $message->toArray();
        $messageData['exchange'] = $this->exchange;
        $messageData['routing_key'] = $routingKey;
        $messageData['status'] = 'delayed';
        $messageData['available_at'] = date('Y-m-d H:i:s', time() + $delaySeconds);

        $id = Capsule::table($this->table)->insertGetId($messageData);

        return $message->getId();
    }

    /**
     * Declare an exchange
     */
    public function declareExchange(string $name, string $type = 'direct'): void
    {
        $this->exchanges[$name] = [
            'name' => $name,
            'type' => $type,
        ];
    }

    /**
     * Set current exchange
     */
    public function setExchange(string $name): self
    {
        $this->exchange = $name;
        return $this;
    }
}
```

## Queue Consumer

### Message Consumer

```php
<?php
/**
 * Message queue consumer
 */
class MessageConsumer
{
    private string $table = 'mod_message_queue';
    private array $handlers = [];
    private int $maxRetries = 3;
    private bool $running = false;

    /**
     * Register a handler for a message type
     */
    public function handle(string $messageType, callable $handler): self
    {
        $this->handlers[$messageType] = $handler;
        return $this;
    }

    /**
     * Start consuming messages
     */
    public function consume(int $limit = 100, int $timeout = 30): int
    {
        $this->running = true;
        $processed = 0;
        $startTime = time();

        while ($this->running && $processed < $limit) {
            if ((time() - $startTime) >= $timeout) {
                break;
            }

            $message = $this->fetchMessage();

            if ($message === null) {
                usleep(100000); // 100ms
                continue;
            }

            $this->processMessage($message);
            $processed++;
        }

        return $processed;
    }

    /**
     * Stop consuming
     */
    public function stop(): void
    {
        $this->running = false;
    }

    /**
     * Fetch next available message
     */
    private function fetchMessage(): ?array
    {
        $message = Capsule::table($this->table)
            ->where('status', 'pending')
            ->where('available_at', '<=', date('Y-m-d H:i:s'))
            ->orderBy('created_at', 'asc')
            ->first();

        if (!$message) {
            return null;
        }

        // Mark as processing
        Capsule::table($this->table)
            ->where('id', $message->id)
            ->update([
                'status' => 'processing',
                'started_at' => date('Y-m-d H:i:s'),
            ]);

        return (array) $message;
    }

    /**
     * Process a single message
     */
    private function processMessage(array $messageData): void
    {
        $messageId = $messageData['id'];
        $messageType = $messageData['type'];
        $payload = json_decode($messageData['payload'], true);
        $retries = (int) $messageData['retries'];

        if (!isset($this->handlers[$messageType])) {
            // No handler for this message type
            $this->acknowledge($messageId, 'no_handler');
            return;
        }

        try {
            $handler = $this->handlers[$messageType];
            $result = $handler($payload, $messageData['headers']);

            if ($result === false) {
                throw new Exception('Handler returned false');
            }

            $this->acknowledge($messageId, 'success');
        } catch (\Throwable $e) {
            $this->handleFailure($messageId, $messageData, $e, $retries);
        }
    }

    /**
     * Acknowledge message completion
     */
    private function acknowledge(int $messageId, string $status): void
    {
        Capsule::table($this->table)
            ->where('id', $messageId)
            ->update([
                'status' => 'completed',
                'completed_at' => date('Y-m-d H:i:s'),
                'result' => $status,
            ]);
    }

    /**
     * Handle message processing failure
     */
    private function handleFailure(int $messageId, array $messageData, \Throwable $e, int $retries): void
    {
        if ($retries < $this->maxRetries) {
            // Retry with exponential backoff
            $delay = pow(2, $retries) * 60; // 1min, 2min, 4min

            Capsule::table($this->table)
                ->where('id', $messageId)
                ->update([
                    'status' => 'pending',
                    'retries' => $retries + 1,
                    'available_at' => date('Y-m-d H:i:s', time() + $delay),
                    'last_error' => $e->getMessage(),
                ]);
        } else {
            // Move to dead letter queue
            Capsule::table($this->table)
                ->where('id', $messageId)
                ->update([
                    'status' => 'failed',
                    'failed_at' => date('Y-m-d H:i:s'),
                    'last_error' => $e->getMessage(),
                ]);
        }

        logActivity("Message {$messageId} failed: " . $e->getMessage());
    }
}
```

## Queue Manager

### Queue Administration

```php
<?php
<?php
/**
 * Queue administration and monitoring
 */
class QueueManager
{
    private string $table = 'mod_message_queue';
    private string $dlqTable = 'mod_message_queue_dlq';

    /**
     * Get queue statistics
     */
    public function getStats(): array
    {
        $pending = Capsule::table($this->table)->where('status', 'pending')->count();
        $processing = Capsule::table($this->table)->where('status', 'processing')->count();
        $completed = Capsule::table($this->table)->where('status', 'completed')->count();
        $failed = Capsule::table($this->table)->where('status', 'failed')->count();

        $oldestPending = Capsule::table($this->table)
            ->where('status', 'pending')
            ->orderBy('created_at', 'asc')
            ->first();

        return [
            'pending' => $pending,
            'processing' => $processing,
            'completed' => $completed,
            'failed' => $failed,
            'total' => $pending + $processing + $completed + $failed,
            'oldest_pending_age' => $oldestPending
                ? time() - strtotime($oldestPending->created_at)
                : null,
        ];
    }

    /**
     * Get message types breakdown
     */
    public function getMessageTypes(): array
    {
        $results = Capsule::table($this->table)
            ->select('type', 'status', Capsule::raw('COUNT(*) as count'))
            ->groupBy('type', 'status')
            ->get();

        $types = [];

        foreach ($results as $row) {
            if (!isset($types[$row->type])) {
                $types[$row->type] = [
                    'pending' => 0,
                    'processing' => 0,
                    'completed' => 0,
                    'failed' => 0,
                ];
            }
            $types[$row->type][$row->status] = (int) $row->count;
        }

        return $types;
    }

    /**
     * Retry failed messages
     */
    public function retryFailed(int $limit = 100): int
    {
        return Capsule::table($this->table)
            ->where('status', 'failed')
            ->limit($limit)
            ->update([
                'status' => 'pending',
                'retries' => 0,
                'available_at' => date('Y-m-d H:i:s'),
                'last_error' => null,
            ]);
    }

    /**
     * Retry specific message
     */
    public function retryMessage(int $messageId): bool
    {
        $affected = Capsule::table($this->table)
            ->where('id', $messageId)
            ->where('status', 'failed')
            ->update([
                'status' => 'pending',
                'retries' => 0,
                'available_at' => date('Y-m-d H:i:s'),
            ]);

        return $affected > 0;
    }

    /**
     * Purge completed messages older than specified days
     */
    public function purgeCompleted(int $daysOld = 7): int
    {
        $cutoff = date('Y-m-d H:i:s', strtotime("-{$daysOld} days"));

        return Capsule::table($this->table)
            ->where('status', 'completed')
            ->where('completed_at', '<', $cutoff)
            ->delete();
    }

    /**
     * Move failed to dead letter queue
     */
    public function moveToDeadLetter(int $messageId): bool
    {
        $message = Capsule::table($this->table)
            ->where('id', $messageId)
            ->first();

        if (!$message) {
            return false;
        }

        Capsule::table($this->dlqTable)->insert([
            'original_id' => $message->id,
            'type' => $message->type,
            'payload' => $message->payload,
            'headers' => $message->headers,
            'error' => $message->last_error,
            'failed_at' => $message->failed_at,
            'original_queue' => $message->exchange,
        ]);

        Capsule::table($this->table)->where('id', $messageId)->delete();

        return true;
    }

    /**
     * Get failed messages
     */
    public function getFailedMessages(int $limit = 50): array
    {
        return Capsule::table($this->table)
            ->where('status', 'failed')
            ->orderBy('failed_at', 'desc')
            ->limit($limit)
            ->get();
    }

    /**
     * Get dead letter queue messages
     */
    public function getDeadLetterMessages(int $limit = 50): array
    {
        return Capsule::table($this->dlqTable)
            ->orderBy('failed_at', 'desc')
            ->limit($limit)
            ->get();
    }
}
```

## WHMCS Integration

### Queue Service Provider

```php
<?php
/**
 * Queue service provider for WHMCS
 */
class QueueServiceProvider
{
    /**
     * Register queue services
     */
    public static function register(): void
    {
        // Create tables if not exist
        self::createTables();

        // Register message handlers
        add_hook('AfterModuleCreate', 1, function ($vars) {
            $producer = new MessageProducer();
            $producer->publish(new ServiceProvisionedMessage(
                $vars['serviceid'],
                $vars['userid'],
                $vars['server']['hostname'] ?? ''
            ));
        });

        add_hook('InvoicePaid', 1, function ($vars) {
            $invoice = Capsule::table('tblinvoices')->where('id', $vars['invoice_id'])->first();

            if ($invoice) {
                $producer = new MessageProducer();
                $producer->publish(new PaymentProcessedMessage(
                    $vars['invoice_id'],
                    $invoice->total,
                    $vars['payment_method'] ?? 'unknown'
                ));
            }
        });
    }

    /**
     * Create queue tables
     */
    private static function createTables(): void
    {
        if (!Capsule::schema()->hasTable('mod_message_queue')) {
            Capsule::schema()->create('mod_message_queue', function ($t) {
                $t->increments('id');
                $t->string('message_id', 64)->unique();
                $t->string('type', 100)->index();
                $t->text('payload');
                $t->text('headers')->nullable();
                $t->string('exchange', 100)->default('default');
                $t->string('routing_key', 255)->nullable();
                $t->enum('status', ['pending', 'processing', 'completed', 'failed', 'delayed'])->default('pending');
                $t->timestamp('available_at');
                $t->timestamp('created_at');
                $t->timestamp('started_at')->nullable();
                $t->timestamp('completed_at')->nullable();
                $t->timestamp('failed_at')->nullable();
                $t->integer('retries')->default(0);
                $t->string('last_error', 500)->nullable();
                $t->string('result', 100)->nullable();
            });
        }

        if (!Capsule::schema()->hasTable('mod_message_queue_dlq')) {
            Capsule::schema()->create('mod_message_queue_dlq', function ($t) {
                $t->increments('id');
                $t->string('original_id', 64);
                $t->string('type', 100);
                $t->text('payload');
                $t->text('headers')->nullable();
                $t->string('error', 500)->nullable();
                $t->timestamp('failed_at');
                $t->string('original_queue', 100)->nullable();
                $t->timestamp('created_at');
            });
        }
    }
}

/**
 * Queue consumer command
 */
function runQueueConsumer_cli($args)
{
    $limit = (int) ($args[0] ?? 100);
    $timeout = (int) ($args[1] ?? 30);

    echo "Starting queue consumer...\n";

    $consumer = new MessageConsumer();

    // Register handlers
    $consumer->handle('client.created', function ($payload, $headers) {
        // Send welcome email
        $clientId = $payload['client_id'];
        $clientData = $payload['client_data'];

        sendEmail('Welcome Email', $clientId, [
            'client_name' => $clientData['firstname'] . ' ' . $clientData['lastname'],
        ]);

        return true;
    });

    $consumer->handle('service.provisioned', function ($payload, $headers) {
        // Send provisioning notification
        $serviceId = $payload['service_id'];

        Capsule::table('tblhosting')
            ->where('id', $serviceId)
            ->update(['notes' => 'Provisioned at ' . date('Y-m-d H:i:s')]);

        return true;
    });

    $processed = $consumer->consume($limit, $timeout);

    echo "Processed {$processed} messages.\n";
}
```

## Best Practices

1. **Use typed messages** - Define message classes for each type
2. **Make messages idempotent** - Safe to process multiple times
3. **Implement retry logic** - With exponential backoff
4. **Use dead letter queues** - Capture failed messages for analysis
5. **Monitor queue depth** - Alert when backlogs grow
6. **Keep messages small** - Store large data as references
7. **Set appropriate timeouts** - Prevent stuck processing
8. **Clean up old messages** - Prune completed messages regularly

## Related Patterns

- [Event Sourcing](./event-sourcing.md) - Event-driven messaging
- [Queue Processing](./queue-processing.md) - Background jobs
- [Observer Pattern](./observer-pattern.md) - Event notifications
