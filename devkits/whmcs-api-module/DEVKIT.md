# WHMCS API Module DevKit
# Version: 1.0 | Updated: 2026-05-28

## DevKit Structure

```
devkits/whmcs-api-module/
├── api.php              # API controller
├── lib/
│   ├── ApiServer.php    # API server/handler
│   ├── Auth.php         # Authentication
│   ├── RateLimiter.php  # Rate limiting
│   └── Validators.php   # Input validation
├── endpoints/
│   ├── Clients.php      # Client endpoints
│   ├── Services.php     # Service endpoints
│   └── Invoices.php     # Invoice endpoints
└── templates/
    └── docs.tpl         # API documentation
```

## API Controller Template

```php
<?php
/**
 * WHMCS API Module: {Module}
 * DevKit Template
 * 
 * Installation: Upload to modules/addons/{module}/
 * Access via: /modules/addons/{module}/api.php
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
        'name' => '{API Module}',
        'description' => 'External API integration module',
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
        $t->string('key_name');
        $t->string('api_key', 64)->unique();
        $t->string('api_secret', 64);
        $t->text('permissions');
        $t->boolean('is_active');
        $t->string('ip_whitelist')->nullable();
        $t->integer('rate_limit')->default(100);
        $t->integer('requests_today')->default(0);
        $t->timestamp('last_request')->nullable();
        $t->timestamp('created_at');
    });
    
    Capsule::schema()->create('mod_{module}_api_logs', function($t) {
        $t->increments('id');
        $t->integer('api_key_id');
        $t->string('endpoint');
        $t->string('method');
        $t->text('request_data');
        $t->text('response_data');
        $t->integer('status_code');
        $t->string('ip_address', 45);
        $t->float('execution_time');
        $t->timestamp('created_at');
    });
    
    Capsule::schema()->create('mod_{module}_api_webhooks', function($t) {
        $t->increments('id');
        $t->string('event_type');
        $t->string('webhook_url');
        $t->text('headers');
        $t->boolean('is_active');
        $t->integer('retry_count');
        $t->timestamp('last_triggered')->nullable();
    });
    
    return ['status' => 'success', 'description' => 'Module activated'];
}

/**
 * Deactivate
 */
function {module}_deactivate(): array {
    Capsule::schema()->dropIfExists('mod_{module}_api_keys');
    Capsule::schema()->dropIfExists('mod_{module}_api_logs');
    Capsule::schema()->dropIfExists('mod_{module}_api_webhooks');
    
    return ['status' => 'success'];
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
        case 'logs':
            {module}_viewLogs();
            break;
        case 'webhooks':
            {module}_manageWebhooks();
            break;
        case 'settings':
            {module}_showSettings();
            break;
        case 'docs':
            {module}_showDocs();
            break;
        default:
            {module}_showDashboard();
    }
}

/**
 * Show Dashboard
 */
function {module}_showDashboard(): void {
    $stats = [
        'total_keys' => Capsule::table('mod_{module}_api_keys')->count(),
        'active_keys' => Capsule::table('mod_{module}_api_keys')
            ->where('is_active', 1)->count(),
        'requests_today' => Capsule::table('mod_{module}_api_logs')
            ->whereDate('created_at', date('Y-m-d'))->count(),
        'errors_today' => Capsule::table('mod_{module}_api_logs')
            ->whereDate('created_at', date('Y-m-d'))
            ->where('status_code', '>=', 400)->count(),
    ];
    
    echo <<<HTML
<div class="api-module">
    <div class="row">
        <div class="col-md-12">
            <div class="panel panel-default">
                <div class="panel-heading">
                    <h3 class="panel-title">API Dashboard</h3>
                </div>
                <div class="panel-body">
                    <div class="row">
                        <div class="col-md-3">
                            <div class="stat-box">
                                <div class="stat-value">{$stats['total_keys']}</div>
                                <div class="stat-label">Total API Keys</div>
                            </div>
                        </div>
                        <div class="col-md-3">
                            <div class="stat-box">
                                <div class="stat-value">{$stats['active_keys']}</div>
                                <div class="stat-label">Active Keys</div>
                            </div>
                        </div>
                        <div class="col-md-3">
                            <div class="stat-box">
                                <div class="stat-value">{$stats['requests_today']}</div>
                                <div class="stat-label">Requests Today</div>
                            </div>
                        </div>
                        <div class="col-md-3">
                            <div class="stat-box">
                                <div class="stat-value">{$stats['errors_today']}</div>
                                <div class="stat-label">Errors Today</div>
                            </div>
                        </div>
                    </div>
                </div>
            </div>
        </div>
    </div>
    
    <div class="row">
        <div class="col-md-12">
            <div class="btn-group">
                <a href="?module={module}&action=keys" class="btn btn-primary">
                    <i class="fa fa-key"></i> API Keys
                </a>
                <a href="?module={module}&action=webhooks" class="btn btn-primary">
                    <i class="fa fa-bell"></i> Webhooks
                </a>
                <a href="?module={module}&action=logs" class="btn btn-default">
                    <i class="fa fa-file-alt"></i> Logs
                </a>
                <a href="?module={module}&action=settings" class="btn btn-default">
                    <i class="fa fa-cog"></i> Settings
                </a>
                <a href="?module={module}&action=docs" class="btn btn-info">
                    <i class="fa fa-book"></i> API Docs
                </a>
            </div>
        </div>
    </div>
</div>
HTML;
}
```

## API Server Class

```php
<?php
namespace {Module};

use WHMCS\Database\Capsule;

class ApiServer {
    
    private array $headers = [
        'Content-Type' => 'application/json',
        'X-API-Version' => '1.0',
    ];
    
    private array $routes = [];
    private array $authData = [];
    
    public function __construct() {
        $this->registerRoutes();
    }
    
    private function registerRoutes(): void {
        // Client routes
        $this->routes['GET']['/api/clients'] = ['{Module}\Clients', 'index'];
        $this->routes['GET']['/api/clients/{id}'] = ['{Module}\Clients', 'show'];
        $this->routes['POST']['/api/clients'] = ['{Module}\Clients', 'store'];
        $this->routes['PUT']['/api/clients/{id}'] = ['{Module}\Clients', 'update'];
        $this->routes['DELETE']['/api/clients/{id}'] = ['{Module}\Clients', 'destroy'];
        
        // Services routes
        $this->routes['GET']['/api/services'] = ['{Module}\Services', 'index'];
        $this->routes['GET']['/api/services/{id}'] = ['{Module}\Services', 'show'];
        $this->routes['POST']['/api/services'] = ['{Module}\Services', 'store'];
        $this->routes['PUT']['/api/services/{id}'] = ['{Module}\Services', 'update'];
        $this->routes['DELETE']['/api/services/{id}'] = ['{Module}\Services', 'destroy'];
        
        // Invoices routes
        $this->routes['GET']['/api/invoices'] = ['{Module}\Invoices', 'index'];
        $this->routes['GET']['/api/invoices/{id}'] = ['{Module}\Invoices', 'show'];
        $this->routes['POST']['/api/invoices'] = ['{Module}\Invoices', 'store'];
        
        // Domain routes
        $this->routes['GET']['/api/domains'] = ['{Module}\Domains', 'index'];
        $this->routes['GET']['/api/domains/{id}'] = ['{Module}\Domains', 'show'];
        
        // Ticket routes
        $this->routes['GET']['/api/tickets'] = ['{Module}\Tickets', 'index'];
        $this->routes['POST']['/api/tickets'] = ['{Module}\Tickets', 'store'];
        $this->routes['POST']['/api/tickets/{id}/reply'] = ['{Module}\Tickets', 'reply'];
    }
    
    public function handle(): void {
        $startTime = microtime(true);
        
        try {
            // Get request details
            $method = $_SERVER['REQUEST_METHOD'];
            $path = parse_url($_SERVER['REQUEST_URI'], PHP_URL_PATH);
            $path = str_replace('/modules/addons/{module}/', '', $path);
            
            // Authenticate
            $auth = new Auth();
            $this->authData = $auth->authenticate();
            
            // Rate limiting
            $rateLimiter = new RateLimiter($this->authData['key_id']);
            if (!$rateLimiter->check()) {
                throw new ApiException('Rate limit exceeded', 429);
            }
            
            // Find and execute route
            $route = $this->matchRoute($method, $path);
            
            if (!$route) {
                throw new ApiException('Endpoint not found', 404);
            }
            
            // Execute controller
            [$controller, $action] = $route['handler'];
            $params = $route['params'];
            
            $data = file_get_contents('php://input');
            $requestData = json_decode($data, true) ?? [];
            $requestData = array_merge($requestData, $_GET, $_POST);
            
            $controllerInstance = new $controller();
            $result = $controllerInstance->$action($requestData, $params);
            
            // Log request
            $this->logRequest($route, $requestData, $result, 200, $startTime);
            
            $this->respond($result, 200);
            
        } catch (ApiException $e) {
            $this->logRequest(null, [], ['error' => $e->getMessage()], $e->getCode(), $startTime);
            $this->respond(['error' => $e->getMessage()], $e->getCode());
        } catch (\Exception $e) {
            $this->logRequest(null, [], ['error' => 'Internal server error'], 500, $startTime);
            $this->respond(['error' => 'Internal server error'], 500);
        }
    }
    
    private function matchRoute(string $method, string $path): ?array {
        foreach ($this->routes[$method] ?? [] as $pattern => $handler) {
            $regex = preg_replace('/\{[^}]+\}/', '([^/]+)', $pattern);
            $regex = '#^' . $regex . '$#';
            
            if (preg_match($regex, $path, $matches)) {
                array_shift($matches);
                return [
                    'handler' => $handler,
                    'params' => $matches,
                    'pattern' => $pattern,
                ];
            }
        }
        
        return null;
    }
    
    private function respond(array $data, int $statusCode): void {
        http_response_code($statusCode);
        
        foreach ($this->headers as $name => $value) {
            header("$name: $value");
        }
        
        echo json_encode($data, JSON_PRETTY_PRINT);
        exit;
    }
    
    private function logRequest(array $route, array $request, array $response, int $statusCode, float $startTime): void {
        $executionTime = microtime(true) - $startTime;
        
        Capsule::table('mod_{module}_api_logs')->insert([
            'api_key_id' => $this->authData['key_id'] ?? 0,
            'endpoint' => $route['pattern'] ?? 'unknown',
            'method' => $_SERVER['REQUEST_METHOD'] ?? 'GET',
            'request_data' => json_encode($request),
            'response_data' => json_encode($response),
            'status_code' => $statusCode,
            'ip_address' => $_SERVER['REMOTE_ADDR'] ?? '',
            'execution_time' => round($executionTime * 1000, 2),
            'created_at' => date('Y-m-d H:i:s'),
        ]);
        
        // Update request count
        if (isset($this->authData['key_id'])) {
            Capsule::table('mod_{module}_api_keys')
                ->where('id', $this->authData['key_id'])
                ->update([
                    'requests_today' => Capsule::raw('requests_today + 1'),
                    'last_request' => date('Y-m-d H:i:s'),
                ]);
        }
    }
}

class ApiException extends \Exception {
    public function __construct(string $message, int $code = 400) {
        parent::__construct($message, $code);
    }
}
```

## Authentication Class

```php
<?php
namespace {Module};

use WHMCS\Database\Capsule;

class Auth {
    
    public function authenticate(): array {
        // Try API Key header first
        $apiKey = $_SERVER['HTTP_X_API_KEY'] ?? '';
        
        // Try Bearer token
        if (empty($apiKey)) {
            $authHeader = $_SERVER['HTTP_AUTHORIZATION'] ?? '';
            if (preg_match('/Bearer\s+(.+)/i', $authHeader, $matches)) {
                $apiKey = $matches[1];
            }
        }
        
        // Try query parameter (not recommended for production)
        if (empty($apiKey)) {
            $apiKey = $_GET['api_key'] ?? '';
        }
        
        if (empty($apiKey)) {
            throw new ApiException('API key required', 401);
        }
        
        // Find API key in database
        $key = Capsule::table('mod_{module}_api_keys')
            ->where('api_key', $apiKey)
            ->where('is_active', 1)
            ->first();
        
        if (!$key) {
            throw new ApiException('Invalid API key', 401);
        }
        
        // Check IP whitelist
        if (!empty($key->ip_whitelist)) {
            $allowedIPs = explode(',', $key->ip_whitelist);
            $clientIP = $_SERVER['REMOTE_ADDR'] ?? '';
            
            if (!in_array($clientIP, array_map('trim', $allowedIPs))) {
                throw new ApiException('IP address not allowed', 403);
            }
        }
        
        return [
            'key_id' => $key->id,
            'key_name' => $key->key_name,
            'permissions' => json_decode($key->permissions, true) ?? [],
        ];
    }
    
    public function hasPermission(string $permission): bool {
        $permissions = $this->authenticate()['permissions'] ?? [];
        
        // '*' means all permissions
        if (in_array('*', $permissions)) {
            return true;
        }
        
        return in_array($permission, $permissions);
    }
}
```

## Rate Limiter Class

```php
<?php
namespace {Module};

use WHMCS\Database\Capsule;

class RateLimiter {
    
    private int $keyId;
    private int $limit;
    private string $window = 'minute';
    
    public function __construct(int $keyId) {
        $this->keyId = $keyId;
        
        $key = Capsule::table('mod_{module}_api_keys')
            ->where('id', $keyId)
            ->first();
        
        $this->limit = $key ? $key->rate_limit : 100;
    }
    
    public function check(): bool {
        $windowStart = date('Y-m-d H:i:00');
        $windowEnd = date('Y-m-d H:i:59');
        
        $requests = Capsule::table('mod_{module}_api_logs')
            ->where('api_key_id', $this->keyId)
            ->whereBetween('created_at', [$windowStart, $windowEnd])
            ->count();
        
        return $requests < $this->limit;
    }
    
    public function getRemaining(): int {
        $windowStart = date('Y-m-d H:i:00');
        $windowEnd = date('Y-m-d H:i:59');
        
        $requests = Capsule::table('mod_{module}_api_logs')
            ->where('api_key_id', $this->keyId)
            ->whereBetween('created_at', [$windowStart, $windowEnd])
            ->count();
        
        return max(0, $this->limit - $requests);
    }
    
    public function getResetTime(): int {
        return strtotime(date('Y-m-d H:i:59')) - time();
    }
}
```

## Endpoint Examples

```php
<?php
namespace {Module};

use WHMCS\Database\Capsule;

class Clients {
    
    public function index(array $request, array $params): array {
        $query = Capsule::table('tblclients');
        
        // Filters
        if (!empty($request['search'])) {
            $search = $request['search'];
            $query->where(function($q) use ($search) {
                $q->where('firstname', 'LIKE', "%{$search}%")
                  ->orWhere('lastname', 'LIKE', "%{$search}%")
                  ->orWhere('email', 'LIKE', "%{$search}%");
            });
        }
        
        if (!empty($request['status'])) {
            $query->where('status', $request['status']);
        }
        
        // Pagination
        $page = (int) ($request['page'] ?? 1);
        $perPage = (int) ($request['per_page'] ?? 20);
        $offset = ($page - 1) * $perPage;
        
        $total = $query->count();
        $clients = $query->limit($perPage)->offset($offset)->get();
        
        return [
            'data' => $clients->toArray(),
            'meta' => [
                'current_page' => $page,
                'per_page' => $perPage,
                'total' => $total,
                'total_pages' => ceil($total / $perPage),
            ],
        ];
    }
    
    public function show(array $request, array $params): array {
        $id = (int) $params[0];
        
        $client = Capsule::table('tblclients')->where('id', $id)->first();
        
        if (!$client) {
            throw new ApiException('Client not found', 404);
        }
        
        // Include related data
        $client->services = Capsule::table('tblhosting')
            ->where('userid', $id)
            ->get();
        
        $client->invoices = Capsule::table('tblinvoices')
            ->where('userid', $id)
            ->orderBy('date', 'desc')
            ->limit(10)
            ->get();
        
        return ['data' => $client];
    }
    
    public function store(array $request, array $params): array {
        // Validate required fields
        $required = ['firstname', 'lastname', 'email'];
        foreach ($required as $field) {
            if (empty($request[$field])) {
                throw new ApiException("Field '{$field}' is required", 400);
            }
        }
        
        // Check for duplicate email
        $exists = Capsule::table('tblclients')
            ->where('email', $request['email'])
            ->first();
        
        if ($exists) {
            throw new ApiException('Email already exists', 409);
        }
        
        $id = Capsule::table('tblclients')->insertGetId([
            'firstname' => $request['firstname'],
            'lastname' => $request['lastname'],
            'email' => $request['email'],
            'companyname' => $request['companyname'] ?? '',
            'country' => $request['country'] ?? 'US',
            'created_at' => date('Y-m-d H:i:s'),
        ]);
        
        return [
            'message' => 'Client created successfully',
            'data' => ['id' => $id],
        ];
    }
}
```

## Checklist

```
Pre-Dev:
□ Define API design (REST, GraphQL)
□ Plan authentication method
□ Design rate limiting strategy
□ Identify required endpoints
□ Plan versioning strategy
□ Design error responses

Development:
□ Create API tables
□ Implement ApiServer class
□ Implement routing system
□ Create Auth class
□ Implement API key authentication
□ Add IP whitelist support
□ Create RateLimiter class
□ Implement request logging
□ Build endpoint controllers
□ Create input validation
□ Add error handling
□ Build admin management UI
□ Create API documentation

Testing:
□ Test API authentication
□ Test rate limiting
□ Test all endpoints
□ Verify error responses
□ Test input validation
□ Test with invalid API keys
□ Test IP whitelist
□ Test request logging
□ Test concurrent requests
□ Verify response format
```