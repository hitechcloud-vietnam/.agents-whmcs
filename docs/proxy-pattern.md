# Proxy Pattern in WHMCS

The Proxy Pattern provides a substitute or placeholder for another object to control access to it. In WHMCS, this pattern is excellent for lazy loading, access control, logging, caching, and remote service integration.

## Overview

A proxy acts as an intermediary between the client and the real object:
- **Virtual Proxy**: Delays expensive object creation
- **Protection Proxy**: Controls access to the real object
- **Remote Proxy**: Represents objects in different address spaces
- **Smart Proxy**: Adds additional functionality on access

## Core Structure

### Virtual Proxy

```php
<?php
// includes/Proxy/VirtualProxy.php

namespace CustomModule\Proxy;

/**
 * Virtual proxy - delays object creation until needed
 */
class VirtualProxy
{
    protected string $realClass;
    protected ?object $instance = null;
    protected array $constructorArgs;

    public function __construct(string $realClass, array $constructorArgs = [])
    {
        $this->realClass = $realClass;
        $this->constructorArgs = $constructorArgs;
    }

    public function getInstance(): object
    {
        if ($this->instance === null) {
            $this->instance = new $this->realClass(...array_values($this->constructorArgs));
        }
        return $this->instance;
    }

    public function __call(string $method, array $args)
    {
        return call_user_func_array([$this->getInstance(), $method], $args);
    }
}
```

## Real-World WHMCS Examples

### Lazy-Loading Service Proxy

```php
<?php
// includes/Proxy/LazyServiceProxy.php

namespace CustomModule\Proxy;

use WHMCS\Database\Capsule;

/**
 * Lazy loads expensive service data on first access
 */
class LazyServiceProxy
{
    protected int $serviceId;
    protected ?object $serviceData = null;
    protected ?object $clientData = null;
    protected ?array $configData = null;

    public function __construct(int $serviceId)
    {
        $this->serviceId = $serviceId;
    }

    protected function loadServiceData(): void
    {
        if ($this->serviceData !== null) {
            return;
        }

        $this->serviceData = Capsule::table('tblhosting')
            ->where('id', $this->serviceId)
            ->first();

        if (!$this->serviceData) {
            throw new \RuntimeException("Service not found: {$this->serviceId}");
        }
    }

    public function getServiceData(): object
    {
        $this->loadServiceData();
        return $this->serviceData;
    }

    public function getClientId(): int
    {
        $this->loadServiceData();
        return $this->serviceData->userid;
    }

    public function getDomain(): string
    {
        $this->loadServiceData();
        return $this->serviceData->domain;
    }

    public function getClientData(): object
    {
        if ($this->clientData === null) {
            $this->loadServiceData();
            $this->clientData = Capsule::table('tblclients')
                ->where('id', $this->serviceData->userid)
                ->first();
        }
        return $this->clientData;
    }

    public function getConfigData(): array
    {
        if ($this->configData === null) {
            $this->loadServiceData();
            $this->configData = json_decode($this->serviceData->configdata, true) ?? [];
        }
        return $this->configData;
    }

    public function getConfigOption(string $key, $default = null)
    {
        $config = $this->getConfigData();
        return $config[$key] ?? $default;
    }

    public function isSuspended(): bool
    {
        $this->loadServiceData();
        return $this->serviceData->domainstatus === 'Suspended';
    }

    public function isTerminated(): bool
    {
        $this->loadServiceData();
        return $this->serviceData->domainstatus === 'Terminated';
    }

    public function isActive(): bool
    {
        $this->loadServiceData();
        return $this->serviceData->domainstatus === 'Active';
    }

    // Lazy load relationships
    public function getAddons(): array
    {
        return Capsule::table('tblhostingaddons')
            ->where('hostingid', $this->serviceId)
            ->where('status', 'Active')
            ->get()
            ->toArray();
    }

    public function getInvoices(): array
    {
        return Capsule::table('tblinvoices')
            ->where('userid', $this->getClientId())
            ->where('status', 'Unpaid')
            ->get()
            ->toArray();
    }
}
```

### Protection Proxy with Access Control

```php
<?php
// includes/Proxy/ProtectedServiceProxy.php

namespace CustomModule\Proxy;

use WHMCS\Authentication\CurrentUser;

/**
 * Protection proxy - controls access to sensitive operations
 */
class ProtectedServiceProxy
{
    protected object $realService;
    protected array $permissions = [];
    protected array $auditLog = [];

    public function __construct(object $realService, array $permissions = [])
    {
        $this->realService = $realService;
        $this->permissions = $permissions;
    }

    public function __call(string $method, array $args)
    {
        // Check if method requires permission
        if ($this->requiresPermission($method)) {
            $this->checkPermission($method);
        }

        // Log access
        $this->logAccess($method, $args);

        // Execute actual method
        return call_user_func_array([$this->realService, $method], $args);
    }

    protected function requiresPermission(string $method): bool
    {
        $sensitiveMethods = [
            'deleteClient' => 'DeleteClient',
            'terminateService' => 'TerminateService',
            'processRefund' => 'ProcessRefund',
            'modifyPricing' => 'ModifyPricing',
            'accessLogs' => 'ViewAuditLogs'
        ];

        return isset($sensitiveMethods[$method]);
    }

    protected function checkPermission(string $method): void
    {
        $permissionMap = [
            'deleteClient' => 'DeleteClient',
            'terminateService' => 'TerminateService',
            'processRefund' => 'ProcessRefund',
            'modifyPricing' => 'ModifyPricing',
            'accessLogs' => 'ViewAuditLogs'
        ];

        $permission = $permissionMap[$method] ?? null;

        if ($permission && !$this->hasPermission($permission)) {
            throw new \RuntimeException("Permission denied: {$permission}");
        }
    }

    protected function hasPermission(string $permission): bool
    {
        $currentUser = new CurrentUser();

        if ($currentUser->isAdmin()) {
            // Admin has all permissions
            return true;
        }

        return in_array($permission, $this->permissions);
    }

    protected function logAccess(string $method, array $args): void
    {
        $this->auditLog[] = [
            'timestamp' => date('Y-m-d H:i:s'),
            'method' => $method,
            'args' => $this->sanitizeArgs($args),
            'user_id' => $_SESSION['adminid'] ?? 'system',
            'ip_address' => $_SERVER['REMOTE_ADDR'] ?? 'unknown'
        ];

        // Persist audit log
        if (count($this->auditLog) >= 10) {
            $this->flushAuditLog();
        }
    }

    protected function sanitizeArgs(array $args): array
    {
        // Remove sensitive data from logs
        return array_map(function($arg) {
            if (is_array($arg)) {
                return array_map(function($key, $value) {
                    return in_array(strtolower($key), ['password', 'secret', 'token'])
                        ? '***REDACTED***'
                        : $value;
                }, array_keys($arg), array_values($arg));
            }
            return $arg;
        }, $args);
    }

    protected function flushAuditLog(): void
    {
        foreach ($this->auditLog as $logEntry) {
            Capsule::table('mod_audit_log')->insert($logEntry);
        }
        $this->auditLog = [];
    }

    public function getAuditLog(): array
    {
        return $this->auditLog;
    }
}
```

### Caching Proxy

```php
<?php
// includes/Proxy/CachedServiceProxy.php

namespace CustomModule\Proxy;

use WHMCS\Cache\CacheManager;

/**
 * Caching proxy - caches results of expensive operations
 */
class CachedServiceProxy
{
    protected object $realService;
    protected CacheManager $cache;
    protected string $cachePrefix;
    protected int $defaultTtl = 300;
    protected array $methodCacheConfig = [];

    public function __construct(object $realService, string $cachePrefix = 'default')
    {
        $this->realService = $realService;
        $this->cache = CacheManager::driver('file');
        $this->cachePrefix = $cachePrefix;
    }

    public function setCacheConfig(string $method, int $ttl, array $tags = []): self
    {
        $this->methodCacheConfig[$method] = [
            'ttl' => $ttl,
            'tags' => $tags
        ];
        return $this;
    }

    public function __call(string $method, array $args)
    {
        $cacheConfig = $this->methodCacheConfig[$method] ?? null;

        // If method is not cacheable, call directly
        if ($cacheConfig === false || !isset($this->methodCacheConfig[$method]) && !$this->isCacheableMethod($method)) {
            return call_user_func_array([$this->realService, $method], $args);
        }

        $cacheKey = $this->buildCacheKey($method, $args);
        $ttl = $cacheConfig['ttl'] ?? $this->defaultTtl;

        // Try to get from cache
        $cached = $this->cache->get($cacheKey);
        if ($cached !== null) {
            return $cached;
        }

        // Execute and cache result
        $result = call_user_func_array([$this->realService, $method], $args);

        // Don't cache null results or errors
        if ($result !== null && !($result instanceof ErrorResult)) {
            $this->cache->put($cacheKey, $result, $ttl);

            // Add cache tags if configured
            if (!empty($cacheConfig['tags'])) {
                $this->addCacheTags($cacheKey, $cacheConfig['tags']);
            }
        }

        return $result;
    }

    protected function isCacheableMethod(string $method): bool
    {
        $nonCacheable = ['create', 'update', 'delete', 'remove'];

        foreach ($nonCacheable as $prefix) {
            if (strpos($method, $prefix) === 0) {
                return false;
            }
        }

        return true;
    }

    protected function buildCacheKey(string $method, array $args): string
    {
        return "{$this->cachePrefix}:{$method}:" . md5(serialize($args));
    }

    protected function addCacheTags(string $cacheKey, array $tags): void
    {
        foreach ($tags as $tag) {
            $tagKey = "tag:{$tag}";
            $taggedKeys = $this->cache->get($tagKey) ?? [];
            $taggedKeys[] = $cacheKey;
            $this->cache->put($tagKey, array_unique($taggedKeys), 86400);
        }
    }

    public function invalidateByTag(string $tag): void
    {
        $tagKey = "tag:{$tag}";
        $taggedKeys = $this->cache->get($tagKey) ?? [];

        foreach ($taggedKeys as $key) {
            $this->cache->forget($key);
        }

        $this->cache->forget($tagKey);
    }

    public function clearCache(): void
    {
        // Clear all cached values with this prefix
        // Implementation depends on cache driver capabilities
    }
}
```

### Remote Proxy for External API

```php
<?php
// includes/Proxy/RemoteServiceProxy.php

namespace CustomModule\Proxy;

use GuzzleHttp\Client;
use GuzzleHttp\Exception\RequestException;

/**
 * Remote proxy - handles communication with external services
 */
class RemoteServiceProxy
{
    protected Client $httpClient;
    protected string $baseUrl;
    protected string $apiKey;
    protected int $timeout = 30;
    protected array $retryConfig = ['max_attempts' => 3, 'delay_ms' => 1000];

    public function __construct(string $baseUrl, string $apiKey)
    {
        $this->baseUrl = rtrim($baseUrl, '/');
        $this->apiKey = $apiKey;

        $this->httpClient = new Client([
            'base_uri' => $this->baseUrl,
            'timeout' => $this->timeout,
            'headers' => [
                'Authorization' => "Bearer {$this->apiKey}",
                'Content-Type' => 'application/json',
                'User-Agent' => 'WHMCS-CustomModule/1.0'
            ]
        ]);
    }

    public function __call(string $method, array $args)
    {
        $endpoint = $this->buildEndpoint($method, $args);
        $httpMethod = $this->determineHttpMethod($method);
        $body = $this->extractBody($args);

        return $this->executeWithRetry($httpMethod, $endpoint, $body);
    }

    protected function buildEndpoint(string $method, array $args): string
    {
        // Convert camelCase method to snake_case endpoint
        $endpoint = preg_replace('/(?<!^)[A-Z]/', '_', lcfirst($method));
        $endpoint = strtolower($endpoint);

        // Replace IDs in path
        if (preg_match('/ById$/', $method)) {
            $id = array_shift($args);
            $endpoint = str_replace('_by_id', "/{$id}", $endpoint);
        }

        return '/' . ltrim($endpoint, '/');
    }

    protected function determineHttpMethod(string $method): string
    {
        if (strpos($method, 'get') === 0) return 'GET';
        if (strpos($method, 'create') === 0) return 'POST';
        if (strpos($method, 'update') === 0) return 'PUT';
        if (strpos($method, 'delete') === 0) return 'DELETE';

        return 'POST';
    }

    protected function extractBody(array $args): array
    {
        // Extract body parameters from args
        foreach ($args as $arg) {
            if (is_array($arg)) {
                return $arg;
            }
        }
        return [];
    }

    protected function executeWithRetry(string $method, string $endpoint, array $body): mixed
    {
        $attempts = 0;
        $maxAttempts = $this->retryConfig['max_attempts'];
        $delayMs = $this->retryConfig['delay_ms'];

        while ($attempts < $maxAttempts) {
            try {
                return $this->executeRequest($method, $endpoint, $body);
            } catch (RequestException $e) {
                $attempts++;

                if ($attempts >= $maxAttempts) {
                    throw $e;
                }

                // Exponential backoff
                usleep($delayMs * 1000 * pow(2, $attempts - 1));

                // Log retry
                logActivity("Remote API retry attempt {$attempts}: {$endpoint}");
            }
        }

        throw new \RuntimeException('Max retry attempts exceeded');
    }

    protected function executeRequest(string $method, string $endpoint, array $body): mixed
    {
        $options = [];

        if (!empty($body) && in_array($method, ['POST', 'PUT', 'PATCH'])) {
            $options['json'] = $body;
        } elseif (!empty($body)) {
            $options['query'] = $body;
        }

        $response = $this->httpClient->request($method, $endpoint, $options);

        return json_decode($response->getBody()->getContents(), true);
    }

    public function setTimeout(int $seconds): self
    {
        $this->timeout = $seconds;
        return $this;
    }

    public function setRetryConfig(int $maxAttempts, int $delayMs): self
    {
        $this->retryConfig = [
            'max_attempts' => $maxAttempts,
            'delay_ms' => $delayMs
        ];
        return $this;
    }
}
```

### Logging Proxy

```php
<?php
// includes/Proxy/LoggingProxy.php

namespace CustomModule\Proxy;

use Psr\Log\LoggerInterface;

/**
 * Logging proxy - logs all method calls
 */
class LoggingProxy
{
    protected object $realService;
    protected LoggerInterface $logger;
    protected array $methodsToLog;
    protected bool $logArguments;
    protected bool $logResults;

    public function __construct(
        object $realService,
        LoggerInterface $logger,
        array $methodsToLog = [],
        bool $logArguments = true,
        bool $logResults = false
    ) {
        $this->realService = $realService;
        $this->logger = $logger;
        $this->methodsToLog = $methodsToLog;
        $this->logArguments = $logArguments;
        $this->logResults = $logResults;
    }

    public function __call(string $method, array $args)
    {
        // Check if method should be logged
        if (!empty($this->methodsToLog) && !in_array($method, $this->methodsToLog)) {
            return call_user_func_array([$this->realService, $method], $args);
        }

        $context = [
            'service' => get_class($this->realService),
            'method' => $method
        ];

        if ($this->logArguments) {
            $context['arguments'] = $this->sanitizeData($args);
        }

        $this->logger->info("Calling {$method}", $context);

        $startTime = microtime(true);

        try {
            $result = call_user_func_array([$this->realService, $method], $args);

            $duration = round((microtime(true) - $startTime) * 1000, 2);
            $context['duration_ms'] = $duration;

            if ($this->logResults) {
                $context['result'] = $this->sanitizeData([$result]);
            }

            $this->logger->info("{$method} completed", $context);

            return $result;
        } catch (\Throwable $e) {
            $this->logger->error("{$method} failed", [
                'service' => get_class($this->realService),
                'method' => $method,
                'error' => $e->getMessage(),
                'trace' => $e->getTraceAsString()
            ]);
            throw $e;
        }
    }

    protected function sanitizeData(array $data): array
    {
        return array_map(function($item) {
            if (is_array($item)) {
                return array_map(function($key, $value) {
                    if (in_array(strtolower($key), ['password', 'secret', 'token', 'key', 'auth'])) {
                        return '***REDACTED***';
                    }
                    return $value;
                }, array_keys($item), array_values($item));
            }
            return $item;
        }, $data);
    }
}
```

### Usage Examples

```php
<?php
// Combining proxies for comprehensive service management

// Create the real service
$clientService = new ClientService();

// Wrap with logging
$loggedService = new LoggingProxy(
    $clientService,
    $logger,
    ['createClient', 'updateClient', 'deleteClient']
);

// Wrap with caching
$cachedService = new CachedServiceProxy($loggedService, 'clients');
$cachedService->setCacheConfig('getClient', 300, ['client_data']);
$cachedService->setCacheConfig('searchClients', 60, ['client_search']);

// Wrap with access control
$protectedService = new ProtectedServiceProxy($cachedService, [
    'ViewClient',
    'CreateClient',
    'EditClient'
]);

// Use the proxy chain
$client = $protectedService->getClient(123);
$protectedService->updateClient(123, ['firstname' => 'John']);
```

### Service Proxy Factory

```php
<?php
// includes/Proxy/ProxyFactory.php

namespace CustomModule\Proxy;

class ProxyFactory
{
    protected static array $proxies = [];

    public static function createLazyProxy(string $class, array $args = []): LazyServiceProxy
    {
        $key = $class . ':' . md5(serialize($args));

        if (!isset(self::$proxies[$key])) {
            self::$proxies[$key] = new VirtualProxy($class, $args);
        }

        return self::$proxies[$key];
    }

    public static function createCachedProxy(object $service, string $prefix): CachedServiceProxy
    {
        return new CachedServiceProxy($service, $prefix);
    }

    public static function createLoggingProxy(object $service, LoggerInterface $logger): LoggingProxy
    {
        return new LoggingProxy($service, $logger);
    }

    public static function createProtectedProxy(object $service, array $permissions): ProtectedServiceProxy
    {
        return new ProtectedServiceProxy($service, $permissions);
    }

    public static function wrapWithAllProxies(
        object $service,
        array $config
    ): ProtectedServiceProxy {
        $wrapped = $service;

        // Add logging first (innermost)
        if (!empty($config['log_methods'])) {
            $wrapped = new LoggingProxy(
                $wrapped,
                $config['logger'],
                $config['log_methods']
            );
        }

        // Add caching
        if (!empty($config['cache_prefix'])) {
            $cachedProxy = new CachedServiceProxy($wrapped, $config['cache_prefix']);

            if (!empty($config['cache_config'])) {
                foreach ($config['cache_config'] as $method => $cacheConfig) {
                    $cachedProxy->setCacheConfig(
                        $method,
                        $cacheConfig['ttl'],
                        $cacheConfig['tags'] ?? []
                    );
                }
            }

            $wrapped = $cachedProxy;
        }

        // Add protection (outermost)
        if (!empty($config['permissions'])) {
            $wrapped = new ProtectedServiceProxy($wrapped, $config['permissions']);
        }

        return $wrapped;
    }
}
```

## Pros

- **Lazy Loading**: Delay expensive operations until needed
- **Access Control**: Centralize permission checks
- **Caching**: Reduce repeated expensive operations
- **Logging**: Add transparent logging
- **Flexibility**: Combine multiple proxies

## Cons

- **Complexity**: Proxy chains can be hard to follow
- **Overhead**: Each proxy adds slight performance cost
- **Debugging**: Harder to trace through proxy layers
- **Interface Matching**: Must implement all target methods

## Best Practices

1. Use proxy chains for different cross-cutting concerns
2. Keep proxy logic simple and focused
3. Document proxy behavior clearly
4. Consider using decorators instead of explicit proxies
5. Implement proper error handling in remote proxies