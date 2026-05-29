# WHMCS Advanced API

Complete guide to advanced API techniques.

## Overview

Implement sophisticated API integrations.

## REST API

### Custom REST Endpoint

```php
<?php
/**
 * Custom REST API router
 */
class RESTApiRouter
{
    private array $routes = [];
    
    /**
     * Register route
     */
    public function register(string $method, string $path, callable $handler, array $middleware = []): void
    {
        $this->routes[] = [
            'method' => strtoupper($method),
            'path' => $path,
            'handler' => $handler,
            'middleware' => $middleware,
        ];
    }
    
    /**
     * Handle request
     */
    public function handle(): void
    {
        $method = $_SERVER['REQUEST_METHOD'];
        $path = parse_url($_SERVER['REQUEST_URI'], PHP_URL_PATH);
        
        foreach ($this->routes as $route) {
            if ($route['method'] !== $method) continue;
            
            $params = $this->matchPath($route['path'], $path);
            if ($params === false) continue;
            
            // Run middleware
            foreach ($route['middleware'] as $middleware) {
                $result = $middleware($params);
                if ($result !== true) {
                    $this->sendResponse($result, 403);
                    return;
                }
            }
            
            // Execute handler
            try {
                $result = $route['handler']($params, $this->getRequestBody());
                $this->sendResponse($result);
            } catch (Exception $e) {
                $this->sendResponse(['error' => $e->getMessage()], 500);
            }
            return;
        }
        
        $this->sendResponse(['error' => 'Not Found'], 404);
    }
    
    /**
     * Match path with parameters
     */
    private function matchPath(string $pattern, string $path): array|false
    {
        $pattern = preg_replace('/\{(\w+)\}/', '(?P<$1>[^/]+)', $pattern);
        $pattern = '#^' . $pattern . '$#';
        
        if (preg_match($pattern, $path, $matches)) {
            return array_filter($matches, 'is_string', ARRAY_FILTER_USE_KEY);
        }
        
        return false;
    }
    
    /**
     * Get request body
     */
    private function getRequestBody(): array
    {
        $body = file_get_contents('php://input');
        return json_decode($body, true) ?? [];
    }
    
    /**
     * Send JSON response
     */
    private function sendResponse(array $data, int $status = 200): void
    {
        http_response_code($status);
        header('Content-Type: application/json');
        echo json_encode($data);
    }
}
```

### API Implementation

```php
<?php
/**
 * Initialize REST API
 */
$api = new RESTApiRouter();

// Middleware: API Key authentication
$authMiddleware = function($params) {
    $apiKey = $_SERVER['HTTP_X_API_KEY'] ?? '';
    
    if (empty($apiKey)) {
        return ['error' => 'API key required', 'code' => 'AUTH_REQUIRED'];
    }
    
    $valid = Capsule::table('mod_api_keys')
        ->where('api_key', $apiKey)
        ->where('active', 1)
        ->where('expires_at', '>', date('Y-m-d H:i:s'))
        ->exists();
    
    if (!$valid) {
        return ['error' => 'Invalid API key', 'code' => 'AUTH_FAILED'];
    }
    
    return true;
};

// Middleware: Rate limiting
$rateLimitMiddleware = function($params) {
    $apiKey = $_SERVER['HTTP_X_API_KEY'] ?? '';
    $limiter = new RateLimiter(new RedisCache(['host' => REDIS_HOST, 'port' => REDIS_PORT]));
    
    if (!$limiter->attempt("api:{$apiKey}")) {
        return ['error' => 'Rate limit exceeded', 'code' => 'RATE_LIMITED'];
    }
    
    return true;
};

// Register routes
$api->register('GET', '/api/v1/clients', function($params, $body) {
    return [
        'clients' => Capsule::table('tblclients')
            ->limit(100)
            ->get()
            ->toArray(),
    ];
}, [$authMiddleware, $rateLimitMiddleware]);

$api->register('GET', '/api/v1/clients/{id}', function($params, $body) {
    $client = Capsule::table('tblclients')
        ->where('id', $params['id'])
        ->first();
    
    if (!$client) {
        return ['error' => 'Client not found'];
    }
    
    return ['client' => $client];
}, [$authMiddleware]);

$api->register('POST', '/api/v1/clients', function($params, $body) {
    $clientId = Capsule::table('tblclients')->insertGetId([
        'email' => $body['email'],
        'firstname' => $body['first_name'],
        'lastname' => $body['last_name'],
        'datecreated' => date('Y-m-d'),
        'status' => 'Active',
    ]);
    
    return ['success' => true, 'client_id' => $clientId];
}, [$authMiddleware]);

$api->register('PUT', '/api/v1/clients/{id}', function($params, $body) {
    Capsule::table('tblclients')
        ->where('id', $params['id'])
        ->update($body);
    
    return ['success' => true];
}, [$authMiddleware]);

$api->register('DELETE', '/api/v1/clients/{id}', function($params, $body) {
    Capsule::table('tblclients')
        ->where('id', $params['id'])
        ->delete();
    
    return ['success' => true];
}, [$authMiddleware]);

// Handle the request
$api->handle();
```

## GraphQL API

```php
<?php
/**
 * Simple GraphQL implementation
 */
class GraphQLServer
{
    private array $schema = [];
    private array $resolvers = [];
    
    /**
     * Define schema
     */
    public function schema(string $type, array $fields): void
    {
        $this->schema[$type] = $fields;
    }
    
    /**
     * Define resolver
     */
    public function resolve(string $type, string $field, callable $resolver): void
    {
        $this->resolvers["{$type}.{$field}"] = $resolver;
    }
    
    /**
     * Handle GraphQL request
     */
    public function handle(): void
    {
        $body = json_decode(file_get_contents('php://input'), true);
        
        $query = $body['query'] ?? '';
        $variables = $body['variables'] ?? [];
        
        $result = $this->execute($query, $variables);
        
        header('Content-Type: application/json');
        echo json_encode($result);
    }
    
    /**
     * Execute query
     */
    private function execute(string $query, array $variables): array
    {
        try {
            $ast = $this->parseQuery($query);
            $data = $this->executeOperations($ast['operations'], $variables);
            
            return ['data' => $data];
        } catch (Exception $e) {
            return ['errors' => [['message' => $e->getMessage()]]];
        }
    }
    
    /**
     * Execute operations
     */
    private function executeOperations(array $operations, array $variables): array
    {
        $results = [];
        
        foreach ($operations as $operation) {
            $results[$operation['name']] = $this->executeOperation($operation, $variables);
        }
        
        return $results;
    }
    
    /**
     * Execute single operation
     */
    private function executeOperation(array $operation, array $variables): array
    {
        $type = $operation['type'];
        $selection = $operation['selection'];
        
        if ($type === 'query' && isset($selection['clients'])) {
            return $this->resolveClients($selection['clients']);
        }
        
        return [];
    }
    
    /**
     * Resolve clients query
     */
    private function resolveClients(array $selection): array
    {
        $fields = array_keys($selection);
        
        $clients = Capsule::table('tblclients')
            ->limit(100)
            ->get();
        
        $result = [];
        foreach ($clients as $client) {
            $item = [];
            foreach ($fields as $field) {
                $item[$field] = $client->$field ?? null;
            }
            $result[] = $item;
        }
        
        return $result;
    }
    
    /**
     * Parse simple query
     */
    private function parseQuery(string $query): array
    {
        // Simplified parser
        preg_match_all('/(\w+)\s*\{([^{}]+)\}/', $query, $matches, PREG_SET_ORDER);
        
        $operations = [];
        foreach ($matches as $match) {
            $fields = array_filter(explode(' ', trim($match[2])));
            $operations[] = [
                'type' => 'query',
                'name' => $match[1],
                'selection' => array_fill_keys($fields, true),
            ];
        }
        
        return ['operations' => $operations];
    }
}
```

## Webhook API

```php
<?php
/**
 * Webhook dispatcher
 */
class WebhookDispatcher
{
    private array $handlers = [];
    private int $timeout;
    
    public function __construct(int $timeout = 30)
    {
        $this->timeout = $timeout;
    }
    
    /**
     * Register webhook handler
     */
    public function on(string $event, callable $handler): void
    {
        $this->handlers[$event] = $handler;
    }
    
    /**
     * Dispatch webhook
     */
    public function dispatch(string $event, array $data): array
    {
        if (!isset($this->handlers[$event])) {
            return ['success' => false, 'error' => 'No handler for event'];
        }
        
        $handler = $this->handlers[$event];
        
        // Log the webhook
        $webhookLogId = Capsule::table('mod_webhook_log')->insertGetId([
            'event' => $event,
            'payload' => json_encode($data),
            'status' => 'pending',
            'created_at' => date('Y-m-d H:i:s'),
        ]);
        
        try {
            $result = $handler($data);
            
            Capsule::table('mod_webhook_log')
                ->where('id', $webhookLogId)
                ->update([
                    'status' => 'success',
                    'response' => json_encode($result),
                    'completed_at' => date('Y-m-d H:i:s'),
                ]);
            
            return ['success' => true, 'result' => $result];
            
        } catch (Exception $e) {
            Capsule::table('mod_webhook_log')
                ->where('id', $webhookLogId)
                ->update([
                    'status' => 'failed',
                    'error' => $e->getMessage(),
                    'attempts' => 1,
                ]);
            
            return ['success' => false, 'error' => $e->getMessage()];
        }
    }
    
    /**
     * Send outgoing webhook
     */
    public function send(string $url, string $event, array $data): bool
    {
        $payload = [
            'event' => $event,
            'timestamp' => date('c'),
            'data' => $data,
        ];
        
        $ch = curl_init($url);
        curl_setopt_array($ch, [
            CURLOPT_POST => true,
            CURLOPT_POSTFIELDS => json_encode($payload),
            CURLOPT_RETURNTRANSFER => true,
            CURLOPT_TIMEOUT => $this->timeout,
            CURLOPT_HTTPHEADER => [
                'Content-Type: application/json',
                'X-Webhook-Event: ' . $event,
                'X-Webhook-Signature: ' . $this->generateSignature($payload),
            ],
        ]);
        
        $response = curl_exec($ch);
        $httpCode = curl_getinfo($ch, CURLINFO_HTTP_CODE);
        curl_close($ch);
        
        return $httpCode >= 200 && $httpCode < 300;
    }
    
    /**
     * Generate webhook signature
     */
    private function generateSignature(array $payload): string
    {
        $secret = getWebhookSecret();
        return base64_encode(hash_hmac('sha256', json_encode($payload), $secret, true));
    }
}
```

## API Documentation

```php
<?php
/**
 * API documentation generator
 */
class APIDocumentation
{
    private array $endpoints = [];
    
    /**
     * Add endpoint documentation
     */
    public function addEndpoint(string $method, string $path, array $doc): void
    {
        $this->endpoints[] = [
            'method' => $method,
            'path' => $path,
            'summary' => $doc['summary'] ?? '',
            'description' => $doc['description'] ?? '',
            'parameters' => $doc['parameters'] ?? [],
            'responses' => $doc['responses'] ?? [],
        ];
    }
    
    /**
     * Generate OpenAPI spec
     */
    public function generateOpenAPI(): array
    {
        $spec = [
            'openapi' => '3.0.0',
            'info' => [
                'title' => 'WHMCS Custom API',
                'version' => '1.0.0',
            ],
            'paths' => [],
        ];
        
        foreach ($this->endpoints as $endpoint) {
            $path = $endpoint['path'];
            
            if (!isset($spec['paths'][$path])) {
                $spec['paths'][$path] = [];
            }
            
            $spec['paths'][$path][strtolower($endpoint['method'])] = [
                'summary' => $endpoint['summary'],
                'description' => $endpoint['description'],
                'parameters' => array_map(function($param) {
                    return [
                        'name' => $param['name'],
                        'in' => $param['in'] ?? 'query',
                        'required' => $param['required'] ?? false,
                        'schema' => ['type' => $param['type'] ?? 'string'],
                    ];
                }, $endpoint['parameters']),
                'responses' => $endpoint['responses'],
            ];
        }
        
        return $spec;
    }
}
```

## Best Practices

1. **Use REST conventions** - Follow REST principles
2. **Version your API** - /api/v1/, /api/v2/
3. **Authenticate properly** - API keys, OAuth
4. **Rate limit** - Prevent abuse
5. **Document thoroughly** - OpenAPI specs
6. **Return proper codes** - 200, 400, 401, 404, 500

## Related Documentation

- [whmcs-integration-api.md](whmcs-integration-api.md)
- [whmcs-advanced-performance.md](whmcs-advanced-performance.md)
