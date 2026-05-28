# Distributed Tracing Implementation

Distributed tracing tracks requests as they flow through multiple services, enabling end-to-end visibility into complex WHMCS integrations and external service calls.

## Tracing Architecture

### Trace Context

```php
<?php
/**
 * Trace context for distributed tracing
 */
class TraceContext
{
    private string $traceId;
    private string $spanId;
    private ?string $parentSpanId;
    private array $baggage = [];
    private float $startTime;

    public function __construct(
        ?string $traceId = null,
        ?string $spanId = null,
        ?string $parentSpanId = null
    ) {
        $this->traceId = $traceId ?? $this->generateId(32);
        $this->spanId = $spanId ?? $this->generateId(16);
        $this->parentSpanId = $parentSpanId;
        $this->startTime = microtime(true);
    }

    public function getTraceId(): string
    {
        return $this->traceId;
    }

    public function getSpanId(): string
    {
        return $this->spanId;
    }

    public function getParentSpanId(): ?string
    {
        return $this->parentSpanId;
    }

    public function getBaggage(string $key, $default = null)
    {
        return $this->baggage[$key] ?? $default;
    }

    public function withBaggage(string $key, $value): self
    {
        $clone = clone $this;
        $clone->baggage[$key] = $value;
        return $clone;
    }

    public function getDuration(): float
    {
        return microtime(true) - $this->startTime;
    }

    public function toArray(): array
    {
        return [
            'trace_id' => $this->traceId,
            'span_id' => $this->spanId,
            'parent_span_id' => $this->parentSpanId,
            'baggage' => $this->baggage,
            'start_time' => $this->startTime,
        ];
    }

    private function generateId(int $length): string
    {
        return bin2hex(random_bytes($length / 2));
    }

    /**
     * Extract from HTTP headers (W3C Trace Context format)
     */
    public static function fromHeaders(array $headers): self
    {
        $traceparent = $headers['traceparent'] ?? $headers['TRACEPARENT'] ?? null;

        if ($traceparent && preg_match('/^([0-9a-f]{32})-([0-9a-f]{16})-([0-9a-f]{16})$/', $traceparent, $matches)) {
            return new self($matches[1], $matches[2], $matches[3]);
        }

        return new self();
    }

    public function toHeaders(): array
    {
        return [
            'traceparent' => "{$this->traceId}-{$this->spanId}-01",
            'tracestate' => $this->buildTraceState(),
        ];
    }

    private function buildTraceState(): string
    {
        $parts = [];
        foreach ($this->baggage as $key => $value) {
            $parts[] = "{$key}=" . urlencode((string) $value);
        }
        return implode(',', $parts);
    }
}
```

### Span Interface

```php
<?php
/**
 * Span interface for tracing operations
 */
interface SpanInterface
{
    public function getContext(): TraceContext;
    public function setName(string $name): void;
    public function setAttribute(string $key, $value): void;
    public function addEvent(string $name, array $attributes = []): void;
    public function setStatus(string $code, ?string $description = null): void;
    public function end(): void;
    public function recordException(\Throwable $exception): void;
}
```

### Span Implementation

```php
<?php
/**
 * Span implementation
 */
class Span implements SpanInterface
{
    private TraceContext $context;
    private string $name;
    private array $attributes = [];
    private array $events = [];
    private string $statusCode = 'UNSET';
    private ?string $statusDescription = null;
    private float $startTime;
    private ?float $endTime = null;
    private ?Span $parent = null;

    public function __construct(TraceContext $context, string $name, ?Span $parent = null)
    {
        $this->context = $context;
        $this->name = $name;
        $this->parent = $parent;
        $this->startTime = microtime(true);
    }

    public function getContext(): TraceContext
    {
        return $this->context;
    }

    public function setName(string $name): void
    {
        $this->name = $name;
    }

    public function setAttribute(string $key, $value): void
    {
        $this->attributes[$key] = $this->normalizeValue($value);
    }

    public function addEvent(string $name, array $attributes = []): void
    {
        $this->events[] = [
            'name' => $name,
            'timestamp' => microtime(true),
            'attributes' => array_map([$this, 'normalizeValue'], $attributes),
        ];
    }

    public function setStatus(string $code, ?string $description = null): void
    {
        $validCodes = ['UNSET', 'OK', 'ERROR'];

        if (!in_array($code, $validCodes)) {
            throw new InvalidArgumentException("Invalid status code: {$code}");
        }

        $this->statusCode = $code;
        $this->statusDescription = $description;
    }

    public function end(): void
    {
        if ($this->endTime !== null) {
            return; // Already ended
        }

        $this->endTime = microtime(true);
        $this->record();
    }

    public function recordException(\Throwable $exception): void
    {
        $this->addEvent('exception', [
            'exception.type' => get_class($exception),
            'exception.message' => $exception->getMessage(),
            'exception.stacktrace' => $exception->getTraceAsString(),
        ]);

        $this->setStatus('ERROR', $exception->getMessage());
    }

    public function isEnded(): bool
    {
        return $this->endTime !== null;
    }

    public function getDuration(): float
    {
        if ($this->endTime === null) {
            return microtime(true) - $this->startTime;
        }

        return $this->endTime - $this->startTime;
    }

    private function record(): void
    {
        // Record to the tracer
        $tracer = Tracer::getInstance();
        $tracer->recordSpan($this);
    }

    private function normalizeValue($value)
    {
        if (is_object($value) && method_exists($value, '__toString')) {
            return (string) $value;
        }

        if (is_scalar($value) || is_array($value)) {
            return $value;
        }

        return json_encode($value);
    }

    public function toArray(): array
    {
        return [
            'context' => $this->context->toArray(),
            'name' => $this->name,
            'attributes' => $this->attributes,
            'events' => $this->events,
            'status' => [
                'code' => $this->statusCode,
                'description' => $this->statusDescription,
            ],
            'start_time' => $this->startTime,
            'end_time' => $this->endTime,
            'duration_ms' => $this->getDuration() * 1000,
        ];
    }
}
```

## Tracer Implementation

### Tracer Service

```php
<?php
/**
 * Tracer service for distributed tracing
 */
class Tracer
{
    private static ?Tracer $instance = null;
    private array $spans = [];
    private array $exporters = [];
    private bool $enabled = true;
    private array $samplerConfig;

    private function __construct()
    {
        $this->samplerConfig = [
            'type' => 'always_on',
            'sample_rate' => 1.0,
        ];
    }

    public static function getInstance(): self
    {
        if (self::$instance === null) {
            self::$instance = new self();
        }

        return self::$instance;
    }

    /**
     * Start a new span
     */
    public function startSpan(string $name, ?Span $parent = null): Span
    {
        $context = $this->sampleContext($parent);

        if (!$this->shouldSample($context)) {
            return new NoOpSpan($context);
        }

        $span = new Span($context, $name, $parent);
        $this->spans[$context->getSpanId()] = $span;

        return $span;
    }

    /**
     * Start span from incoming request
     */
    public function startSpanFromRequest(string $name, array $headers = []): Span
    {
        $context = TraceContext::fromHeaders($headers);
        $span = $this->startSpan($name);

        return $span;
    }

    /**
     * Inject context into HTTP headers
     */
    public function injectContext(TraceContext $context, array $headers = []): array
    {
        $injected = $context->toHeaders();

        return array_merge($headers, $injected);
    }

    /**
     * Add span exporter
     */
    public function addExporter(SpanExporterInterface $exporter): void
    {
        $this->exporters[] = $exporter;
    }

    /**
     * Record completed span
     */
    public function recordSpan(Span $span): void
    {
        if (!$this->enabled) {
            return;
        }

        foreach ($this->exporters as $exporter) {
            try {
                $exporter->export([$span->toArray()]);
            } catch (\Throwable $e) {
                logActivity("Span export failed: " . $e->getMessage());
            }
        }
    }

    /**
     * Flush all pending spans
     */
    public function flush(): void
    {
        foreach ($this->exporters as $exporter) {
            if (method_exists($exporter, 'flush')) {
                $exporter->flush();
            }
        }
    }

    /**
     * Enable/disable tracing
     */
    public function setEnabled(bool $enabled): void
    {
        $this->enabled = $enabled;
    }

    private function sampleContext(?Span $parent): TraceContext
    {
        $parentContext = $parent?->getContext();

        return new TraceContext(
            $parentContext?->getTraceId(),
            null, // New span ID
            $parentContext?->getSpanId()
        );
    }

    private function shouldSample(TraceContext $context): bool
    {
        $type = $this->samplerConfig['type'];

        switch ($type) {
            case 'always_on':
                return true;

            case 'always_off':
                return false;

            case 'trace_id_ratio':
                $hash = crc32($context->getTraceId());
                return ($hash / PHP_INT_MAX) < $this->samplerConfig['sample_rate'];

            default:
                return true;
        }
    }

    public function setSamplerConfig(array $config): void
    {
        $this->samplerConfig = array_merge($this->samplerConfig, $config);
    }
}
```

## Span Exporters

### Database Exporter

```php
<?php
/**
 * Database span exporter
 */
class DatabaseSpanExporter implements SpanExporterInterface
{
    private string $table = 'mod_trace_spans';

    public function export(array $spans): void
    {
        foreach ($spans as $span) {
            $this->insertSpan($span);
        }
    }

    private function insertSpan(array $span): void
    {
        Capsule::table($this->table)->insert([
            'trace_id' => $span['context']['trace_id'],
            'span_id' => $span['context']['span_id'],
            'parent_span_id' => $span['context']['parent_span_id'],
            'name' => $span['name'],
            'attributes' => json_encode($span['attributes']),
            'events' => json_encode($span['events']),
            'status_code' => $span['status']['code'],
            'status_description' => $span['status']['description'],
            'start_time' => date('Y-m-d H:i:s', (int) $span['start_time']),
            'end_time' => $span['end_time'] ? date('Y-m-d H:i:s', (int) $span['end_time']) : null,
            'duration_ms' => $span['duration_ms'],
            'created_at' => date('Y-m-d H:i:s'),
        ]);
    }

    public function flush(): void
    {
        // No-op for database exporter
    }

    public static function createTable(): void
    {
        Capsule::schema()->create($this->table ?? 'mod_trace_spans', function ($t) {
            $t->increments('id');
            $t->string('trace_id', 32)->index();
            $t->string('span_id', 16)->index();
            $t->string('parent_span_id', 16)->nullable();
            $t->string('name', 255);
            $t->text('attributes')->nullable();
            $t->text('events')->nullable();
            $t->string('status_code', 20);
            $t->string('status_description', 500)->nullable();
            $t->timestamp('start_time');
            $t->timestamp('end_time')->nullable();
            $t->decimal('duration_ms', 10, 3);
            $t->timestamp('created_at');
        });
    }
}
```

### File Exporter

```php
<?php
/**
 * File-based span exporter (for development/debugging)
 */
class FileSpanExporter implements SpanExporterInterface
{
    private string $logPath;
    private array $buffer = [];
    private int $flushThreshold = 100;

    public function __construct(string $logPath = null)
    {
        $this->logPath = $logPath ?? __DIR__ . '/../../logs/traces.log';
    }

    public function export(array $spans): void
    {
        foreach ($spans as $span) {
            $this->buffer[] = json_encode([
                'timestamp' => date('Y-m-d H:i:s.u'),
                'span' => $span,
            ]);
        }

        if (count($this->buffer) >= $this->flushThreshold) {
            $this->flush();
        }
    }

    public function flush(): void
    {
        if (empty($this->buffer)) {
            return;
        }

        $content = implode("\n", $this->buffer) . "\n";
        file_put_contents($this->logPath, $content, FILE_APPEND | LOCK_EX);

        $this->buffer = [];
    }

    public function __destruct()
    {
        $this->flush();
    }
}
```

## Tracing Integration

### WHMCS Hook Integration

```php
<?php
/**
 * Tracing hook for WHMCS requests
 */
add_hook('preAutoload', 1, function ($vars) {
    $tracer = Tracer::getInstance();

    // Add database exporter
    $tracer->addExporter(new DatabaseSpanExporter());

    // Add file exporter for debugging
    if (App::isDebugMode()) {
        $tracer->addExporter(new FileSpanExporter());
    }
});

/**
 * Trace API calls
 */
add_hook('ApiStart', 1, function ($vars) {
    $tracer = Tracer::getInstance();

    $span = $tracer->startSpan('api.' . ($vars['action'] ?? 'unknown'));
    $span->setAttribute('api.action', $vars['action'] ?? 'unknown');
    $span->setAttribute('api.controller', $vars['controller'] ?? 'unknown');

    // Store span in request for later reference
    $_SESSION['current_span'] = $span;
});

add_hook('ApiComplete', 1, function ($vars) {
    $span = $_SESSION['current_span'] ?? null;

    if ($span instanceof Span) {
        $span->setAttribute('api.success', $vars['success'] ?? false);
        $span->setAttribute('api.response_time_ms', $vars['response_time'] ?? 0);
        $span->end();
    }
});
```

### External API Tracing

```php
<?php
/**
 * Traced HTTP client
 */
class TracedHttpClient
{
    private Tracer $tracer;

    public function __construct(Tracer $tracer)
    {
        $this->tracer = $tracer;
    }

    public function request(
        string $method,
        string $url,
        array $options = [],
        ?Span $parentSpan = null
    ): array {
        $span = $this->tracer->startSpan("http.{$method}", $parentSpan);

        try {
            $span->setAttribute('http.method', $method);
            $span->setAttribute('http.url', $url);

            // Inject trace context into headers
            $headers = $options['headers'] ?? [];
            $context = $parentSpan?->getContext() ?? new TraceContext();
            $headers = $this->tracer->injectContext($context, $headers);

            $response = $this->executeRequest($method, $url, array_merge($options, [
                'headers' => $headers,
            ]));

            $span->setAttribute('http.status_code', $response['status_code']);
            $span->setStatus('OK');

            return $response;
        } catch (\Throwable $e) {
            $span->recordException($e);
            throw $e;
        } finally {
            $span->end();
        }
    }

    private function executeRequest(string $method, string $url, array $options): array
    {
        $ch = curl_init();

        curl_setopt_array($ch, [
            CURLOPT_URL => $url,
            CURLOPT_RETURNTRANSFER => true,
            CURLOPT_TIMEOUT => $options['timeout'] ?? 30,
            CURLOPT_CUSTOMREQUEST => $method,
        ]);

        if (!empty($options['headers'])) {
            $curlHeaders = [];
            foreach ($options['headers'] as $key => $value) {
                $curlHeaders[] = "{$key}: {$value}";
            }
            curl_setopt($ch, CURLOPT_HTTPHEADER, $curlHeaders);
        }

        if (!empty($options['body'])) {
            curl_setopt($ch, CURLOPT_POSTFIELDS, json_encode($options['body']));
        }

        $responseBody = curl_exec($ch);
        $statusCode = curl_getinfo($ch, CURLINFO_HTTP_CODE);
        $error = curl_error($ch);

        curl_close($ch);

        if ($error) {
            throw new HttpException("cURL error: {$error}");
        }

        return [
            'status_code' => $statusCode,
            'body' => json_decode($responseBody, true) ?? $responseBody,
        ];
    }
}
```

## Trace Analysis

### Trace Query

```php
<?php
/**
 * Trace query service
 */
class TraceQuery
{
    private string $table = 'mod_trace_spans';

    /**
     * Get trace by ID
     */
    public function getTrace(string $traceId): array
    {
        $spans = Capsule::table($this->table)
            ->where('trace_id', $traceId)
            ->orderBy('start_time', 'asc')
            ->get();

        return array_map(fn($s) => $this->formatSpan($s), $spans);
    }

    /**
     * Get slow spans
     */
    public function getSlowSpans(float $thresholdMs = 1000, int $limit = 50): array
    {
        $spans = Capsule::table($this->table)
            ->where('duration_ms', '>', $thresholdMs)
            ->orderBy('duration_ms', 'desc')
            ->limit($limit)
            ->get();

        return array_map(fn($s) => $this->formatSpan($s), $spans);
    }

    /**
     * Get error spans
     */
    public function getErrorSpans(int $limit = 50): array
    {
        $spans = Capsule::table($this->table)
            ->where('status_code', 'ERROR')
            ->orderBy('start_time', 'desc')
            ->limit($limit)
            ->get();

        return array_map(fn($s) => $this->formatSpan($s), $spans);
    }

    /**
     * Get service dependency graph
     */
    public function getDependencyGraph(): array
    {
        $spans = Capsule::table($this->table)
            ->select('name', 'parent_span_id')
            ->get();

        $nodes = [];
        $edges = [];

        foreach ($spans as $span) {
            $nodes[$span->name] = true;

            if ($span->parent_span_id) {
                $parentSpan = Capsule::table($this->table)
                    ->where('span_id', $span->parent_span_id)
                    ->first();

                if ($parentSpan) {
                    $edges[] = [
                        'from' => $parentSpan->name,
                        'to' => $span->name,
                    ];
                }
            }
        }

        return [
            'nodes' => array_keys($nodes),
            'edges' => $edges,
        ];
    }

    private function formatSpan(stdClass $row): array
    {
        return [
            'trace_id' => $row->trace_id,
            'span_id' => $row->span_id,
            'parent_span_id' => $row->parent_span_id,
            'name' => $row->name,
            'attributes' => json_decode($row->attributes ?? '{}', true),
            'events' => json_decode($row->events ?? '[]', true),
            'status' => [
                'code' => $row->status_code,
                'description' => $row->status_description,
            ],
            'start_time' => $row->start_time,
            'end_time' => $row->end_time,
            'duration_ms' => (float) $row->duration_ms,
        ];
    }
}
```

## Best Practices

1. **Sample strategically** - Use trace_id_ratio sampling in production
2. **Propagate context** - Always inject trace headers in external calls
3. **Add meaningful attributes** - Include service name, operation type, user ID
4. **Record exceptions** - Use recordException() for all caught errors
5. **End spans properly** - Always call end() or use try-finally
6. **Use NoOpSpan** - Avoid null checks with a no-op implementation
7. **Flush on shutdown** - Ensure spans are exported before request ends
8. **Index trace IDs** - Enable fast lookups by trace_id

## Related Patterns

- [Queue Processing](./queue-processing.md) - Async span recording
- [Event Sourcing](./event-sourcing.md) - Event correlation with traces
- [Service Layer](./service-layer.md) - Tracing service boundaries
