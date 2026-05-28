# Middleware Pattern in WHMCS

Middleware pattern provides a way to process requests and responses through a chain of handlers, allowing for cross-cutting concerns like logging, authentication, and caching to be applied consistently across your module.

## Overview

In WHMCS, middleware acts as a bridge between the entry point of your module and the final action. Each middleware component can:
- Process the request before passing it along
- Transform the response before returning it
- Short-circuit the request pipeline (auth failures, rate limits)
- Perform cleanup after the response is sent

## Core Structure

### Base Middleware Interface

```php
<?php
// includes/Middleware/MiddlewareInterface.php

namespace WHMCS\Module\Middleware;

interface MiddlewareInterface
{
    /**
     * Handle the request and return a response
     *
     * @param mixed $request The incoming request
     * @param callable $next The next middleware or handler
     * @return mixed
     */
    public function handle($request, callable $next);
}
```

### Middleware Pipeline

```php
<?php
// includes/Middleware/Pipeline.php

namespace WHMCS\Module\Middleware;

class Pipeline
{
    protected array $middlewares = [];
    protected $destination;

    public function send($request)
    {
        return $this->execute($request);
    }

    public function through(array $middlewares): self
    {
        $this->middlewares = $middlewares;
        return $this;
    }

    public function then(callable $destination): mixed
    {
        $this->destination = $destination;
        return $this->execute($request ?? null);
    }

    protected function execute($request)
    {
        $next = function ($request) {
            return call_user_func($this->destination, $request);
        };

        foreach (array_reverse($this->middlewares) as $middleware) {
            $currentNext = $next;
            $middlewareInstance = $this->resolveMiddleware($middleware);
            $next = function ($request) use ($middlewareInstance, $currentNext) {
                return $middlewareInstance->handle($request, $currentNext);
            };
        }

        return $next($request);
    }

    protected function resolveMiddleware($middleware): MiddlewareInterface
    {
        if (is_string($middleware)) {
            return new $middleware();
        }
        return $middleware;
    }
}
```

## Real-World WHMCS Examples

### Authentication Middleware

```php
<?php
// includes/Middleware/AuthMiddleware.php

namespace CustomModule\Middleware;

use WHMCS\Module\Middleware\MiddlewareInterface;

class AuthMiddleware implements MiddlewareInterface
{
    public function handle($request, callable $next)
    {
        $userId = $request['user_id'] ?? null;
        $user = $request['user'] ?? null;

        if (!$userId && !$user) {
            return [
                'success' => false,
                'error' => 'Authentication required',
                'status_code' => 401
            ];
        }

        if ($user && $user->status !== 'Active') {
            return [
                'success' => false,
                'error' => 'Account is not active',
                'status_code' => 403
            ];
        }

        // Continue to next handler
        return $next($request);
    }
}
```

### Logging Middleware

```php
<?php
// includes/Middleware/LoggingMiddleware.php

namespace CustomModule\Middleware;

use WHMCS\Module\Middleware\MiddlewareInterface;

class LoggingMiddleware implements MiddlewareInterface
{
    protected LoggerInterface $logger;

    public function __construct(LoggerInterface $logger)
    {
        $this->logger = $logger;
    }

    public function handle($request, callable $next)
    {
        $startTime = microtime(true);
        $requestId = uniqid('req_');

        // Log incoming request
        $this->logger->info("Request started", [
            'request_id' => $requestId,
            'path' => $request['path'] ?? 'unknown',
            'method' => $request['method'] ?? 'unknown'
        ]);

        try {
            $response = $next($request);

            $duration = microtime(true) - $startTime;
            $this->logger->info("Request completed", [
                'request_id' => $requestId,
                'duration_ms' => round($duration * 1000, 2),
                'success' => $response['success'] ?? true
            ]);

            return $response;
        } catch (\Exception $e) {
            $this->logger->error("Request failed", [
                'request_id' => $requestId,
                'exception' => $e->getMessage()
            ]);
            throw $e;
        }
    }
}
```

### Rate Limiting Middleware

```php
<?php
// includes/Middleware/RateLimitMiddleware.php

namespace CustomModule\Middleware;

use WHMCS\Module\Middleware\MiddlewareInterface;
use WHMCS\Carbon;

class RateLimitMiddleware implements MiddlewareInterface
{
    protected int $maxRequests;
    protected int $windowSeconds;

    public function __construct(int $maxRequests = 60, int $windowSeconds = 60)
    {
        $this->maxRequests = $maxRequests;
        $this->windowSeconds = $windowSeconds;
    }

    public function handle($request, callable $next)
    {
        $identifier = $this->getIdentifier($request);
        $key = "rate_limit:{$identifier}";

        $currentCount = (int) Capsule::table('mod_custom_cache')
            ->where('key', $key)
            ->where('expires_at', '>', Carbon::now())
            ->first()->value ?? 0;

        if ($currentCount >= $this->maxRequests) {
            return [
                'success' => false,
                'error' => 'Rate limit exceeded',
                'retry_after' => $this->windowSeconds,
                'status_code' => 429
            ];
        }

        // Increment counter
        $this->incrementCounter($key);

        $response = $next($request);

        // Add rate limit headers
        $response['headers'] = $response['headers'] ?? [];
        $response['headers']['X-RateLimit-Remaining'] = $this->maxRequests - $currentCount - 1;
        $response['headers']['X-RateLimit-Limit'] = $this->maxRequests;

        return $response;
    }

    protected function getIdentifier($request): string
    {
        return $request['user_id'] ?? $request['ip_address'] ?? 'anonymous';
    }

    protected function incrementCounter(string $key): void
    {
        Capsule::table('mod_custom_cache')->updateOrInsert(
            ['key' => $key],
            [
                'value' => Capsule::raw('value + 1'),
                'expires_at' => Carbon::now()->addSeconds($this->windowSeconds)
            ]
        );
    }
}
```

### Caching Middleware

```php
<?php
// includes/Middleware/CacheMiddleware.php

namespace CustomModule\Middleware;

use WHMCS\Module\Middleware\MiddlewareInterface;
use WHMCS\Cache\CacheManager;

class CacheMiddleware implements MiddlewareInterface
{
    protected string $cacheKey;
    protected int $ttlSeconds;
    protected bool $useCache;

    public function __construct(string $cacheKey, int $ttlSeconds = 300, bool $useCache = true)
    {
        $this->cacheKey = $cacheKey;
        $this->ttlSeconds = $ttlSeconds;
        $this->useCache = $useCache;
    }

    public function handle($request, callable $next)
    {
        if (!$this->useCache) {
            return $next($request);
        }

        // Try to get from cache
        $cache = CacheManager::driver('file');
        $cached = $cache->get($this->cacheKey);

        if ($cached !== null) {
            return [
                'success' => true,
                'data' => $cached,
                'cached' => true,
                'cache_age' => $cached['_cached_at'] ?? null
            ];
        }

        // Execute handler
        $response = $next($request);

        // Store in cache if successful
        if ($response['success'] ?? false) {
            $cachedData = $response['data'];
            $cachedData['_cached_at'] = time();
            $cache->put($this->cacheKey, $cachedData, $this->ttlSeconds);
            $response['cached'] = false;
        }

        return $response;
    }
}
```

### Usage in Module Controller

```php
<?php
// controllers/ApiController.php

namespace CustomModule\Controllers;

use WHMCS\Module\Middleware\Pipeline;
use CustomModule\Middleware\AuthMiddleware;
use CustomModule\Middleware\LoggingMiddleware;
use CustomModule\Middleware\RateLimitMiddleware;
use CustomModule\Middleware\CacheMiddleware;

class ApiController
{
    public function getClients($params)
    {
        $request = [
            'user_id' => $_SESSION['uid'] ?? null,
            'user' => $this->getUserFromSession(),
            'path' => '/api/clients',
            'method' => 'GET',
            'ip_address' => $_SERVER['REMOTE_ADDR'] ?? '',
            'params' => $params
        ];

        $pipeline = new Pipeline();
        $response = $pipeline
            ->send($request)
            ->through([
                LoggingMiddleware::class,
                RateLimitMiddleware::class,
                new CacheMiddleware('clients_list', 60),
                AuthMiddleware::class,
            ])
            ->then(function ($request) {
                return $this->fetchClients($request['params']);
            });

        return $response;
    }

    protected function fetchClients($params): array
    {
        $query = Capsule::table('tblclients');

        if (!empty($params['status'])) {
            $query->where('status', $params['status']);
        }

        $clients = $query->limit(100)->get();

        return [
            'success' => true,
            'data' => $clients
        ];
    }

    protected function getUserFromSession()
    {
        if (empty($_SESSION['uid'])) {
            return null;
        }
        return Capsule::table('tblclients')
            ->where('id', $_SESSION['uid'])
            ->first();
    }
}
```

## Pros

- **Separation of Concerns**: Each middleware handles one responsibility
- **Reusability**: Middleware components can be reused across different controllers
- **Testability**: Individual middleware can be unit tested in isolation
- **Flexibility**: Easy to add, remove, or reorder middleware
- **Cross-Cutting Concerns**: Centralize logging, caching, auth in one place

## Cons

- **Complexity**: Additional abstraction layer can be overkill for simple modules
- **Debugging**: Tracing issues through multiple middleware layers takes time
- **Performance**: Each middleware adds slight overhead to request processing
- **Order Sensitivity**: Middleware order matters and can cause subtle bugs

## Best Practices

1. Keep middleware focused on a single responsibility
2. Always call the next middleware in the chain
3. Handle exceptions and ensure cleanup code runs
4. Document the order dependencies of middleware
5. Use middleware for cross-cutting concerns, not business logic
6. Consider using middleware for audit trails and compliance requirements