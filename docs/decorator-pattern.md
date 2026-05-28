# Decorator Pattern in WHMCS

The Decorator Pattern allows behavior to be added to an individual object, dynamically, without affecting the behavior of other objects from the same class. In WHMCS, this pattern is excellent for adding features like logging, caching, validation, or transformation to existing services.

## Overview

Decorators wrap the original object and add new behavior before, after, or around the original method calls:
- Add responsibilities dynamically
- Extend functionality without subclassing
- Compose behaviors in flexible combinations
- Single Responsibility: each decorator does one thing

## Core Structure

### Base Decorator

```php
<?php
// includes/Decorators/Decorator.php

namespace CustomModule\Decorators;

abstract class Decorator
{
    protected object $wrapped;

    public function __construct(object $wrapped)
    {
        $this->wrapped = $wrapped;
    }

    public function __call(string $method, array $args): mixed
    {
        return call_user_func_array([$this->wrapped, $method], $args);
    }
}
```

## Real-World WHMCS Examples

### Caching Decorator

```php
<?php
// includes/Decorators/CachingDecorator.php

namespace CustomModule\Decorators;

use WHMCS\Cache\CacheManager;

class CachingDecorator extends Decorator
{
    protected string $cachePrefix;
    protected int $ttlSeconds;
    protected CacheManager $cache;

    public function __construct(object $wrapped, string $cachePrefix, int $ttlSeconds = 300)
    {
        parent::__construct($wrapped);
        $this->cachePrefix = $cachePrefix;
        $this->ttlSeconds = $ttlSeconds;
        $this->cache = CacheManager::driver('file');
    }

    public function __call(string $method, array $args): mixed
    {
        // Only cache read methods
        if (!in_array($method, ['getClient', 'getService', 'getDomain'])) {
            return parent::__call($method, $args);
        }

        $cacheKey = $this->buildCacheKey($method, $args);

        $cached = $this->cache->get($cacheKey);
        if ($cached !== null) {
            return $cached;
        }

        $result = parent::__call($method, $args);
        $this->cache->put($cacheKey, $result, $this->ttlSeconds);

        return $result;
    }

    public function invalidate(array $tags = []): void
    {
        // Clear cache for specific tags
        foreach ($tags as $tag) {
            $this->cache->forget("{$this->cachePrefix}:{$tag}");
        }
    }

    protected function buildCacheKey(string $method, array $args): string
    {
        return "{$this->cachePrefix}:{$method}:" . md5(serialize($args));
    }
}
```

### Logging Decorator

```php
<?php
// includes/Decorators/LoggingDecorator.php

namespace CustomModule\Decorators;

use Psr\Log\LoggerInterface;

class LoggingDecorator extends Decorator
{
    protected LoggerInterface $logger;
    protected array $methodsToLog;
    protected bool $logArguments;
    protected bool $logResult;

    public function __construct(
        object $wrapped,
        LoggerInterface $logger,
        array $methodsToLog = [],
        bool $logArguments = true,
        bool $logResult = false
    ) {
        parent::__construct($wrapped);
        $this->logger = $logger;
        $this->methodsToLog = $methodsToLog;
        $this->logArguments = $logArguments;
        $this->logResult = $logResult;
    }

    public function __call(string $method, array $args): mixed
    {
        // Check if method should be logged
        if (!empty($this->methodsToLog) && !in_array($method, $this->methodsToLog)) {
            return parent::__call($method, $args);
        }

        $context = [
            'method' => $method,
            'class' => get_class($this->wrapped)
        ];

        if ($this->logArguments) {
            $context['arguments'] = $this->sanitizeArgs($args);
        }

        $this->logger->info("Calling {$method}", $context);

        $startTime = microtime(true);

        try {
            $result = parent::__call($method, $args);

            $duration = microtime(true) - $startTime;
            $context['duration_ms'] = round($duration * 1000, 2);

            if ($this->logResult) {
                $context['result'] = $this->sanitizeResult($result);
            }

            $this->logger->info("{$method} completed", $context);

            return $result;
        } catch (\Throwable $e) {
            $this->logger->error("{$method} failed", [
                'method' => $method,
                'error' => $e->getMessage(),
                'trace' => $e->getTraceAsString()
            ]);
            throw $e;
        }
    }

    protected function sanitizeArgs(array $args): array
    {
        // Remove sensitive data from logs
        return array_map(function($arg) {
            if (is_array($arg)) {
                return array_map(function($key, $value) {
                    return in_array(strtolower($key), ['password', 'secret', 'token', 'key'])
                        ? '***REDACTED***'
                        : $value;
                }, array_keys($arg), array_values($arg));
            }
            return $arg;
        }, $args);
    }

    protected function sanitizeResult($result): mixed
    {
        if (is_array($result)) {
            return array_map(function($key, $value) {
                return in_array(strtolower($key), ['password', 'secret', 'token'])
                    ? '***REDACTED***'
                    : $value;
            }, array_keys($result), array_values($result));
        }
        return $result;
    }
}
```

### Validation Decorator

```php
<?php
// includes/Decorators/ValidationDecorator.php

namespace CustomModule\Decorators;

use CustomModule\Validators\ValidatorInterface;

class ValidationDecorator extends Decorator
{
    protected array $validators = [];

    public function __construct(object $wrapped, array $validators = [])
    {
        parent::__construct($wrapped);
        $this->validators = $validators;
    }

    public function __call(string $method, array $args): mixed
    {
        // Run validation before method call
        $validationContext = $this->buildValidationContext($method, $args);

        foreach ($this->validators as $validator) {
            if (!$validator->validate($validationContext)) {
                throw new ValidationException(
                    $validator->getErrorMessage(),
                    $validator->getErrorCode()
                );
            }
        }

        // Execute the actual method
        return parent::__call($method, $args);
    }

    public function addValidator(ValidatorInterface $validator): self
    {
        $this->validators[] = $validator;
        return $this;
    }

    protected function buildValidationContext(string $method, array $args): array
    {
        return [
            'method' => $method,
            'arguments' => $args,
            'timestamp' => date('Y-m-d H:i:s')
        ];
    }
}
```

### Composing Decorators

```php
<?php
// includes/Decorators/DecoratorStack.php

namespace CustomModule\Decorators;

class DecoratorStack
{
    protected object $core;
    protected array $decorators = [];

    /**
     * Set the core service to wrap
     */
    public function setCore(object $service): self
    {
        $this->core = $service;
        return $this;
    }

    /**
     * Add a decorator to the stack
     */
    public function add(callable $decoratorFactory, int $priority = 0): self
    {
        $this->decorators[] = [
            'factory' => $decoratorFactory,
            'priority' => $priority
        ];

        // Sort by priority (higher = closer to core)
        usort($this->decorators, fn($a, $b) => $b['priority'] <=> $a['priority']);

        return $this;
    }

    /**
     * Build the decorated service
     */
    public function build(): object
    {
        if (empty($this->decorators)) {
            return $this->core;
        }

        $wrapped = $this->core;

        foreach ($this->decorators as $decorator) {
            $wrapped = call_user_func($decorator['factory'], $wrapped);
        }

        return $wrapped;
    }
}
```

### Usage in WHMCS Services

```php
<?php
// includes/Services/DecoratedClientService.php

namespace CustomModule\Services;

use WHMCS\Database\Capsule;
use CustomModule\Decorators\CachingDecorator;
use CustomModule\Decorators\LoggingDecorator;
use CustomModule\Decorators\ValidationDecorator;

class ClientService
{
    public function getClient(int $clientId): ?object
    {
        return Capsule::table('tblclients')->find($clientId);
    }

    public function createClient(array $data): int
    {
        return Capsule::table('tblclients')->insertGetId($data);
    }

    public function updateClient(int $clientId, array $data): bool
    {
        return Capsule::table('tblclients')
            ->where('id', $clientId)
            ->update($data) > 0;
    }
}

// Creating a decorated client service
$coreService = new ClientService();

// Build decorator stack
$decoratedService = (new DecoratorStack())
    ->setCore($coreService)
    ->add(function($wrapped) use ($logger) {
        return new LoggingDecorator($wrapped, $logger, ['getClient', 'createClient', 'updateClient']);
    }, 10)
    ->add(function($wrapped) {
        return new CachingDecorator($wrapped, 'clients', 300);
    }, 20)
    ->add(function($wrapped) {
        return new ValidationDecorator($wrapped, [
            new ClientIdValidator(),
            new ClientEmailValidator()
        ]);
    }, 30)
    ->build();

// Use decorated service
$client = $decoratedService->getClient(123);
```

### Webhook Processing Decorator Chain

```php
<?php
// includes/Decorators/WebhookProcessor.php

namespace CustomModule\Decorators;

class WebhookProcessor extends Decorator
{
    protected array $preprocessors = [];
    protected array $postprocessors = [];

    public function __construct(object $wrapped)
    {
        parent::__construct($wrapped);
    }

    public function processWebhook(string $provider, array $payload): array
    {
        // Run preprocessors
        foreach ($this->preprocessors as $processor) {
            $payload = $processor->process($payload);
            if ($payload === null) {
                return ['success' => false, 'error' => 'Preprocessing failed'];
            }
        }

        // Call the wrapped handler
        $result = $this->wrapped->handleWebhook($provider, $payload);

        // Run postprocessors
        foreach ($this->postprocessors as $processor) {
            $result = $processor->process($result);
        }

        return $result;
    }

    public function addPreprocessor(WebhookPreprocessorInterface $processor): self
    {
        $this->preprocessors[] = $processor;
        return $this;
    }

    public function addPostprocessor(WebhookPostprocessorInterface $processor): self
    {
        $this->postprocessors[] = $processor;
        return $this;
    }
}

// Preprocessor: Verify signature
class SignatureVerificationPreprocessor implements WebhookPreprocessorInterface
{
    public function process(array $payload): ?array
    {
        $signature = $payload['signature'] ?? '';
        $secret = getenv('WEBHOOK_SECRET');

        if (!$this->verifySignature($signature, $payload, $secret)) {
            throw new UnauthorizedException('Invalid webhook signature');
        }

        unset($payload['signature']);
        return $payload;
    }

    protected function verifySignature(string $signature, array $payload, string $secret): bool
    {
        $expected = hash_hmac('sha256', json_encode($payload), $secret);
        return hash_equals($expected, $signature);
    }
}

// Preprocessor: Rate limiting
class RateLimitPreprocessor implements WebhookPreprocessorInterface
{
    public function process(array $payload): ?array
    {
        $ip = $_SERVER['REMOTE_ADDR'] ?? '';
        $key = "webhook_rate:{$ip}";

        $count = Capsule::table('mod_cache')
            ->where('key', $key)
            ->first()->value ?? 0;

        if ($count > 100) {
            throw new RateLimitException('Too many requests');
        }

        Capsule::table('mod_cache')->updateOrInsert(
            ['key' => $key],
            ['value' => $count + 1, 'expires_at' => date('Y-m-d H:i:s', strtotime('+1 minute'))]
        );

        return $payload;
    }
}
```

### Invoice Processing Decorators

```php
<?php
// includes/Decorators/InvoiceProcessorDecorator.php

namespace CustomModule\Decorators;

class InvoiceProcessingDecorator extends Decorator
{
    public function createInvoice(array $data): int
    {
        // Add default values
        $data['date'] = $data['date'] ?? date('Y-m-d');
        $data['duedate'] = $data['duedate'] ?? date('Y-m-d', strtotime('+14 days'));
        $data['status'] = $data['status'] ?? 'Draft';
        $data['paymentmethod'] = $data['paymentmethod'] ?? 'invoice';

        return $this->wrapped->createInvoice($data);
    }

    public function addLineItem(int $invoiceId, array $item): array
    {
        // Auto-calculate taxes if not provided
        if (!isset($item['taxrate']) && !isset($item['taxed'])) {
            $client = $this->getClientFromInvoice($invoiceId);
            $item['taxrate'] = $this->calculateTaxRate($client);
            $item['taxed'] = true;
        }

        return $this->wrapped->addLineItem($invoiceId, $item);
    }

    public function sendInvoice(int $invoiceId): bool
    {
        // Add invoice email template variables
        $invoice = $this->wrapped->getInvoice($invoiceId);

        add_hook('EmailPreSend', 1, function($vars) use ($invoice) {
            $vars['custom_invoice_number'] = 'INV-' . str_pad($invoice->id, 6, '0', STR_PAD_LEFT);
            $vars['invoice_notes'] = $this->getInvoiceNotes($invoice);
            return $vars;
        });

        return $this->wrapped->sendInvoice($invoiceId);
    }

    protected function getClientFromInvoice(int $invoiceId): ?object
    {
        $invoice = Capsule::table('tblinvoices')->find($invoiceId);
        if ($invoice && $invoice->userid) {
            return Capsule::table('tblclients')->find($invoice->userid);
        }
        return null;
    }

    protected function calculateTaxRate(object $client): float
    {
        // Default tax rate based on country
        return match($client->country ?? 'US') {
            'GB', 'DE', 'FR' => 0.20,
            'CA' => 0.13,
            'AU' => 0.10,
            default => 0.0
        };
    }

    protected function getInvoiceNotes(object $invoice): string
    {
        return "Thank you for your business!";
    }
}
```

## Pros

- **Flexibility**: Add/remove behaviors without changing classes
- **Single Responsibility**: Each decorator handles one concern
- **Composition**: Combine multiple decorators
- **Extensibility**: Add new features without modifying existing code
- **Testing**: Test decorators individually

## Cons

- **Complexity**: Many small classes
- **Order Sensitivity**: Decorator order can affect behavior
- **Debugging**: Hard to trace through multiple layers
- **Metadata**: Can't set decorator metadata on methods

## Best Practices

1. Keep decorators focused on single responsibility
2. Document the order requirements
3. Use decorator stacks for managing multiple decorators
4. Consider using attributes/annotations for decorator configuration
5. Implement interface-based decorators for type safety