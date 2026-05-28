# API Gateway Patterns

API gateway patterns provide a unified entry point for managing, securing, and routing API requests in WHMCS integrations.

## Gateway Architecture

### Gateway Router

```php
<?php
/**
 * API Gateway router
 */
class ApiGatewayRouter
{
    private array $routes = [];
    private MiddlewarePipeline $pipeline;

    public function __construct()
    {
        $this->pipeline = new MiddlewarePipeline();
    }

    /**
     * Register a route
     */
    public function register(string $method, string $pattern, callable $handler, array $options = []): self
    {
        $this->routes[] = [
            'method' => strtoupper($method),
            'pattern' => $this->compilePattern($pattern),
            'handler' => $handler,
            'middleware' => $options['middleware'] ?? [],
            'rate_limit' => $options['rate_limit'] ?? null,
            'auth' => $options['auth'] ?? 'none',
        ];

        return $this;
    }

    /**
     * Handle incoming request
     */
    public function handle(ServerRequest $request): Response
    {
        $matched = $this->matchRoute($request);

        if (!$matched) {
            return new Response(404, ['Content-Type' => 'application/json'], json_encode([
                'error' => 'Not Found',
                'message' => 'No route matches the requested URI',
            ]));
        }

        // Build middleware stack
        $stack = $this->pipeline;

        foreach ($matched['middleware'] as $middleware) {
            $stack->pipe($middleware);
        }

        // Add route handler
        $stack->pipe(new HandlerMiddleware($matched['handler']));

        return $stack->handle($request);
    }

    private function matchRoute(ServerRequest $request): ?array
    {
        $method = $request->getMethod();
        $path = $request->getPath();

        foreach ($this->routes as $route) {
            if ($route['method'] !== $method && $route['method'] !== '*') {
                continue;
            }

            if (preg_match($route['pattern'], $path, $matches)) {
                // Extract named parameters
                $params = array_filter($matches, 'is_string', ARRAY_FILTER_USE_KEY);
                $request = $request->withAttribute('params', $params);

                return $route;
            }
        }

        return null;
    }

    private function compilePattern(string $pattern): string
    {
        // Convert {param} to named regex groups
        $pattern = preg_replace('/\{([a-zA-Z_]+)\}/', '(?P<$1>[^/]+)', $pattern);
        $pattern = str_replace('/', '\/', $pattern);

        return '/^' . $pattern . '$/';
    }
}

/**
 * Simple middleware pipeline
 */
class MiddlewarePipeline
{
    private array $middleware = [];
    private int $index = 0;

    public function pipe($middleware): self
    {
        $this->middleware[] = $middleware;
        return $this;
    }

    public function handle(ServerRequest $request): Response
    {
        if ($this->index >= count($this->middleware)) {
            throw new RuntimeException('No more middleware to handle request');
        }

        $middleware = $this->middleware[$this->index++];

        return $middleware->handle($request, [$this, 'handle']);
    }
}
```

### Request/Response Objects

```php
<?php
/**
 * Simple PSR-7 inspired request object
 */
class ServerRequest
{
    private string $method;
    private string $path;
    private array $query;
    private array $body;
    private array $headers;
    private array $attributes = [];

    public function __construct(
        string $method = 'GET',
        string $path = '/',
        array $query = [],
        array $body = [],
        array $headers = []
    ) {
        $this->method = $method;
        $this->path = $path;
        $this->query = $query;
        $this->body = $body;
        $this->headers = array_change_key_case($headers, CASE_LOWER);
    }

    public static function fromGlobals(): self
    {
        return new self(
            $_SERVER['REQUEST_METHOD'] ?? 'GET',
            parse_url($_SERVER['REQUEST_URI'] ?? '/', PHP_URL_PATH),
            $_GET,
            $_POST,
            getallheaders()
        );
    }

    public function getMethod(): string
    {
        return $this->method;
    }

    public function getPath(): string
    {
        return $this->path;
    }

    public function getQuery(string $key, $default = null)
    {
        return $this->query[$key] ?? $default;
    }

    public function getBody(string $key, $default = null)
    {
        return $this->body[$key] ?? $default;
    }

    public function getHeader(string $key, $default = null)
    {
        return $this->headers[strtolower($key)] ?? $default;
    }

    public function getAttribute(string $key, $default = null)
    {
        return $this->attributes[$key] ?? $default;
    }

    public function withAttribute(string $key, $value): self
    {
        $clone = clone $this;
        $clone->attributes[$key] = $value;
        return $clone;
    }

    public function all(): array
    {
        return [
            'method' => $this->method,
            'path' => $this->path,
            'query' => $this->query,
            'body' => $this->body,
            'headers' => $this->headers,
            'attributes' => $this->attributes,
        ];
    }
}

/**
 * Simple response object
 */
class Response
{
    private int $statusCode;
    private array $headers;
    private string $body;

    public function __construct(int $statusCode = 200, array $headers = [], string $body = '')
    {
        $this->statusCode = $statusCode;
        $this->headers = $headers;
        $this->body = $body;
    }

    public function getStatusCode(): int
    {
        return $this->statusCode;
    }

    public function getHeaders(): array
    {
        return $this->headers;
    }

    public function getBody(): string
    {
        return $this->body;
    }

    public static function json(array $data, int $statusCode = 200): self
    {
        return new self($statusCode, [
            'Content-Type' => 'application/json',
        ], json_encode($data));
    }

    public static function error(string $message, int $statusCode = 400): self
    {
        return self::json(['error' => $message], $statusCode);
    }

    public function send(): void
    {
        http_response_code($this->statusCode);

        foreach ($this->headers as $name => $value) {
            header("{$name}: {$value}");
        }

        echo $this->body;
    }
}
```

## Authentication Middleware

### API Key Authentication

```php
<?php
/**
 * API Key authentication middleware
 */
class ApiKeyAuthMiddleware
{
    private array $validKeys;
    private string $headerName;

    public function __construct(array $validKeys = [], string $headerName = 'X-API-Key')
    {
        $this->validKeys = $validKeys;
        $this->headerName = $headerName;
    }

    public function handle(ServerRequest $request, callable $next): Response
    {
        $apiKey = $request->getHeader($this->headerName);

        if (!$apiKey) {
            return Response::error('API key required', 401);
        }

        if (!in_array($apiKey, $this->validKeys)) {
            return Response::error('Invalid API key', 401);
        }

        return $next($request);
    }
}

/**
 * JWT authentication middleware
 */
class JwtAuthMiddleware
{
    private string $secret;
    private string $algorithm;

    public function __construct(string $secret, string $algorithm = 'HS256')
    {
        $this->secret = $secret;
        $this->algorithm = $algorithm;
    }

    public function handle(ServerRequest $request, callable $next): Response
    {
        $authHeader = $request->getHeader('Authorization');

        if (!$authHeader) {
            return Response::error('Authorization header required', 401);
        }

        if (!str_starts_with($authHeader, 'Bearer ')) {
            return Response::error('Invalid authorization format', 401);
        }

        $token = substr($authHeader, 7);

        try {
            $payload = $this->decodeToken($token);
            $request = $request->withAttribute('user', $payload);
        } catch (\Throwable $e) {
            return Response::error('Invalid token: ' . $e->getMessage(), 401);
        }

        return $next($request);
    }

    private function decodeToken(string $token): array
    {
        $parts = explode('.', $token);

        if (count($parts) !== 3) {
            throw new InvalidArgumentException('Invalid token format');
        }

        [$header, $payload, $signature] = $parts;

        // Verify signature
        $expectedSignature = $this->base64UrlEncode(
            hash_hmac('sha256', "{$header}.{$payload}", $this->secret, true)
        );

        if (!hash_equals($expectedSignature, $signature)) {
            throw new InvalidArgumentException('Invalid signature');
        }

        $payloadData = json_decode($this->base64UrlDecode($payload), true);

        // Check expiration
        if (isset($payloadData['exp']) && $payloadData['exp'] < time()) {
            throw new InvalidArgumentException('Token expired');
        }

        return $payloadData;
    }

    private function base64UrlEncode(string $data): string
    {
        return rtrim(strtr(base64_encode($data), '+/', '-_'), '=');
    }

    private function base64UrlDecode(string $data): string
    {
        return base64_decode(strtr($data, '-_', '+/'));
    }
}
```

### Rate Limiting Middleware

```php
<?php
<?php
/**
 * Rate limiting middleware
 */
class RateLimitMiddleware
{
    private RateLimiter $limiter;
    private int $maxRequests;
    private int $windowSeconds;

    public function __construct(
        RateLimiter $limiter,
        int $maxRequests = 100,
        int $windowSeconds = 60
    ) {
        $this->limiter = $limiter;
        $this->maxRequests = $maxRequests;
        $this->windowSeconds = $windowSeconds;
    }

    public function handle(ServerRequest $request, callable $next): Response
    {
        $identifier = $this->getIdentifier($request);
        $key = "rate_limit_{$identifier}";

        $limiter = RateLimiter::load($key);

        if (!$limiter) {
            $limiter = new RateLimiter($key, $this->maxRequests, $this->maxRequests / $this->windowSeconds);
        }

        if (!$limiter->allow()) {
            $waitTime = $limiter->getWaitTime();

            return new Response(429, [
                'Content-Type' => 'application/json',
                'Retry-After' => (string) ceil($waitTime),
                'X-RateLimit-Remaining' => '0',
                'X-RateLimit-Limit' => (string) $this->maxRequests,
            ], json_encode([
                'error' => 'Too Many Requests',
                'message' => 'Rate limit exceeded. Please retry later.',
                'retry_after' => (int) ceil($waitTime),
            ]));
        }

        $response = $next($request);

        // Add rate limit headers
        $headers = $response->getHeaders();
        $headers['X-RateLimit-Remaining'] = (string) floor($limiter->getAvailableTokens());
        $headers['X-RateLimit-Limit'] = (string) $this->maxRequests;

        return new Response(
            $response->getStatusCode(),
            $headers,
            $response->getBody()
        );
    }

    private function getIdentifier(ServerRequest $request): string
    {
        // Use API key if available
        $apiKey = $request->getHeader('X-API-Key');

        if ($apiKey) {
            return hash('sha256', $apiKey);
        }

        // Fall back to IP address
        return $request->getHeader('X-Forwarded-For') ?? $_SERVER['REMOTE_ADDR'] ?? 'unknown';
    }
}
```

## Gateway Implementation

### WHMCS API Gateway

```php
<?php
<?php
/**
 * WHMCS API Gateway
 */
class WhmcsApiGateway
{
    private ApiGatewayRouter $router;
    private ApiKeyAuthMiddleware $auth;
    private ServiceContainer $container;

    public function __construct()
    {
        $this->router = new ApiGatewayRouter();
        $this->container = new ServiceContainer();
        $this->setupRoutes();
    }

    private function setupRoutes(): void
    {
        // Load API keys from config
        $apiKeys = $this->loadApiKeys();

        // Add authentication
        $this->router->register('*', '/api/{path}', function ($request) {
            return Response::error('Authentication required', 401);
        }, [
            'middleware' => [
                new ApiKeyAuthMiddleware($apiKeys),
            ],
        ]);

        // Client endpoints
        $this->router->register('GET', '/api/clients', [$this, 'getClients'], [
            'auth' => 'api_key',
            'rate_limit' => ['max' => 100, 'window' => 60],
        ]);

        $this->router->register('GET', '/api/clients/{id}', [$this, 'getClient'], [
            'auth' => 'api_key',
            'rate_limit' => ['max' => 100, 'window' => 60],
        ]);

        $this->router->register('POST', '/api/clients', [$this, 'createClient'], [
            'auth' => 'api_key',
            'rate_limit' => ['max' => 50, 'window' => 60],
        ]);

        $this->router->register('PUT', '/api/clients/{id}', [$this, 'updateClient'], [
            'auth' => 'api_key',
            'rate_limit' => ['max' => 50, 'window' => 60],
        ]);

        // Invoice endpoints
        $this->router->register('GET', '/api/invoices', [$this, 'getInvoices'], [
            'auth' => 'api_key',
        ]);

        $this->router->register('GET', '/api/invoices/{id}', [$this, 'getInvoice'], [
            'auth' => 'api_key',
        ]);

        // Order endpoints
        $this->router->register('POST', '/api/orders', [$this, 'createOrder'], [
            'auth' => 'api_key',
            'rate_limit' => ['max' => 30, 'window' => 60],
        ]);

        // Service endpoints
        $this->router->register('GET', '/api/services', [$this, 'getServices'], [
            'auth' => 'api_key',
        ]);

        $this->router->register('POST', '/api/services/{id}/suspend', [$this, 'suspendService'], [
            'auth' => 'api_key',
        ]);

        $this->router->register('POST', '/api/services/{id}/unsuspend', [$this, 'unsuspendService'], [
            'auth' => 'api_key',
        ]);

        $this->router->register('POST', '/api/services/{id}/terminate', [$this, 'terminateService'], [
            'auth' => 'api_key',
        ]);
    }

    private function loadApiKeys(): array
    {
        $keys = Capsule::table('mod_api_keys')
            ->where('active', 1)
            ->where('expires_at', '>', date('Y-m-d H:i:s'))
            ->get()
            ->pluck('api_key')
            ->toArray();

        return $keys;
    }

    public function handleRequest(ServerRequest $request): Response
    {
        return $this->router->handle($request);
    }

    // Controller methods

    public function getClients(ServerRequest $request): Response
    {
        $params = $request->getAttribute('params', []);
        $limit = (int) $request->getQuery('limit', 50);
        $offset = (int) $request->getQuery('offset', 0);

        $clients = Capsule::table('tblclients')
            ->limit($limit)
            ->offset($offset)
            ->orderBy('id', 'desc')
            ->get();

        return Response::json([
            'data' => $clients,
            'limit' => $limit,
            'offset' => $offset,
        ]);
    }

    public function getClient(ServerRequest $request): Response
    {
        $id = (int) $request->getAttribute('params')['id'];

        $client = Capsule::table('tblclients')->where('id', $id)->first();

        if (!$client) {
            return Response::error('Client not found', 404);
        }

        return Response::json(['data' => $client]);
    }

    public function createClient(ServerRequest $request): Response
    {
        $data = $request->body;

        $required = ['firstname', 'lastname', 'email'];

        foreach ($required as $field) {
            if (empty($data[$field])) {
                return Response::error("Field '{$field}' is required", 400);
            }
        }

        $id = Capsule::table('tblclients')->insertGetId([
            'firstname' => $data['firstname'],
            'lastname' => $data['lastname'],
            'email' => $data['email'],
            'companyname' => $data['companyname'] ?? '',
            'created_at' => date('Y-m-d H:i:s'),
        ]);

        return Response::json(['data' => ['id' => $id]], 201);
    }

    public function getInvoices(ServerRequest $request): Response
    {
        $clientId = $request->getQuery('client_id');

        $query = Capsule::table('tblinvoices');

        if ($clientId) {
            $query->where('userid', (int) $clientId);
        }

        $invoices = $query->orderBy('id', 'desc')->limit(50)->get();

        return Response::json(['data' => $invoices]);
    }

    public function createOrder(ServerRequest $request): Response
    {
        $data = $request->body;

        // Validate order data
        if (empty($data['client_id']) || empty($data['items'])) {
            return Response::error('client_id and items are required', 400);
        }

        // Create order through WHMCS API
        $result = localAPI('AddOrder', [
            'client_id' => $data['client_id'],
            'pid' => is_array($data['items']) ? implode(',', $data['items']) : $data['items'],
            'billingcycle' => $data['billing_cycle'] ?? 'monthly',
        ]);

        if ($result['result'] !== 'success') {
            return Response::error($result['message'] ?? 'Order creation failed', 400);
        }

        return Response::json([
            'data' => [
                'order_id' => $result['orderid'],
            ],
        ], 201);
    }
}
```

### Gateway Hook

```php
<?php
/**
 * API Gateway hook
 */
add_hook('ApiStart', 1, function ($vars) {
    // Check if this is a gateway request
    if (defined('WHMCS_GATEWAY_REQUEST')) {
        $gateway = new WhmcsApiGateway();
        $request = ServerRequest::fromGlobals();
        $response = $gateway->handleRequest($request);
        $response->send();
        exit;
    }
});

/**
 * Route incoming gateway requests
 */
add_hook('preAutoload', 1, function ($vars) {
    // Check for gateway prefix
    $uri = $_SERVER['REQUEST_URI'] ?? '';

    if (str_starts_with($uri, '/gateway/')) {
        define('WHMCS_GATEWAY_REQUEST', true);
    }
});
```

## Best Practices

1. **Use middleware composably** - Stack authentication, logging, rate limiting
2. **Implement versioning** - /api/v1/, /api/v2/ for backward compatibility
3. **Validate input strictly** - Reject malformed requests early
4. **Return consistent errors** - Standard error format across all endpoints
5. **Log all requests** - Track usage and debug issues
6. **Use appropriate timeouts** - Don't wait indefinitely for backends
7. **Implement circuit breakers** - Handle downstream failures gracefully
8. **Document your API** - OpenAPI/Swagger specifications

## Related Patterns

- [Middleware Patterns](./middleware-patterns.md) - Middleware implementation
- [API Integration Patterns](./api-integration-patterns.md) - External API calls
- [Rate Limiting](./queue-processing.md) - Rate limiting strategies
