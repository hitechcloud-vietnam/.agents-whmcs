# WHMCS Custom API DevKit
# Version: 1.0 | Updated: 2026-05-28

## DevKit Structure

```
devkits/whmcs-custom-api/
├── api-server.php          # Main API server
├── lib/
│   ├── Router.php          # Route handling
│   ├── Middleware.php      # Request middleware
│   ├── Controllers/
│   │   ├── BaseController.php
│   │   ├── ClientsController.php
│   │   ├── ServicesController.php
│   │   ├── InvoicesController.php
│   │   └── DomainsController.php
│   └── Validators/
│       └── RequestValidator.php
├── routes/
│   └── api-routes.php     # Route definitions
└── templates/
    └── api-docs.tpl       # API documentation
```

## Main API Server

```php
<?php
/**
 * WHMCS Custom REST API Server
 * DevKit Template
 * 
 * Provides custom REST API endpoints for WHMCS
 */

if (!defined("WHMCS")) {
    die("Direct access denied");
}

use WHMCS\Database\Capsule;

/**
 * Config function
 */
function {module}_config(): array {
    return [
        'name' => '{Custom API}',
        'description' => 'Custom REST API with CRUD operations',
        'version' => '1.0',
        'author' => '{Author}',
    ];
}

/**
 * Activate
 */
function {module}_activate(): array {
    Capsule::schema()->create('mod_{module}_api_keys', function($t) {
        $t->increments('id');
        $t->string('name');
        $t->string('api_key', 64)->unique();
        $t->string('api_secret', 64);
        $t->text('scopes');
        $t->boolean('is_active');
        $t->string('ip_whitelist')->nullable();
        $t->integer('rate_limit')->default(100);
        $t->timestamp('last_used')->nullable();
        $t->timestamp('created_at');
    });
    
    Capsule::schema()->create('mod_{module}_api_logs', function($t) {
        $t->increments('id');
        $t->integer('api_key_id');
        $t->string('method');
        $t->string('endpoint');
        $t->text('request_body');
        $t->integer('response_code');
        $t->float('execution_time');
        $t->string('ip_address', 45);
        $t->timestamp('created_at');
    });
    
    Capsule::schema()->create('mod_{module}_endpoints', function($t) {
        $t->increments('id');
        $t->string('method');
        $t->string('path');
        $t->string('controller');
        $t->text('description');
        $t->text('parameters');
        $t->boolean('is_active');
        $t->timestamp('created_at');
    });
    
    // Insert default endpoints
    {module}_registerDefaultEndpoints();
    
    return ['status' => 'success', 'description' => 'Custom API activated'];
}

/**
 * Deactivate
 */
function {module}_deactivate(): array {
    Capsule::schema()->dropIfExists('mod_{module}_api_keys');
    Capsule::schema()->dropIfExists('mod_{module}_api_logs');
    Capsule::schema()->dropIfExists('mod_{module}_endpoints');
    
    return ['status' => 'success'];
}

/**
 * Register default endpoints
 */
function {module}_registerDefaultEndpoints(): void {
    $endpoints = [
        ['GET', '/clients', 'ClientsController@index', 'List all clients'],
        ['GET', '/clients/{id}', 'ClientsController@show', 'Get client details'],
        ['POST', '/clients', 'ClientsController@store', 'Create new client'],
        ['PUT', '/clients/{id}', 'ClientsController@update', 'Update client'],
        ['DELETE', '/clients/{id}', 'ClientsController@destroy', 'Delete client'],
        ['GET', '/services', 'ServicesController@index', 'List all services'],
        ['GET', '/services/{id}', 'ServicesController@show', 'Get service details'],
        ['POST', '/services', 'ServicesController@store', 'Create service'],
        ['PUT', '/services/{id}', 'ServicesController@update', 'Update service'],
        ['DELETE', '/services/{id}', 'ServicesController@destroy', 'Delete service'],
        ['GET', '/invoices', 'InvoicesController@index', 'List all invoices'],
        ['GET', '/invoices/{id}', 'InvoicesController@show', 'Get invoice details'],
        ['POST', '/invoices', 'InvoicesController@store', 'Create invoice'],
        ['PUT', '/invoices/{id}/paid', 'InvoicesController@markPaid', 'Mark invoice as paid'],
        ['GET', '/domains', 'DomainsController@index', 'List all domains'],
        ['GET', '/domains/{id}', 'DomainsController@show', 'Get domain details'],
        ['POST', '/domains', 'DomainsController@store', 'Register domain'],
    ];
    
    foreach ($endpoints as $endpoint) {
        Capsule::table('mod_{module}_endpoints')->insert([
            'method' => $endpoint[0],
            'path' => $endpoint[1],
            'controller' => $endpoint[2],
            'description' => $endpoint[3],
            'parameters' => json_encode([]),
            'is_active' => 1,
            'created_at' => date('Y-m-d H:i:s'),
        ]);
    }
}

/**
 * API Entry Point
 */
function {module}_api(): void {
    // Set headers
    header('Content-Type: application/json');
    header('X-API-Version: 1.0');
    header('Access-Control-Allow-Origin: *');
    header('Access-Control-Allow-Methods: GET, POST, PUT, DELETE, OPTIONS');
    header('Access-Control-Allow-Headers: Content-Type, Authorization, X-API-Key');
    
    // Handle preflight
    if ($_SERVER['REQUEST_METHOD'] === 'OPTIONS') {
        http_response_code(200);
        exit;
    }
    
    // Start timing
    $startTime = microtime(true);
    
    try {
        // Authenticate
        $middleware = new \CustomApi\Middleware();
        $authResult = $middleware->authenticate();
        
        // Parse request
        $method = $_SERVER['REQUEST_METHOD'];
        $path = isset($_SERVER['REQUEST_URI']) ? parse_url($_SERVER['REQUEST_URI'], PHP_URL_PATH) : '/';
        $path = preg_replace('#^/modules/addons/{module}/api\.php#', '', $path) ?: '/';
        
        // Get request body
        $requestBody = [];
        if (in_array($method, ['POST', 'PUT', 'PATCH'])) {
            $input = file_get_contents('php://input');
            $requestBody = json_decode($input, true) ?? [];
        }
        
        // Merge GET and POST params
        $params = array_merge($_GET, $_POST, $requestBody);
        
        // Route request
        $router = new \CustomApi\Router();
        $result = $router->dispatch($method, $path, $params, $authResult);
        
        // Log request
        $executionTime = (microtime(true) - $startTime) * 1000;
        {module}_logRequest($authResult['key_id'] ?? 0, $method, $path, $requestBody, 200, $executionTime);
        
        // Send response
        http_response_code(200);
        echo json_encode($result, JSON_PRETTY_PRINT);
        
    } catch (\Exception $e) {
        $executionTime = (microtime(true) - $startTime) * 1000;
        {module}_logRequest(0, $_SERVER['REQUEST_METHOD'] ?? 'GET', $path ?? '/', [], $e->getCode() ?: 500, $executionTime);
        
        http_response_code($e->getCode() ?: 500);
        echo json_encode([
            'error' => true,
            'message' => $e->getMessage(),
        ]);
    }
    
    exit;
}

/**
 * Log API request
 */
function {module}_logRequest(int $keyId, string $method, string $endpoint, array $body, int $responseCode, float $executionTime): void {
    Capsule::table('mod_{module}_api_logs')->insert([
        'api_key_id' => $keyId,
        'method' => $method,
        'endpoint' => $endpoint,
        'request_body' => json_encode($body),
        'response_code' => $responseCode,
        'execution_time' => round($executionTime, 2),
        'ip_address' => $_SERVER['REMOTE_ADDR'] ?? '0.0.0.0',
        'created_at' => date('Y-m-d H:i:s'),
    ]);
}

/**
 * Output function (Admin Interface)
 */
function {module}_output(array $vars): void {
    $action = $_REQUEST['action'] ?? 'dashboard';
    
    if ($_SERVER['REQUEST_METHOD'] === 'POST') {
        check_token('WHMCS.admin.default');
    }
    
    switch ($action) {
        case 'keys':
            {module}_manageApiKeys();
            break;
        case 'endpoints':
            {module}_manageEndpoints();
            break;
        case 'logs':
            {module}_viewLogs();
            break;
        case 'docs':
            {module}_showDocs();
            break;
        default:
            {module}_showDashboard();
    }
}
```

## Router Class

```php
<?php
/**
 * API Router
 */

namespace CustomApi;

use WHMCS\Database\Capsule;

class Router {
    
    private array $routes = [];
    private array $middleware = [];
    
    public function __construct() {
        $this->loadRoutes();
    }
    
    /**
     * Load routes from database
     */
    private function loadRoutes(): void {
        $endpoints = Capsule::table('mod_{module}_endpoints')
            ->where('is_active', 1)
            ->get();
        
        foreach ($endpoints as $endpoint) {
            $this->routes[$endpoint->method][] = [
                'path' => $endpoint->path,
                'controller' => $endpoint->controller,
                'description' => $endpoint->description,
            ];
        }
    }
    
    /**
     * Dispatch request to appropriate controller
     */
    public function dispatch(string $method, string $path, array $params, array $auth): array {
        $method = strtoupper($method);
        
        if (!isset($this->routes[$method])) {
            throw new \Exception('Method not allowed', 405);
        }
        
        foreach ($this->routes[$method] as $route) {
            $params = $this->matchRoute($route['path'], $path, $params);
            
            if ($params !== false) {
                return $this->callController($route['controller'], $params, $auth);
            }
        }
        
        throw new \Exception('Endpoint not found', 404);
    }
    
    /**
     * Match route pattern to path
     */
    private function matchRoute(string $pattern, string $path, array $params): array|false {
        // Convert pattern to regex
        $regex = preg_replace('/\{([a-zA-Z_]+)\}/', '(?P<$1>[^/]+)', $pattern);
        $regex = '#^' . $regex . '$#';
        
        if (preg_match($regex, $path, $matches)) {
            // Extract named parameters
            foreach ($matches as $key => $value) {
                if (is_string($key)) {
                    $params[$key] = $value;
                }
            }
            return $params;
        }
        
        return false;
    }
    
    /**
     * Call controller method
     */
    private function callController(string $controllerAction, array $params, array $auth): array {
        [$controller, $action] = explode('@', $controllerAction);
        
        $controllerClass = "\\CustomApi\\Controllers\\{$controller}";
        
        if (!class_exists($controllerClass)) {
            throw new \Exception("Controller not found: {$controller}", 500);
        }
        
        $instance = new $controllerClass();
        
        if (!method_exists($instance, $action)) {
            throw new \Exception("Method not found: {$action}", 500);
        }
        
        return $instance->$action($params, $auth);
    }
    
    /**
     * Add custom route
     */
    public function addRoute(string $method, string $path, string $controller): void {
        $this->routes[$method][] = [
            'path' => $path,
            'controller' => $controller,
        ];
    }
}
```

## Middleware Class

```php
<?php
/**
 * API Middleware
 */

namespace CustomApi;

use WHMCS\Database\Capsule;

class Middleware {
    
    /**
     * Authenticate API request
     */
    public function authenticate(): array {
        // Try X-API-Key header
        $apiKey = $_SERVER['HTTP_X_API_KEY'] ?? '';
        
        // Try Bearer token
        if (empty($apiKey)) {
            $authHeader = $_SERVER['HTTP_AUTHORIZATION'] ?? '';
            if (preg_match('/Bearer\s+(.+)/i', $authHeader, $matches)) {
                $apiKey = $matches[1];
            }
        }
        
        if (empty($apiKey)) {
            throw new \Exception('API key required', 401);
        }
        
        // Look up API key
        $key = Capsule::table('mod_{module}_api_keys')
            ->where('api_key', $apiKey)
            ->where('is_active', 1)
            ->first();
        
        if (!$key) {
            throw new \Exception('Invalid API key', 401);
        }
        
        // Check IP whitelist
        if (!empty($key->ip_whitelist)) {
            $allowedIPs = array_filter(array_map('trim', explode(',', $key->ip_whitelist)));
            $clientIP = $this->getClientIp();
            
            if (!empty($allowedIPs) && !in_array($clientIP, $allowedIPs)) {
                throw new \Exception('IP address not allowed', 403);
            }
        }
        
        // Update last used
        Capsule::table('mod_{module}_api_keys')
            ->where('id', $key->id)
            ->update(['last_used' => date('Y-m-d H:i:s')]);
        
        return [
            'key_id' => $key->id,
            'name' => $key->name,
            'scopes' => json_decode($key->scopes, true) ?? [],
        ];
    }
    
    /**
     * Check if request has scope
     */
    public function hasScope(array $auth, string $scope): bool {
        $scopes = $auth['scopes'] ?? [];
        return in_array('*', $scopes) || in_array($scope, $scopes);
    }
    
    /**
     * Get client IP
     */
    private function getClientIp(): string {
        $keys = ['HTTP_CF_CONNECTING_IP', 'HTTP_X_FORWARDED_FOR', 'REMOTE_ADDR'];
        
        foreach ($keys as $key) {
            if (!empty($_SERVER[$key])) {
                $ip = $_SERVER[$key];
                if (strpos($ip, ',') !== false) {
                    $ip = trim(explode(',', $ip)[0]);
                }
                return $ip;
            }
        }
        
        return '0.0.0.0';
    }
    
    /**
     * Rate limiting check
     */
    public function checkRateLimit(int $keyId): void {
        $key = Capsule::table('mod_{module}_api_keys')
            ->where('id', $keyId)
            ->first();
        
        if (!$key) {
            return;
        }
        
        $limit = $key->rate_limit;
        $windowStart = date('Y-m-d H:i:00');
        $windowEnd = date('Y-m-d H:i:59');
        
        $requests = Capsule::table('mod_{module}_api_logs')
            ->where('api_key_id', $keyId)
            ->whereBetween('created_at', [$windowStart, $windowEnd])
            ->count();
        
        if ($requests >= $limit) {
            throw new \Exception('Rate limit exceeded. Try again later.', 429);
        }
    }
}
```

## Base Controller

```php
<?php
/**
 * Base API Controller
 */

namespace CustomApi\Controllers;

use WHMCS\Database\Capsule;
use CustomApi\Middleware;

abstract class BaseController {
    
    protected Middleware $middleware;
    
    public function __construct() {
        $this->middleware = new Middleware();
    }
    
    /**
     * Return success response
     */
    protected function success(array $data, int $statusCode = 200): array {
        return [
            'success' => true,
            'status_code' => $statusCode,
            'data' => $data,
        ];
    }
    
    /**
     * Return error response
     */
    protected function error(string $message, int $statusCode = 400): array {
        throw new \Exception($message, $statusCode);
    }
    
    /**
     * Paginate results
     */
    protected function paginate($query, int $page = 1, int $perPage = 20): array {
        $page = max(1, $page);
        $perPage = min(100, max(1, $perPage));
        
        $total = $query->count();
        $offset = ($page - 1) * $perPage;
        
        $items = $query->limit($perPage)->offset($offset)->get();
        
        return [
            'data' => $items->toArray(),
            'meta' => [
                'current_page' => $page,
                'per_page' => $perPage,
                'total' => $total,
                'total_pages' => ceil($total / $perPage),
            ],
        ];
    }
    
    /**
     * Validate required fields
     */
    protected function validateRequired(array $data, array $fields): void {
        foreach ($fields as $field) {
            if (!isset($data[$field]) || $data[$field] === '') {
                throw new \Exception("Field '{$field}' is required", 400);
            }
        }
    }
    
    /**
     * Sanitize input
     */
    protected function sanitize(string $input): string {
        return htmlspecialchars(trim($input), ENT_QUOTES, 'UTF-8');
    }
}
```

## Clients Controller

```php
<?php
/**
 * Clients API Controller
 */

namespace CustomApi\Controllers;

class ClientsController extends BaseController {
    
    /**
     * List clients
     */
    public function index(array $params, array $auth): array {
        $query = Capsule::table('tblclients');
        
        // Filters
        if (!empty($params['search'])) {
            $search = $params['search'];
            $query->where(function($q) use ($search) {
                $q->where('firstname', 'LIKE', "%{$search}%")
                  ->orWhere('lastname', 'LIKE', "%{$search}%")
                  ->orWhere('email', 'LIKE', "%{$search}%")
                  ->orWhere('companyname', 'LIKE', "%{$search}%");
            });
        }
        
        if (!empty($params['status'])) {
            $query->where('status', $params['status']);
        }
        
        // Sorting
        $sortField = $params['sort'] ?? 'id';
        $sortOrder = $params['order'] ?? 'DESC';
        $query->orderBy($sortField, $sortOrder);
        
        // Pagination
        $page = (int) ($params['page'] ?? 1);
        $perPage = (int) ($params['per_page'] ?? 20);
        
        return $this->paginate($query, $page, $perPage);
    }
    
    /**
     * Get single client
     */
    public function show(array $params, array $auth): array {
        $id = (int) ($params['id'] ?? 0);
        
        if (!$id) {
            return $this->error('Client ID required', 400);
        }
        
        $client = Capsule::table('tblclients')->where('id', $id)->first();
        
        if (!$client) {
            return $this->error('Client not found', 404);
        }
        
        // Include related data
        $client = (array) $client;
        $client['services'] = Capsule::table('tblhosting')
            ->where('userid', $id)
            ->get()
            ->toArray();
        
        $client['domains'] = Capsule::table('tbldomains')
            ->where('userid', $id)
            ->get()
            ->toArray();
        
        $client['invoices'] = Capsule::table('tblinvoices')
            ->where('userid', $id)
            ->orderBy('date', 'DESC')
            ->limit(10)
            ->get()
            ->toArray();
        
        return $this->success($client);
    }
    
    /**
     * Create client
     */
    public function store(array $params, array $auth): array {
        $this->validateRequired($params, ['firstname', 'lastname', 'email']);
        
        // Check for duplicate email
        $exists = Capsule::table('tblclients')
            ->where('email', $params['email'])
            ->first();
        
        if ($exists) {
            return $this->error('Email already exists', 409);
        }
        
        $insertData = [
            'firstname' => $this->sanitize($params['firstname']),
            'lastname' => $this->sanitize($params['lastname']),
            'email' => $this->sanitize($params['email']),
            'companyname' => $this->sanitize($params['companyname'] ?? ''),
            'address1' => $this->sanitize($params['address1'] ?? ''),
            'city' => $this->sanitize($params['city'] ?? ''),
            'state' => $this->sanitize($params['state'] ?? ''),
            'country' => $params['country'] ?? 'US',
            'phonenumber' => $params['phonenumber'] ?? '',
            'password' => $params['password'] ?? '',
            'created_at' => date('Y-m-d H:i:s'),
        ];
        
        $id = Capsule::table('tblclients')->insertGetId($insertData);
        
        return $this->success([
            'id' => $id,
            'message' => 'Client created successfully',
        ], 201);
    }
    
    /**
     * Update client
     */
    public function update(array $params, array $auth): array {
        $id = (int) ($params['id'] ?? 0);
        
        if (!$id) {
            return $this->error('Client ID required', 400);
        }
        
        $client = Capsule::table('tblclients')->where('id', $id)->first();
        
        if (!$client) {
            return $this->error('Client not found', 404);
        }
        
        // Check email uniqueness
        if (!empty($params['email']) && $params['email'] !== $client->email) {
            $exists = Capsule::table('tblclients')
                ->where('email', $params['email'])
                ->where('id', '!=', $id)
                ->first();
            
            if ($exists) {
                return $this->error('Email already in use', 409);
            }
        }
        
        $updateData = [];
        $allowedFields = ['firstname', 'lastname', 'email', 'companyname', 'address1', 
                         'city', 'state', 'country', 'phonenumber'];
        
        foreach ($allowedFields as $field) {
            if (isset($params[$field])) {
                $updateData[$field] = $this->sanitize($params[$field]);
            }
        }
        
        if (!empty($updateData)) {
            $updateData['updated_at'] = date('Y-m-d H:i:s');
            Capsule::table('tblclients')->where('id', $id)->update($updateData);
        }
        
        return $this->success([
            'id' => $id,
            'message' => 'Client updated successfully',
        ]);
    }
    
    /**
     * Delete client
     */
    public function destroy(array $params, array $auth): array {
        $id = (int) ($params['id'] ?? 0);
        
        if (!$id) {
            return $this->error('Client ID required', 400);
        }
        
        $client = Capsule::table('tblclients')->where('id', $id)->first();
        
        if (!$client) {
            return $this->error('Client not found', 404);
        }
        
        // Soft delete or check dependencies
        Capsule::table('tblclients')->where('id', $id)->update(['status' => 'Inactive']);
        
        return $this->success([
            'id' => $id,
            'message' => 'Client deactivated successfully',
        ]);
    }
}
```

## Services Controller

```php
<?php
/**
 * Services API Controller
 */

namespace CustomApi\Controllers;

class ServicesController extends BaseController {
    
    public function index(array $params, array $auth): array {
        $query = Capsule::table('tblhosting')
            ->join('tblclients', 'tblhosting.userid', '=', 'tblclients.id')
            ->join('tblproducts', 'tblhosting.packageid', '=', 'tblproducts.id')
            ->select([
                'tblhosting.*',
                'tblclients.firstname',
                'tblclients.lastname',
                'tblclients.email',
                'tblproducts.name as product_name',
            ]);
        
        if (!empty($params['client_id'])) {
            $query->where('tblhosting.userid', (int) $params['client_id']);
        }
        
        if (!empty($params['status'])) {
            $query->where('tblhosting.domainstatus', $params['status']);
        }
        
        $page = (int) ($params['page'] ?? 1);
        $perPage = (int) ($params['per_page'] ?? 20);
        
        return $this->paginate($query, $page, $perPage);
    }
    
    public function show(array $params, array $auth): array {
        $id = (int) ($params['id'] ?? 0);
        
        $service = Capsule::table('tblhosting')
            ->where('id', $id)
            ->first();
        
        if (!$service) {
            return $this->error('Service not found', 404);
        }
        
        return $this->success((array) $service);
    }
    
    public function store(array $params, array $auth): array {
        $this->validateRequired($params, ['client_id', 'product_id']);
        
        $id = Capsule::table('tblhosting')->insertGetId([
            'userid' => (int) $params['client_id'],
            'packageid' => (int) $params['product_id'],
            'domain' => $params['domain'] ?? '',
            'regdate' => date('Y-m-d'),
            'domainstatus' => 'Pending',
        ]);
        
        return $this->success(['id' => $id], 201);
    }
    
    public function update(array $params, array $auth): array {
        $id = (int) ($params['id'] ?? 0);
        
        $updateData = [];
        $allowedFields = ['domain', 'domainstatus', 'subscription_id', 'notes'];
        
        foreach ($allowedFields as $field) {
            if (isset($params[$field])) {
                $updateData[$field] = $params[$field];
            }
        }
        
        if (!empty($updateData)) {
            Capsule::table('tblhosting')->where('id', $id)->update($updateData);
        }
        
        return $this->success(['id' => $id]);
    }
    
    public function destroy(array $params, array $auth): array {
        $id = (int) ($params['id'] ?? 0);
        Capsule::table('tblhosting')->where('id', $id)->update(['domainstatus' => 'Terminated']);
        return $this->success(['id' => $id]);
    }
}
```

## API Documentation Template

```smarty
<div class="api-documentation">
    <h2>Custom API Documentation</h2>
    
    <div class="alert alert-info">
        <strong>API Base URL:</strong>
        <code>{$base_url}modules/addons/{module}/api.php</code>
    </div>

    <h3>Authentication</h3>
    <p>Include your API key in the request header:</p>
    <pre>X-API-Key: your-api-key-here</pre>

    <h3>Rate Limits</h3>
    <p>Default rate limit: 100 requests per minute per API key.</p>

    <h3>Response Format</h3>
    <pre>{
    "success": true,
    "status_code": 200,
    "data": { ... }
}</pre>

    <h3>Error Response</h3>
    <pre>{
    "error": true,
    "message": "Error description",
    "code": 400
}</pre>

    <h3>Endpoints</h3>
    
    <h4>Clients</h4>
    <table class="table table-bordered">
        <tr><th>Method</th><th>Endpoint</th><th>Description</th></tr>
        <tr><td>GET</td><td>/clients</td><td>List all clients</td></tr>
        <tr><td>GET</td><td>/clients/{id}</td><td>Get client details</td></tr>
        <tr><td>POST</td><td>/clients</td><td>Create client</td></tr>
        <tr><td>PUT</td><td>/clients/{id}</td><td>Update client</td></tr>
        <tr><td>DELETE</td><td>/clients/{id}</td><td>Delete client</td></tr>
    </table>

    <h4>Services</h4>
    <table class="table table-bordered">
        <tr><th>Method</th><th>Endpoint</th><th>Description</th></tr>
        <tr><td>GET</td><td>/services</td><td>List all services</td></tr>
        <tr><td>GET</td><td>/services/{id}</td><td>Get service details</td></tr>
        <tr><td>POST</td><td>/services</td><td>Create service</td></tr>
        <tr><td>PUT</td><td>/services/{id}</td><td>Update service</td></tr>
        <tr><td>DELETE</td><td>/services/{id}</td><td>Delete service</td></tr>
    </table>

    <h4>Invoices</h4>
    <table class="table table-bordered">
        <tr><th>Method</th><th>Endpoint</th><th>Description</th></tr>
        <tr><td>GET</td><td>/invoices</td><td>List all invoices</td></tr>
        <tr><td>GET</td><td>/invoices/{id}</td><td>Get invoice details</td></tr>
        <tr><td>POST</td><td>/invoices</td><td>Create invoice</td></tr>
    </table>
</div>
```

## Checklist

```
Pre-Dev:
□ Define API resources (clients, services, etc.)
□ Plan authentication method
□ Design URL structure
□ Define request/response formats
□ Plan versioning strategy

Development:
□ Create API server entry point
□ Implement Router class
□ Implement Middleware class
□ Create BaseController
□ Create resource controllers
□ Build endpoint registration system
□ Add request logging
□ Implement rate limiting
□ Build admin UI
□ Create API documentation

Testing:
□ Test authentication
□ Test all endpoints
□ Verify rate limiting
□ Test error responses
□ Test with invalid keys
□ Verify logging
□ Test pagination
```