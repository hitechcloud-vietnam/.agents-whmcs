# WHMCS REST API Implementation Skill
# Version: 1.0 | Updated: 2026-05-29

## Purpose

Guide for implementing custom REST APIs in WHMCS for external integrations, mobile apps, and third-party service communication.

## When to Use

- Building APIs for mobile applications
- Creating integrations with external services
- Exposing WHMCS data to third-party systems
- Webhook endpoints for incoming data

## REST API Architecture

### 1. Basic REST API Structure

```php
<?php
// modules/addons/myapi/api/Router.php

namespace WHMCS\Module\Addon\MyApi\Api;

class Router {
    private array $routes = [];
    private string $basePath = '/api/v1';

    public function __construct() {
        $this->registerRoutes();
    }

    private function registerRoutes(): void {
        // GET routes
        $this->routes['GET'] = [
            '/clients' => [$this, 'getClients'],
            '/clients/{id}' => [$this, 'getClient'],
            '/invoices' => [$this, 'getInvoices'],
            '/invoices/{id}' => [$this, 'getInvoice'],
            '/services' => [$this, 'getServices'],
            '/services/{id}' => [$this, 'getService'],
            '/tickets' => [$this, 'getTickets'],
        ];

        // POST routes
        $this->routes['POST'] = [
            '/clients' => [$this, 'createClient'],
            '/invoices' => [$this, 'createInvoice'],
            '/tickets' => [$this, 'createTicket'],
        ];

        // PUT routes
        $this->routes['PUT'] = [
            '/clients/{id}' => [$this, 'updateClient'],
            '/services/{id}' => [$this, 'updateService'],
        ];

        // DELETE routes
        $this->routes['DELETE'] = [
            '/clients/{id}' => [$this, 'deleteClient'],
        ];
    }

    public function dispatch(string $method, string $uri): void {
        $method = strtoupper($method);
        $uri = str_replace($this->basePath, '', $uri);

        if (!isset($this->routes[$method])) {
            $this->sendResponse(405, ['error' => 'Method not allowed']);
            return;
        }

        foreach ($this->routes[$method] as $pattern => $handler) {
            $params = $this->matchRoute($pattern, $uri);
            if ($params !== false) {
                $this->processRequest($handler, $params);
                return;
            }
        }

        $this->sendResponse(404, ['error' => 'Endpoint not found']);
    }

    private function matchRoute(string $pattern, string $uri): array|false {
        $regex = preg_replace('/\{(\w+)\}/', '(?P<$1>[^/]+)', $pattern);
        $regex = '#^' . $regex . '$#';

        if (preg_match($regex, $uri, $matches)) {
            return array_filter($matches, 'is_string', ARRAY_FILTER_USE_KEY);
        }

        return false;
    }

    private function processRequest(callable $handler, array $params): void {
        try {
            $requestBody = json_decode(file_get_contents('php://input'), true) ?? [];
            $result = call_user_func_array($handler, [$params, $requestBody]);
            $this->sendResponse(200, $result);
        } catch (\Exception $e) {
            $this->sendResponse(400, ['error' => $e->getMessage()]);
        }
    }

    private function sendResponse(int $status, array $data): void {
        http_response_code($status);
        header('Content-Type: application/json');
        echo json_encode($data);
        exit;
    }
}
```

### 2. API Endpoint Handler

```php
<?php
// modules/addons/myapi/api/ClientController.php

namespace WHMCS\Module\Addon\MyApi\Api;

use WHMCS\Database\Capsule;

class ClientController {
    private AuthService $auth;

    public function __construct() {
        $this->auth = new AuthService();
    }

    // GET /api/v1/clients
    public function getClients(array $params, array $body): array {
        $this->auth->requirePermission('clients:read');

        $page = (int) ($params['page'] ?? 1);
        $limit = min((int) ($params['limit'] ?? 25), 100);
        $offset = ($page - 1) * $limit;

        $clients = Capsule::table('tblclients')
            ->select('id', 'firstname', 'lastname', 'email', 'status', 'created_at')
            ->orderBy('id', 'DESC')
            ->offset($offset)
            ->limit($limit)
            ->get();

        $total = Capsule::table('tblclients')->count();

        return [
            'data' => $clients,
            'meta' => [
                'page' => $page,
                'limit' => $limit,
                'total' => $total,
                'total_pages' => ceil($total / $limit),
            ],
        ];
    }

    // GET /api/v1/clients/{id}
    public function getClient(array $params, array $body): array {
        $this->auth->requirePermission('clients:read');

        $clientId = (int) $params['id'];
        $client = Capsule::table('tblclients')->find($clientId);

        if (!$client) {
            throw new \Exception('Client not found', 404);
        }

        // Include related data
        $clientData = (array) $client;
        $clientData['services'] = $this->getClientServices($clientId);
        $clientData['invoices'] = $this->getClientInvoices($clientId);

        return ['data' => $clientData];
    }

    // POST /api/v1/clients
    public function createClient(array $params, array $body): array {
        $this->auth->requirePermission('clients:write');

        $this->validateRequired($body, [
            'firstname', 'lastname', 'email', 'password'
        ]);

        // Check for duplicate email
        $existing = Capsule::table('tblclients')
            ->where('email', $body['email'])
            ->first();

        if ($existing) {
            throw new \Exception('Email already exists', 409);
        }

        $clientId = Capsule::table('tblclients')->insertGetId([
            'firstname' => $body['firstname'],
            'lastname' => $body['lastname'],
            'email' => $body['email'],
            'password' => \WHMCS\Input\Sanitize::encode($body['password']),
            'country' => $body['country'] ?? '',
            'state' => $body['state'] ?? '',
            'city' => $body['city'] ?? '',
            'address1' => $body['address1'] ?? '',
            'phonenumber' => $body['phonenumber'] ?? '',
            'datecreated' => date('Y-m-d'),
            'status' => 'Active',
        ]);

        return [
            'data' => ['id' => $clientId],
            'message' => 'Client created successfully',
        ];
    }

    // PUT /api/v1/clients/{id}
    public function updateClient(array $params, array $body): array {
        $this->auth->requirePermission('clients:write');

        $clientId = (int) $params['id'];
        $client = Capsule::table('tblclients')->find($clientId);

        if (!$client) {
            throw new \Exception('Client not found', 404);
        }

        $allowedFields = ['firstname', 'lastname', 'email', 'country', 'state', 'city', 'address1', 'phonenumber'];
        $updateData = array_intersect_key($body, array_flip($allowedFields));

        if (empty($updateData)) {
            throw new \Exception('No valid fields to update');
        }

        Capsule::table('tblclients')
            ->where('id', $clientId)
            ->update($updateData);

        return [
            'data' => Capsule::table('tblclients')->find($clientId),
            'message' => 'Client updated successfully',
        ];
    }

    // DELETE /api/v1/clients/{id}
    public function deleteClient(array $params, array $body): array {
        $this->auth->requirePermission('clients:delete');

        $clientId = (int) $params['id'];
        $client = Capsule::table('tblclients')->find($clientId);

        if (!$client) {
            throw new \Exception('Client not found', 404);
        }

        // Soft delete or archive
        Capsule::table('tblclients')
            ->where('id', $clientId)
            ->update(['status' => 'Closed']);

        return ['message' => 'Client deleted successfully'];
    }

    private function getClientServices(int $clientId): array {
        return Capsule::table('tblhosting')
            ->where('userid', $clientId)
            ->get()
            ->toArray();
    }

    private function getClientInvoices(int $clientId): array {
        return Capsule::table('tblinvoices')
            ->where('userid', $clientId)
            ->orderBy('id', 'DESC')
            ->limit(10)
            ->get()
            ->toArray();
    }

    private function validateRequired(array $data, array $required): void {
        $missing = [];
        foreach ($required as $field) {
            if (empty($data[$field])) {
                $missing[] = $field;
            }
        }
        if (!empty($missing)) {
            throw new \Exception('Missing required fields: ' . implode(', ', $missing), 422);
        }
    }
}
```

### 3. Authentication Service

```php
<?php
// modules/addons/myapi/api/AuthService.php

namespace WHMCS\Module\Addon\MyApi\Api;

use WHMCS\Database\Capsule;

class AuthService {
    private ?array $currentUser = null;

    public function authenticate(): bool {
        $token = $this->getBearerToken();

        if (!$token) {
            return false;
        }

        $apiKey = Capsule::table('mod_api_keys')
            ->where('token', hash('sha256', $token))
            ->where('active', 1)
            ->where('expires_at', '>', date('Y-m-d H:i:s'))
            ->first();

        if (!$apiKey) {
            return false;
        }

        $this->currentUser = [
            'id' => $apiKey->user_id,
            'type' => $apiKey->user_type,
            'permissions' => json_decode($apiKey->permissions, true) ?? [],
        ];

        // Update last used timestamp
        Capsule::table('mod_api_keys')
            ->where('id', $apiKey->id)
            ->update(['last_used_at' => date('Y-m-d H:i:s')]);

        return true;
    }

    public function requirePermission(string $permission): void {
        if (!$this->currentUser) {
            $this->sendUnauthorized('Authentication required');
        }

        if (!$this->hasPermission($permission)) {
            $this->sendForbidden('Insufficient permissions');
        }
    }

    public function hasPermission(string $permission): bool {
        if (!$this->currentUser) {
            return false;
        }

        // Admin users have all permissions
        if ($this->currentUser['type'] === 'admin') {
            return true;
        }

        return in_array($permission, $this->currentUser['permissions'] ?? []);
    }

    private function getBearerToken(): ?string {
        $headers = getallheaders();
        $authHeader = $headers['Authorization'] ?? $headers['authorization'] ?? '';

        if (preg_match('/Bearer\s+(.+)/i', $authHeader, $matches)) {
            return $matches[1];
        }

        return null;
    }

    private function sendUnauthorized(string $message): void {
        http_response_code(401);
        header('Content-Type: application/json');
        echo json_encode(['error' => $message]);
        exit;
    }

    private function sendForbidden(string $message): void {
        http_response_code(403);
        header('Content-Type: application/json');
        echo json_encode(['error' => $message]);
        exit;
    }

    public function generateToken(int $userId, string $userType, array $permissions, int $expiresIn = 86400): array {
        $token = bin2hex(random_bytes(32));

        Capsule::table('mod_api_keys')->insert([
            'user_id' => $userId,
            'user_type' => $userType,
            'token' => hash('sha256', $token),
            'permissions' => json_encode($permissions),
            'created_at' => date('Y-m-d H:i:s'),
            'expires_at' => date('Y-m-d H:i:s', time() + $expiresIn),
            'active' => 1,
        ]);

        return [
            'token' => $token,
            'expires_at' => date('c', time() + $expiresIn),
        ];
    }
}
```

### 4. Input Validation Middleware

```php
<?php
// modules/addons/myapi/api/ValidationMiddleware.php

namespace WHMCS\Module\Addon\MyApi\Api;

class ValidationMiddleware {
    public static function validateClientCreate(array $data): array {
        $errors = [];

        if (empty($data['firstname'])) {
            $errors['firstname'] = 'First name is required';
        }

        if (empty($data['lastname'])) {
            $errors['lastname'] = 'Last name is required';
        }

        if (empty($data['email'])) {
            $errors['email'] = 'Email is required';
        } elseif (!filter_var($data['email'], FILTER_VALIDATE_EMAIL)) {
            $errors['email'] = 'Invalid email format';
        }

        if (empty($data['password'])) {
            $errors['password'] = 'Password is required';
        } elseif (strlen($data['password']) < 8) {
            $errors['password'] = 'Password must be at least 8 characters';
        }

        // Country validation
        if (!empty($data['country'])) {
            $validCountries = ['US', 'UK', 'CA', 'AU', 'VN', 'DE', 'FR', 'JP'];
            if (!in_array($data['country'], $validCountries)) {
                $errors['country'] = 'Invalid country code';
            }
        }

        return $errors;
    }

    public static function validateInvoiceCreate(array $data): array {
        $errors = [];

        if (empty($data['userid'])) {
            $errors['userid'] = 'User ID is required';
        } elseif (!is_numeric($data['userid'])) {
            $errors['userid'] = 'User ID must be numeric';
        }

        if (empty($data['items']) || !is_array($data['items'])) {
            $errors['items'] = 'At least one item is required';
        } else {
            foreach ($data['items'] as $index => $item) {
                if (empty($item['description'])) {
                    $errors["items.{$index}.description"] = 'Item description is required';
                }
                if (!isset($item['amount']) || !is_numeric($item['amount'])) {
                    $errors["items.{$index}.amount"] = 'Item amount must be numeric';
                }
            }
        }

        return $errors;
    }
}
```

### 5. Rate Limiting

```php
<?php
// modules/addons/myapi/api/RateLimiter.php

namespace WHMCS\Module\Addon\MyApi\Api;

use WHMCS\Database\Capsule;

class RateLimiter {
    private int $maxRequests;
    private int $windowSeconds;

    public function __construct(int $maxRequests = 100, int $windowSeconds = 60) {
        $this->maxRequests = $maxRequests;
        $this->windowSeconds = $windowSeconds;
    }

    public function check(string $identifier): bool {
        $windowStart = date('Y-m-d H:i:s', time() - $this->windowSeconds);

        // Clean old entries
        Capsule::table('mod_api_rate_limits')
            ->where('created_at', '<', $windowStart)
            ->delete();

        // Count requests in window
        $count = Capsule::table('mod_api_rate_limits')
            ->where('identifier', $identifier)
            ->where('created_at', '>=', $windowStart)
            ->count();

        if ($count >= $this->maxRequests) {
            return false;
        }

        // Record this request
        Capsule::table('mod_api_rate_limits')->insert([
            'identifier' => $identifier,
            'created_at' => date('Y-m-d H:i:s'),
            'endpoint' => $_SERVER['REQUEST_URI'] ?? '',
        ]);

        return true;
    }

    public function getRemainingRequests(string $identifier): int {
        $windowStart = date('Y-m-d H:i:s', time() - $this->windowSeconds);

        $count = Capsule::table('mod_api_rate_limits')
            ->where('identifier', $identifier)
            ->where('created_at', '>=', $windowStart)
            ->count();

        return max(0, $this->maxRequests - $count);
    }

    public function getResetTime(string $identifier): int {
        $latest = Capsule::table('mod_api_rate_limits')
            ->where('identifier', $identifier)
            ->orderBy('created_at', 'DESC')
            ->first();

        if (!$latest) {
            return time();
        }

        $windowEnd = strtotime($latest->created_at) + $this->windowSeconds;
        return max(time(), $windowEnd);
    }
}
```

### 6. API Entry Point

```php
<?php
// modules/addons/myapi/api/index.php

require_once __DIR__ . '/../../../../init.php';
require_once __DIR__ . '/Router.php';
require_once __DIR__ . '/AuthService.php';
require_once __DIR__ . '/ClientController.php';
require_once __DIR__ . '/RateLimiter.php';

use WHMCS\Module\Addon\MyApi\Api\Router;

header('Access-Control-Allow-Origin: *');
header('Access-Control-Allow-Methods: GET, POST, PUT, DELETE, OPTIONS');
header('Access-Control-Allow-Headers: Authorization, Content-Type');
header('X-Content-Type-Options: nosniff');
header('X-Frame-Options: DENY');

// Handle preflight
if ($_SERVER['REQUEST_METHOD'] === 'OPTIONS') {
    http_response_code(204);
    exit;
}

// Rate limiting
$clientIp = $_SERVER['REMOTE_ADDR'] ?? 'unknown';
$rateLimiter = new RateLimiter(100, 60);

if (!$rateLimiter->check($clientIp)) {
    http_response_code(429);
    header('Content-Type: application/json');
    header('Retry-After: ' . ($rateLimiter->getResetTime($clientIp) - time()));
    echo json_encode([
        'error' => 'Rate limit exceeded',
        'retry_after' => $rateLimiter->getResetTime($clientIp),
    ]);
    exit;
}

// Authentication
$auth = new AuthService();
if (!$auth->authenticate()) {
    http_response_code(401);
    header('Content-Type: application/json');
    echo json_encode(['error' => 'Unauthorized']);
    exit;
}

// Dispatch request
$router = new Router();
$router->dispatch($_SERVER['REQUEST_METHOD'], $_SERVER['REQUEST_URI']);
```

## REST API Naming Conventions

| Resource | Endpoint | Methods |
|----------|----------|---------|
| Clients | /api/v1/clients | GET, POST |
| Client | /api/v1/clients/{id} | GET, PUT, DELETE |
| Invoices | /api/v1/invoices | GET, POST |
| Invoice | /api/v1/invoices/{id} | GET, PUT |
| Services | /api/v1/services | GET, POST |
| Service | /api/v1/services/{id} | GET, PUT, DELETE |
| Tickets | /api/v1/tickets | GET, POST |
| Ticket | /api/v1/tickets/{id} | GET, PUT |

## HTTP Status Codes

| Code | Meaning | Use Case |
|------|---------|----------|
| 200 | OK | Successful GET, PUT |
| 201 | Created | Successful POST |
| 204 | No Content | Successful DELETE |
| 400 | Bad Request | Validation errors |
| 401 | Unauthorized | Missing/invalid auth |
| 403 | Forbidden | Insufficient permissions |
| 404 | Not Found | Resource doesn't exist |
| 409 | Conflict | Duplicate resource |
| 422 | Unprocessable | Valid format but invalid content |
| 429 | Too Many Requests | Rate limit exceeded |
| 500 | Internal Error | Server error |

## Checklist

- [ ] API router with proper routing
- [ ] Authentication middleware
- [ ] Rate limiting implemented
- [ ] Input validation on all endpoints
- [ ] Proper HTTP status codes
- [ ] JSON response format
- [ ] Error handling with meaningful messages
- [ ] CORS headers configured
- [ ] Rate limit headers in responses
- [ ] Request logging for debugging

---

**Related Skills:**
- whmcs-api-integration
- whmcs-localapi-usage
- whmcs-security-hardening
- whmcs-ajax-patterns

**Reference:**
- WHMCS REST API: https://developers.whmcs.com/api-rest/