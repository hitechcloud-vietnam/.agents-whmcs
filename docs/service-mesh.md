# Service Mesh Patterns

Service mesh patterns provide infrastructure for managing service-to-service communication in complex WHMCS integrations with external services.

## Service Proxy

### Sidecar Proxy

```php
<?php
/**
 * Sidecar proxy for service communication
 */
class SidecarProxy
{
    private string $serviceName;
    private array $upstreamServices = [];
    private CircuitBreaker $circuitBreaker;
    private LoadBalancer $loadBalancer;
    private RateLimiter $rateLimiter;

    public function __construct(string $serviceName)
    {
        $this->serviceName = $serviceName;
        $this->circuitBreaker = new CircuitBreaker($serviceName);
        $this->loadBalancer = new LoadBalancer();
        $this->rateLimiter = new RateLimiter($serviceName);
    }

    /**
     * Register an upstream service
     */
    public function registerUpstream(string $serviceName, array $endpoints): void
    {
        $this->upstreamServices[$serviceName] = [
            'name' => $serviceName,
            'endpoints' => $endpoints,
            'healthy' => true,
        ];
    }

    /**
     * Make a request through the proxy
     */
    public function request(
        string $method,
        string $service,
        string $path,
        array $options = []
    ): array {
        // Check rate limit
        if (!$this->rateLimiter->allow($service)) {
            throw new RateLimitException("Rate limit exceeded for {$service}");
        }

        // Check circuit breaker
        if (!$this->circuitBreaker->isOpen()) {
            throw new CircuitOpenException("Circuit breaker open for {$service}");
        }

        // Select endpoint
        $endpoint = $this->selectEndpoint($service);
        $url = rtrim($endpoint, '/') . '/' . ltrim($path, '/');

        // Execute request
        try {
            $response = $this->executeRequest($method, $url, $options);
            $this->circuitBreaker->recordSuccess($service);
            return $response;
        } catch (\Throwable $e) {
            $this->circuitBreaker->recordFailure($service);
            throw $e;
        }
    }

    private function selectEndpoint(string $service): string
    {
        if (!isset($this->upstreamServices[$service])) {
            throw new ServiceNotFoundException("Service not registered: {$service}");
        }

        $endpoints = $this->upstreamServices[$service]['endpoints'];
        return $this->loadBalancer->select($endpoints);
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
            curl_setopt($ch, CURLOPT_HTTPHEADER, $options['headers']);
        }

        if (!empty($options['body'])) {
            curl_setopt($ch, CURLOPT_POSTFIELDS, json_encode($options['body']));
        }

        $response = curl_exec($ch);
        $statusCode = curl_getinfo($ch, CURLINFO_HTTP_CODE);
        curl_close($ch);

        return [
            'status_code' => $statusCode,
            'body' => json_decode($response, true) ?? $response,
        ];
    }
}
```

### Circuit Breaker

```php
<?php
/**
 * Circuit breaker pattern implementation
 */
class CircuitBreaker
{
    private string $service;
    private string $state = 'closed';
    private int $failureCount = 0;
    private int $successCount = 0;
    private ?int $lastFailureTime = null;
    private array $config;

    private const STATE_CLOSED = 'closed';
    private const STATE_OPEN = 'open';
    private const STATE_HALF_OPEN = 'half_open';

    public function __construct(string $service, array $config = [])
    {
        $this->service = $service;
        $this->config = array_merge([
            'failure_threshold' => 5,
            'success_threshold' => 3,
            'timeout' => 60, // seconds
        ], $config);
    }

    public function isOpen(): bool
    {
        if ($this->state === self::STATE_CLOSED) {
            return false;
        }

        if ($this->state === self::STATE_OPEN) {
            if ($this->shouldAttemptReset()) {
                $this->state = self::STATE_HALF_OPEN;
                return false;
            }
            return true;
        }

        return false; // Half-open allows requests
    }

    public function recordSuccess(string $service): void
    {
        if ($this->state === self::STATE_HALF_OPEN) {
            $this->successCount++;

            if ($this->successCount >= $this->config['success_threshold']) {
                $this->reset();
            }
        } else {
            $this->failureCount = 0;
        }
    }

    public function recordFailure(string $service): void
    {
        $this->failureCount++;
        $this->lastFailureTime = time();

        if ($this->state === self::STATE_HALF_OPEN) {
            $this->trip();
        } elseif ($this->failureCount >= $this->config['failure_threshold']) {
            $this->trip();
        }
    }

    public function trip(): void
    {
        $this->state = self::STATE_OPEN;
        $this->failureCount = 0;
        $this->successCount = 0;
    }

    public function reset(): void
    {
        $this->state = self::STATE_CLOSED;
        $this->failureCount = 0;
        $this->successCount = 0;
        $this->lastFailureTime = null;
    }

    private function shouldAttemptReset(): bool
    {
        if ($this->lastFailureTime === null) {
            return true;
        }

        return (time() - $this->lastFailureTime) >= $this->config['timeout'];
    }

    public function getState(): string
    {
        return $this->state;
    }

    public function getStatus(): array
    {
        return [
            'service' => $this->service,
            'state' => $this->state,
            'failure_count' => $this->failureCount,
            'success_count' => $this->successCount,
            'last_failure' => $this->lastFailureTime,
        ];
    }
}
```

### Load Balancer

```php
<?php
<?php
/**
 * Load balancer implementations
 */
class LoadBalancer
{
    private string $strategy;

    public function __construct(string $strategy = 'round_robin')
    {
        $this->strategy = $strategy;
    }

    public function select(array $endpoints): string
    {
        switch ($this->strategy) {
            case 'random':
                return $this->randomSelect($endpoints);

            case 'least_connections':
                return $this->leastConnectionsSelect($endpoints);

            case 'round_robin':
            default:
                return $this->roundRobinSelect($endpoints);
        }
    }

    private function randomSelect(array $endpoints): string
    {
        return $endpoints[array_rand($endpoints)];
    }

    private function roundRobinSelect(array $endpoints): string
    {
        static $index = [];

        $key = spl_object_hash($this);
        if (!isset($index[$key])) {
            $index[$key] = 0;
        }

        $selected = $index[$key] % count($endpoints);
        $index[$key]++;

        return $endpoints[$selected];
    }

    private function leastConnectionsSelect(array $endpoints): string
    {
        // In a real implementation, track connections per endpoint
        // For now, select randomly
        return $this->randomSelect($endpoints);
    }
}

/**
 * Weighted load balancer
 */
class WeightedLoadBalancer extends LoadBalancer
{
    private array $weights = [];

    public function setWeights(array $weights): void
    {
        $this->weights = $weights;
    }

    public function select(array $endpoints): string
    {
        if (empty($this->weights)) {
            return parent::select($endpoints);
        }

        $weightedEndpoints = [];

        foreach ($endpoints as $endpoint) {
            $weight = $this->weights[$endpoint] ?? 1;

            for ($i = 0; $i < $weight; $i++) {
                $weightedEndpoints[] = $endpoint;
            }
        }

        return $weightedEndpoints[array_rand($weightedEndpoints)];
    }
}
```

## Rate Limiting

### Token Bucket Rate Limiter

```php
<?php
/**
 * Token bucket rate limiter
 */
class RateLimiter
{
    private string $key;
    private float $capacity;
    private float $refillRate;
    private float $tokens;
    private ?float $lastRefill = null;

    public function __construct(string $key, float $capacity = 100, float $refillRate = 10)
    {
        $this->key = $key;
        $this->capacity = $capacity;
        $this->refillRate = $refillRate;
        $this->tokens = $capacity;
        $this->lastRefill = microtime(true);
    }

    public function allow(string $key = null): bool
    {
        $key = $key ?? $this->key;

        // Refill tokens
        $this->refill();

        if ($this->tokens >= 1) {
            $this->tokens--;
            $this->saveState();
            return true;
        }

        return false;
    }

    public function getAvailableTokens(): float
    {
        $this->refill();
        return $this->tokens;
    }

    public function getWaitTime(): float
    {
        if ($this->tokens >= 1) {
            return 0;
        }

        return (1 - $this->tokens) / $this->refillRate;
    }

    private function refill(): void
    {
        $now = microtime(true);

        if ($this->lastRefill === null) {
            $this->lastRefill = $now;
            return;
        }

        $elapsed = $now - $this->lastRefill;
        $newTokens = $elapsed * $this->refillRate;

        $this->tokens = min($this->capacity, $this->tokens + $newTokens);
        $this->lastRefill = $now;
    }

    private function saveState(): void
    {
        // Save to cache/database for persistence
        $cacheKey = "rate_limiter_{$this->key}";
        Capsule::table('mod_rate_limiter')->updateOrInsert(
            ['limiter_key' => $this->key],
            [
                'tokens' => $this->tokens,
                'last_refill' => $this->lastRefill,
                'updated_at' => date('Y-m-d H:i:s'),
            ]
        );
    }

    public static function load(string $key): ?self
    {
        $data = Capsule::table('mod_rate_limiter')
            ->where('limiter_key', $key)
            ->first();

        if (!$data) {
            return null;
        }

        $limiter = new self($key);
        $limiter->tokens = $data->tokens;
        $limiter->lastRefill = $data->last_refill;

        return $limiter;
    }
}
```

## Service Discovery

### Service Registry

```php
<?php
/**
 * Service registry for dynamic service discovery
 */
class ServiceRegistry
{
    private string $table = 'mod_service_registry';

    /**
     * Register a service instance
     */
    public function register(string $serviceName, string $instanceId, array $metadata): void
    {
        Capsule::table($this->table)->insert([
            'service_name' => $serviceName,
            'instance_id' => $instanceId,
            'host' => $metadata['host'] ?? '',
            'port' => $metadata['port'] ?? 80,
            'metadata' => json_encode($metadata),
            'status' => 'healthy',
            'registered_at' => date('Y-m-d H:i:s'),
            'last_heartbeat' => date('Y-m-d H:i:s'),
        ]);
    }

    /**
     * Deregister a service instance
     */
    public function deregister(string $instanceId): void
    {
        Capsule::table($this->table)
            ->where('instance_id', $instanceId)
            ->update(['status' => 'removed']);
    }

    /**
     * Heartbeat to keep registration alive
     */
    public function heartbeat(string $instanceId): void
    {
        Capsule::table($this->table)
            ->where('instance_id', $instanceId)
            ->update(['last_heartbeat' => date('Y-m-d H:i:s')]);
    }

    /**
     * Discover service instances
     */
    public function discover(string $serviceName, array $criteria = []): array
    {
        $query = Capsule::table($this->table)
            ->where('service_name', $serviceName)
            ->where('status', 'healthy');

        // Filter by metadata criteria
        if (!empty($criteria)) {
            $query->where(function ($q) use ($criteria) {
                foreach ($criteria as $key => $value) {
                    $q->whereRaw("JSON_EXTRACT(metadata, '$.{$key}') = ?", [$value]);
                }
            });
        }

        $instances = $query->get();

        return array_map(function ($instance) {
            return [
                'id' => $instance->instance_id,
                'host' => $instance->host,
                'port' => $instance->port,
                'metadata' => json_decode($instance->metadata, true),
                'url' => "http://{$instance->host}:{$instance->port}",
            ];
        }, $instances);
    }

    /**
     * Clean up stale registrations
     */
    public function cleanup(int $ttlSeconds = 300): int
    {
        $cutoff = date('Y-m-d H:i:s', time() - $ttlSeconds);

        return Capsule::table($this->table)
            ->where('last_heartbeat', '<', $cutoff)
            ->update(['status' => 'unhealthy']);
    }
}
```

## Service Mesh Integration

### WHMCS Service Mesh

```php
<?php
/**
 * WHMCS service mesh manager
 */
class ServiceMeshManager
{
    private SidecarProxy $sidecar;
    private ServiceRegistry $registry;
    private array $config;

    public function __construct(array $config = [])
    {
        $this->config = array_merge([
            'service_name' => 'whmcs',
            'mesh_enabled' => false,
            'upstream_services' => [],
        ], $config);

        $this->sidecar = new SidecarProxy($this->config['service_name']);
        $this->registry = new ServiceRegistry();

        $this->setupUpstreams();
    }

    private function setupUpstreams(): void
    {
        foreach ($this->config['upstream_services'] as $service => $config) {
            $instances = $this->registry->discover($service);

            if (!empty($instances)) {
                $endpoints = array_column($instances, 'url');
                $this->sidecar->registerUpstream($service, $endpoints);
            }
        }
    }

    /**
     * Make a mesh-aware request
     */
    public function request(string $service, string $method, string $path, array $options = []): array
    {
        if (!$this->config['mesh_enabled']) {
            return $this->directRequest($service, $path, $options);
        }

        return $this->sidecar->request($method, $service, $path, $options);
    }

    private function directRequest(string $service, string $path, array $options): array
    {
        $config = $this->config['upstream_services'][$service] ?? [];
        $url = ($config['url'] ?? '') . $path;

        $ch = curl_init();
        curl_setopt_array($ch, [
            CURLOPT_URL => $url,
            CURLOPT_RETURNTRANSFER => true,
            CURLOPT_TIMEOUT => $options['timeout'] ?? 30,
        ]);

        $response = curl_exec($ch);
        $statusCode = curl_getinfo($ch, CURLINFO_HTTP_CODE);
        curl_close($ch);

        return [
            'status_code' => $statusCode,
            'body' => json_decode($response, true) ?? $response,
        ];
    }

    /**
     * Get mesh status
     */
    public function getStatus(): array
    {
        return [
            'enabled' => $this->config['mesh_enabled'],
            'sidecar' => [
                'upstreams' => array_keys($this->config['upstream_services']),
            ],
            'registry' => [
                'services' => $this->getRegisteredServices(),
            ],
        ];
    }

    private function getRegisteredServices(): array
    {
        return Capsule::table('mod_service_registry')
            ->select('service_name')
            ->groupBy('service_name')
            ->get()
            ->pluck('service_name')
            ->toArray();
    }
}
```

### Health Check Integration

```php
<?php
/**
 * Mesh-aware health check
 */
add_hook('AdminAreaPage', 1, function ($vars) {
    $registry = new ServiceRegistry();

    // Cleanup stale registrations
    $registry->cleanup();

    // Register WHMCS instance
    $instanceId = 'whmcs_' . gethostname() . '_' . getmypid();
    $registry->register('whmcs', $instanceId, [
        'host' => $_SERVER['HTTP_HOST'] ?? 'localhost',
        'port' => $_SERVER['SERVER_PORT'] ?? 80,
        'version' => App::getVersion(),
        'url' => $vars['whmcs_url'] ?? '',
    ]);

    // Heartbeat interval (in practice, use cron for heartbeat)
    $_SESSION['mesh_instance_id'] = $instanceId;
});

/**
 * Heartbeat on each page load
 */
add_hook('postAutoload', 1, function ($vars) {
    $instanceId = $_SESSION['mesh_instance_id'] ?? null;

    if ($instanceId) {
        $registry = new ServiceRegistry();
        $registry->heartbeat($instanceId);
    }
});
```

## Best Practices

1. **Use circuit breakers** - Prevent cascading failures
2. **Implement retry policies** - With exponential backoff
3. **Set rate limits** - Protect services from overload
4. **Monitor service health** - Regular heartbeats and cleanup
5. **Use timeouts** - Never wait indefinitely for responses
6. **Balance load** - Distribute traffic across instances
7. **Make services stateless** - Enable horizontal scaling
8. **Document service contracts** - Clear API specifications

## Related Patterns

- [Distributed Tracing](./distributed-tracing.md) - Request tracking
- [Queue Processing](./queue-processing.md) - Async communication
- [API Integration Patterns](./api-integration-patterns.md) - External service calls
