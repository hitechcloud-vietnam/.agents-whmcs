# WHMCS API Development Workflow

## Purpose

Systematic approach to building custom API endpoints for WHMCS integration. Covers design, implementation, security, testing, and documentation of custom REST/Soap APIs.

## Prerequisites

- WHMCS installation (v7.0+)
- PHP 7.4+ development environment
- API authentication credentials
- OpenAPI/Swagger knowledge
- Understanding of WHMCS database schema

## Workflow Steps

### Step 1: Define API Requirements

Document the API specifications:

```markdown
## Custom WHMCS API Specification

### Endpoint: /api/v1/customers/{id}/services

**Method:** GET
**Description:** Retrieve all services for a customer
**Authentication:** API Key or Bearer Token

**Response Schema:**
```json
{
  "success": true,
  "data": {
    "customer_id": 123,
    "services": [
      {
        "id": 456,
        "product_name": "Business Hosting",
        "status": "Active",
        "next_due_date": "2024-02-15",
        "billing_cycle": "Monthly",
        "price": 9.99
      }
    ],
    "total_monthly": 29.97
  }
}
```

### Security Requirements
- Rate limiting: 100 requests per minute
- Authentication required
- HTTPS only
- Input validation mandatory
```

### Step 2: Create API Directory Structure

Organize custom API code:

```bash
mkdir -p /var/www/whmcs/modules/custom/api/
mkdir -p /var/www/whmcs/modules/custom/api/Controllers
mkdir -p /var/www/whmcs/modules/custom/api/Middleware
mkdir -p /var/www/whmcs/modules/custom/api/Validators
mkdir -p /var/www/whmcs/modules/custom/api/Routes
```

### Step 3: Implement API Handler

Create the main API class:

```php
<?php
// modules/custom/api/ApiHandler.php

namespace WHMCS\Custom\Api;

class ApiHandler
{
    protected $response;
    
    public function __construct()
    {
        $this->response = [
            'success' => false,
            'message' => '',
            'data' => null,
        ];
    }
    
    protected function success($data, $message = 'Success')
    {
        $this->response = [
            'success' => true,
            'message' => $message,
            'data' => $data,
            'timestamp' => date('c'),
        ];
        return $this;
    }
    
    protected function error($message, $code = 400)
    {
        $this->response = [
            'success' => false,
            'message' => $message,
            'data' => null,
            'error_code' => $code,
            'timestamp' => date('c'),
        ];
        http_response_code($code);
        return $this;
    }
    
    public function send()
    {
        header('Content-Type: application/json');
        header('Access-Control-Allow-Origin: *');
        header('Access-Control-Allow-Methods: GET, POST, PUT, DELETE, OPTIONS');
        header('Access-Control-Allow-Headers: Content-Type, Authorization');
        
        if ($_SERVER['REQUEST_METHOD'] === 'OPTIONS') {
            http_response_code(200);
            exit;
        }
        
        echo json_encode($this->response, JSON_PRETTY_PRINT);
    }
}
```

### Step 4: Create Authentication Middleware

Secure API endpoints:

```php
<?php
// modules/custom/api/Middleware/ApiAuth.php

namespace WHMCS\Custom\Api\Middleware;

class ApiAuth
{
    private $apiKeys = [];
    
    public function __construct()
    {
        // Load API keys from database or config
        $this->apiKeys = [
            'client_key_123' => [
                'client_id' => 1,
                'permissions' => ['read_customers', 'read_services'],
                'rate_limit' => 100,
            ],
        ];
    }
    
    public function authenticate()
    {
        $authHeader = $_SERVER['HTTP_AUTHORIZATION'] ?? '';
        
        if (empty($authHeader)) {
            throw new \Exception('Authentication required', 401);
        }
        
        // Bearer token authentication
        if (preg_match('/Bearer\s+(.+)/i', $authHeader, $matches)) {
            $token = $matches[1];
            return $this->validateToken($token);
        }
        
        // API key authentication
        if (preg_match('/ApiKey\s+(.+)/i', $authHeader, $matches)) {
            $apiKey = $matches[1];
            return $this->validateApiKey($apiKey);
        }
        
        throw new \Exception('Invalid authentication method', 401);
    }
    
    private function validateToken($token)
    {
        // Verify JWT or session token
        // Implement your token validation logic
        $payload = $this->decodeToken($token);
        
        if (!$payload) {
            throw new \Exception('Invalid token', 401);
        }
        
        return $payload;
    }
    
    private function validateApiKey($apiKey)
    {
        if (!isset($this->apiKeys[$apiKey])) {
            throw new \Exception('Invalid API key', 401);
        }
        
        return (object) $this->apiKeys[$apiKey];
    }
    
    public function checkPermission($requiredPermission)
    {
        $auth = $this->authenticate();
        
        if (!in_array($requiredPermission, $auth->permissions ?? [])) {
            throw new \Exception('Insufficient permissions', 403);
        }
        
        return $auth;
    }
}
```

### Step 5: Create Rate Limiting Middleware

Prevent API abuse:

```php
<?php
// modules/custom/api/Middleware/RateLimiter.php

namespace WHMCS\Custom\Api\Middleware;

use WHMCS\Carbon;
use WHMCS\Custom\Api\Middleware\ApiAuth;

class RateLimiter
{
    private $limit = 100; // requests per window
    private $window = 60; // seconds
    
    public function limit()
    {
        $identifier = $this->getIdentifier();
        $cacheKey = "rate_limit_{$identifier}";
        
        $data = \WHMCS\Session::get($cacheKey);
        
        if (!$data) {
            $data = [
                'count' => 0,
                'reset' => time() + $this->window,
            ];
        }
        
        // Check if window expired
        if (time() > $data['reset']) {
            $data = [
                'count' => 0,
                'reset' => time() + $this->window,
            ];
        }
        
        $data['count']++;
        
        // Check limit
        if ($data['count'] > $this->limit) {
            $retryAfter = $data['reset'] - time();
            header("Retry-After: {$retryAfter}");
            header("X-RateLimit-Limit: {$this->limit}");
            header("X-RateLimit-Remaining: 0");
            header("X-RateLimit-Reset: {$data['reset']}");
            
            throw new \Exception(
                "Rate limit exceeded. Retry after {$retryAfter} seconds.",
                429
            );
        }
        
        \WHMCS\Session::set($cacheKey, $data);
        
        header("X-RateLimit-Limit: {$this->limit}");
        header("X-RateLimit-Remaining: " . ($this->limit - $data['count']));
        header("X-RateLimit-Reset: {$data['reset']}");
    }
    
    private function getIdentifier()
    {
        // Use API key or IP as identifier
        $authHeader = $_SERVER['HTTP_AUTHORIZATION'] ?? '';
        if (preg_match('/ApiKey\s+(.+)/i', $authHeader, $matches)) {
            return hash('sha256', $matches[1]);
        }
        return hash('sha256', $_SERVER['REMOTE_ADDR'] ?? 'unknown');
    }
}
```

### Step 6: Implement API Controllers

Create endpoint handlers:

```php
<?php
// modules/custom/api/Controllers/CustomerController.php

namespace WHMCS\Custom\Api\Controllers;

use WHMCS\Custom\Api\ApiHandler;
use WHMCS\Custom\Api\Middleware\ApiAuth;
use WHMCS\Custom\Api\Middleware\RateLimiter;
use WHMCS\Custom\Api\Validators\CustomerValidator;

class CustomerController extends ApiHandler
{
    private $auth;
    private $rateLimiter;
    
    public function __construct()
    {
        parent::__construct();
        $this->auth = new ApiAuth();
        $this->rateLimiter = new RateLimiter();
    }
    
    public function getServices($customerId)
    {
        try {
            // Authenticate and rate limit
            $this->auth->checkPermission('read_services');
            $this->rateLimiter->limit();
            
            // Validate input
            $validator = new CustomerValidator();
            if (!$validator->validateCustomerId($customerId)) {
                return $this->error('Invalid customer ID', 400)->send();
            }
            
            // Get customer data
            $customer = \WHMCS\Database\Capsule::table('tblclients')
                ->where('id', $customerId)
                ->first();
            
            if (!$customer) {
                return $this->error('Customer not found', 404)->send();
            }
            
            // Get services
            $services = \WHMCS\Database\Capsule::table('tblhosting')
                ->join('tblproducts', 'tblhosting.packageid', '=', 'tblproducts.id')
                ->where('tblhosting.userid', $customerId)
                ->where('tblhosting.domainstatus', '!=', 'Terminated')
                ->select([
                    'tblhosting.id',
                    'tblproducts.name as product_name',
                    'tblhosting.domainstatus as status',
                    'tblhosting.nextduedate as next_due_date',
                    'tblhosting.billingcycle as billing_cycle',
                    'tblhosting.amount as price',
                ])
                ->get();
            
            // Calculate totals
            $totalMonthly = 0;
            foreach ($services as $service) {
                if (in_array($service->billing_cycle, ['Monthly', 'Quarterly', 'Annually'])) {
                    $multiplier = match($service->billing_cycle) {
                        'Monthly' => 1,
                        'Quarterly' => 1/3,
                        'Annually' => 1/12,
                        default => 1,
                    };
                    $totalMonthly += $service->price * $multiplier;
                }
            }
            
            return $this->success([
                'customer_id' => (int) $customerId,
                'services' => $services,
                'total_monthly' => round($totalMonthly, 2),
            ])->send();
            
        } catch (\Exception $e) {
            return $this->error($e->getMessage(), $e->getCode() ?: 500)->send();
        }
    }
    
    public function createService($customerId)
    {
        try {
            $this->auth->checkPermission('create_services');
            $this->rateLimiter->limit();
            
            // Get request body
            $input = json_decode(file_get_contents('php://input'), true);
            
            // Validate
            $validator = new CustomerValidator();
            $errors = $validator->validateServiceCreation($input);
            
            if (!empty($errors)) {
                return $this->error('Validation failed: ' . implode(', ', $errors), 400)->send();
            }
            
            // Create service (simplified example)
            $serviceId = \WHMCS\Database\Capsule::table('tblhosting')->insertGetId([
                'userid' => $customerId,
                'packageid' => $input['product_id'],
                'regdate' => date('Y-m-d'),
                'domainstatus' => 'Pending',
                'nextduedate' => date('Y-m-d', strtotime('+30 days')),
                'billingcycle' => 'Monthly',
                'amount' => $input['price'],
            ]);
            
            return $this->success([
                'service_id' => $serviceId,
                'message' => 'Service created successfully',
            ], 'Service created', 201)->send();
            
        } catch (\Exception $e) {
            return $this->error($e->getMessage(), $e->getCode() ?: 500)->send();
        }
    }
}
```

### Step 7: Create Input Validators

Validate API inputs:

```php
<?php
// modules/custom/api/Validators/CustomerValidator.php

namespace WHMCS\Custom\Api\Validators;

class CustomerValidator
{
    public function validateCustomerId($id)
    {
        return is_numeric($id) && $id > 0;
    }
    
    public function validateServiceCreation(array $data)
    {
        $errors = [];
        
        if (empty($data['product_id'])) {
            $errors[] = 'product_id is required';
        } elseif (!$this->productExists($data['product_id'])) {
            $errors[] = 'product_id does not exist';
        }
        
        if (!isset($data['price']) || !is_numeric($data['price']) || $data['price'] < 0) {
            $errors[] = 'price must be a positive number';
        }
        
        if (!empty($data['billing_cycle'])) {
            $validCycles = ['Monthly', 'Quarterly', 'Semi-Annually', 'Annually', 'Biennially', 'Triennially'];
            if (!in_array($data['billing_cycle'], $validCycles)) {
                $errors[] = 'billing_cycle must be one of: ' . implode(', ', $validCycles);
            }
        }
        
        return $errors;
    }
    
    private function productExists($productId)
    {
        return \WHMCS\Database\Capsule::table('tblproducts')
            ->where('id', $productId)
            ->exists();
    }
}
```

### Step 8: Set Up Routing

Configure API routes:

```php
<?php
// modules/custom/api/Routes/router.php

namespace WHMCS\Custom\Api;

class Router
{
    private $routes = [];
    
    public function register($method, $path, $handler, $middleware = [])
    {
        $this->routes[] = [
            'method' => strtoupper($method),
            'path' => $path,
            'handler' => $handler,
            'middleware' => $middleware,
        ];
    }
    
    public function dispatch()
    {
        $method = $_SERVER['REQUEST_METHOD'];
        $uri = parse_url($_SERVER['REQUEST_URI'], PHP_URL_PATH);
        
        foreach ($this->routes as $route) {
            if ($route['method'] !== $method) {
                continue;
            }
            
            $params = $this->matchRoute($route['path'], $uri);
            if ($params !== false) {
                // Run middleware
                foreach ($route['middleware'] as $middleware) {
                    $middleware->handle();
                }
                
                // Call handler
                $handler = $route['handler'];
                $handler(...$params);
                return;
            }
        }
        
        http_response_code(404);
        echo json_encode([
            'success' => false,
            'message' => 'Endpoint not found',
        ]);
    }
    
    private function matchRoute($pattern, $uri)
    {
        // Convert route pattern to regex
        $regex = preg_replace('/\{([a-zA-Z_]+)\}/', '(?P<$1>[^/]+)', $pattern);
        $regex = '#^' . $regex . '$#';
        
        if (preg_match($regex, $uri, $matches)) {
            $params = array_filter($matches, 'is_string', ARRAY_FILTER_USE_KEY);
            return array_values($params);
        }
        
        return false;
    }
}
```

### Step 9: Register Hook

Connect API to WHMCS:

```php
<?php
// modules/custom/api/hooks.php

use WHMCS\Custom\Api\Router;
use WHMCS\Custom\Api\Controllers\CustomerController;
use WHMCS\Custom\Api\Middleware\RateLimiter;

// Custom API endpoint handler
add_hook('CustomOutput', 1, function($vars) {
    $uri = $_SERVER['REQUEST_URI'];
    
    // Only handle /api/v1/* requests
    if (strpos($uri, '/api/v1/') !== 0) {
        return;
    }
    
    $router = new Router();
    $customerController = new CustomerController();
    $rateLimiter = new RateLimiter();
    
    // Register routes
    $router->register('GET', '/api/v1/customers/{customerId}/services', 
        [$customerController, 'getServices'],
        [$rateLimiter]
    );
    
    $router->register('POST', '/api/v1/customers/{customerId}/services',
        [$customerController, 'createService'],
        [$rateLimiter]
    );
    
    $router->dispatch();
    exit;
});
```

### Step 10: Create API Documentation

Generate OpenAPI specification:

```yaml
# openapi.yaml
openapi: 3.0.3
info:
  title: WHMCS Custom API
  version: 1.0.0
  description: Custom API endpoints for WHMCS integration

servers:
  - url: https://billing.example.com/api/v1
    description: Production
  - url: https://staging.example.com/api/v1
    description: Staging

components:
  securitySchemes:
    ApiKey:
      type: apiKey
      in: header
      name: X-API-Key
    BearerAuth:
      type: http
      scheme: bearer
      bearerFormat: JWT

paths:
  /customers/{customerId}/services:
    get:
      summary: Get customer services
      security:
        - ApiKey: []
      parameters:
        - name: customerId
          in: path
          required: true
          schema:
            type: integer
      responses:
        '200':
          description: Success
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/CustomerServices'
        '401':
          $ref: '#/components/responses/Unauthorized'
        '404':
          $ref: '#/components/responses/NotFound'

  /customers/{customerId}/services:
    post:
      summary: Create customer service
      security:
        - ApiKey: []
      parameters:
        - name: customerId
          in: path
          required: true
          schema:
            type: integer
      requestBody:
        required: true
        content:
          application/json:
            schema:
              $ref: '#/components/schemas/CreateServiceRequest'
      responses:
        '201':
          description: Created
        '400':
          $ref: '#/components/responses/BadRequest'

components:
  schemas:
    CustomerServices:
      type: object
      properties:
        success:
          type: boolean
        data:
          type: object
          properties:
            customer_id:
              type: integer
            services:
              type: array
              items:
                $ref: '#/components/schemas/Service'
            total_monthly:
              type: number

    Service:
      type: object
      properties:
        id:
          type: integer
        product_name:
          type: string
        status:
          type: string
        next_due_date:
          type: string
          format: date
        billing_cycle:
          type: string
        price:
          type: number

  responses:
    Unauthorized:
      description: Authentication required
      content:
        application/json:
          schema:
            type: object
            properties:
              success:
                type: boolean
                example: false
              message:
                type: string
                example: Authentication required
```

## Verification Checklist

- [ ] API requirements documented
- [ ] Authentication implemented
- [ ] Rate limiting configured
- [ ] Input validation complete
- [ ] Error handling implemented
- [ ] Unit tests written
- [ ] API documentation created
- [ ] OpenAPI spec generated
- [ ] Security review completed
- [ ] Load testing performed
- [ ] Monitoring configured

## Related Skills and Documentation

- [WHMCS Webhook Automation](whmcs-webhook-automation-workflow.md)
- [WHMCS Integration Testing](whmcs-integration-testing-workflow.md)
- [WHMCS Security Audit](whmcs-security-audit.md)
- WHMCS API Documentation: https://developers.whmcs.com/api-index/
- WHMCS API Authentication: https://developers.whmcs.com/authentication/

## Notes

- Always use HTTPS for all API endpoints
- Implement proper rate limiting to prevent abuse
- Validate and sanitize all input
- Use prepared statements for database queries
- Log all API requests for audit purposes
- Version your API for backward compatibility
