# WHMCS API Response Formatting

## Skill Description
Implement standardized API response formatting for WHMCS modules to ensure consistent, predictable responses across all API endpoints with proper status codes, error handling, and metadata.

## Prerequisites
- WHMCS 7.0+ installation
- PHP 7.4+ with JSON support
- Understanding of REST API principles
- Basic knowledge of HTTP status codes

## Step-by-Step Implementation

### 1. Response Formatter Class
```php
<?php
// includes/api/ApiResponse.php

namespace WHMCS\Module\YourModule\Api;

class ApiResponse
{
    private $statusCode;
    private $data;
    private $errors = [];
    private $metadata = [];
    private $pagination = null;

    public function __construct(int $statusCode = 200)
    {
        $this->statusCode = $statusCode;
    }

    public static function success(mixed $data = null, array $metadata = []): self
    {
        $response = new self(200);
        $response->data = $data;
        $response->metadata = $metadata;
        return $response;
    }

    public static function created(mixed $data = null, array $metadata = []): self
    {
        $response = new self(201);
        $response->data = $data;
        $response->metadata = $metadata;
        return $response;
    }

    public static function noContent(): self
    {
        return new self(204);
    }

    public static function error(
        string $message,
        int $statusCode = 400,
        array $errors = [],
        string $errorCode = null
    ): self {
        $response = new self($statusCode);
        $response->errors = [
            'message' => $message,
            'code' => $errorCode,
            'details' => $errors
        ];
        return $response;
    }

    public static function validationError(array $errors): self
    {
        return self::error('Validation failed', 422, $errors, 'VALIDATION_ERROR');
    }

    public static function notFound(string $resource = 'Resource'): self
    {
        return self::error($resource . ' not found', 404, [], 'NOT_FOUND');
    }

    public static function unauthorized(string $message = 'Unauthorized'): self
    {
        return self::error($message, 401, [], 'UNAUTHORIZED');
    }

    public static function forbidden(string $message = 'Forbidden'): self
    {
        return self::error($message, 403, [], 'FORBIDDEN');
    }

    public static function serverError(string $message = 'Internal server error'): self
    {
        return self::error($message, 500, [], 'SERVER_ERROR');
    }

    public function setPagination(
        int $currentPage,
        int $perPage,
        int $totalItems,
        int $totalPages
    ): self {
        $this->pagination = [
            'current_page' => $currentPage,
            'per_page' => $perPage,
            'total_items' => $totalItems,
            'total_pages' => $totalPages,
            'has_next' => $currentPage < $totalPages,
            'has_prev' => $currentPage > 1
        ];
        return $this;
    }

    public function addMeta(string $key, mixed $value): self
    {
        $this->metadata[$key] = $value;
        return $this;
    }

    public function toArray(): array
    {
        $response = [
            'success' => $this->statusCode >= 200 && $this->statusCode < 300
        ];

        if ($this->statusCode >= 200 && $this->statusCode < 300) {
            $response['data'] = $this->data;

            if (!empty($this->metadata)) {
                $response['meta'] = $this->metadata;
            }

            if ($this->pagination) {
                $response['pagination'] = $this->pagination;
            }
        } else {
            $response['error'] = $this->errors;

            if (!empty($this->metadata)) {
                $response['meta'] = $this->metadata;
            }
        }

        return $response;
    }

    public function send(): void
    {
        http_response_code($this->statusCode);
        header('Content-Type: application/json');
        header('X-Content-Type-Options: nosniff');
        echo json_encode($this->toArray(), JSON_PRETTY_PRINT | JSON_UNESCAPED_SLASHES);
    }
}
```

### 2. API Controller Base Class
```php
<?php
// includes/api/ApiController.php

namespace WHMCS\Module\YourModule\Api;

abstract class ApiController
{
    protected array $requestData = [];
    protected array $headers = [];

    public function __construct()
    {
        $this->parseRequest();
    }

    private function parseRequest(): void
    {
        // Get JSON body
        $input = file_get_contents('php://input');
        $contentType = $_SERVER['CONTENT_TYPE'] ?? '';

        if (stripos($contentType, 'application/json') !== false && !empty($input)) {
            $decoded = json_decode($input, true);
            if (json_last_error() === JSON_ERROR_NONE) {
                $this->requestData = $decoded;
            }
        }

        // Merge GET and POST parameters
        $this->requestData = array_merge($_GET, $_POST, $this->requestData);

        // Get headers
        $this->headers = $this->getRequestHeaders();
    }

    private function getRequestHeaders(): array
    {
        $headers = [];
        foreach ($_SERVER as $key => $value) {
            if (strpos($key, 'HTTP_') === 0) {
                $header = str_replace(' ', '-', ucwords(strtolower(str_replace('_', ' ', substr($key, 5)))));
                $headers[$header] = $value;
            }
        }
        return $headers;
    }

    protected function getHeader(string $name, mixed $default = null): mixed
    {
        return $this->headers[$name] ?? $default;
    }

    protected function getParam(string $key, mixed $default = null): mixed
    {
        return $this->requestData[$key] ?? $default;
    }

    protected function getRequiredParam(string $key): mixed
    {
        if (!isset($this->requestData[$key])) {
            ApiResponse::validationError([
                $key => 'The ' . $key . ' field is required'
            ])->send();
            exit;
        }
        return $this->requestData[$key];
    }

    protected function paginate(
        \Illuminate\Database\Eloquent\Builder $query,
        int $defaultPerPage = 20,
        int $maxPerPage = 100
    ): array {
        $page = max(1, (int) $this->getParam('page', 1));
        $perPage = min($maxPerPage, max(1, (int) $this->getParam('per_page', $defaultPerPage)));

        $total = $query->count();
        $totalPages = (int) ceil($total / $perPage);
        $offset = ($page - 1) * $perPage;

        $items = $query->skip($offset)->take($perPage)->get();

        $response = ApiResponse::success($items);
        $response->setPagination($page, $perPage, $total, $totalPages);

        return $response->toArray();
    }

    protected function requireAuth(): ?array
    {
        $auth = new \WHMCS\Module\YourModule\Auth\ApiKeyAuthenticator();
        $user = $auth->validateRequest();

        if (!$user) {
            ApiResponse::unauthorized('Invalid or missing API key')->send();
            exit;
        }

        return $user;
    }
}
```

### 3. Example API Endpoint
```php
<?php
// includes/api/Controllers/InvoiceController.php

namespace WHMCS\Module\YourModule\Api\Controllers;

use WHMCS\Module\YourModule\Api\ApiController;
use WHMCS\Module\YourModule\Api\ApiResponse;
use WHMCS\Module\YourModule\Services\InvoiceService;

class InvoiceController extends ApiController
{
    private InvoiceService $invoiceService;

    public function __construct()
    {
        parent::__construct();
        $this->invoiceService = new InvoiceService();
    }

    public function index(): void
    {
        $this->requireAuth();

        $query = \WHMCS\Invoices\Invoice::query();
        $query->orderBy('created_at', 'desc');

        // Apply filters
        if ($status = $this->getParam('status')) {
            $query->where('status', $status);
        }

        if ($userId = $this->getParam('user_id')) {
            $query->where('userid', $userId);
        }

        $response = $this->paginate($query);
        $response->send();
    }

    public function show(): void
    {
        $this->requireAuth();

        $invoiceId = $this->getRequiredParam('id');

        $invoice = \WHMCS\Invoices\Invoice::find($invoiceId);

        if (!$invoice) {
            ApiResponse::notFound('Invoice')->send();
            return;
        }

        ApiResponse::success([
            'invoice' => $invoice->toArray(),
            'items' => $invoice->items,
            'client' => $invoice->client
        ])->send();
    }

    public function store(): void
    {
        $user = $this->requireAuth();

        // Validate required fields
        $errors = $this->validateStoreRequest();
        if (!empty($errors)) {
            ApiResponse::validationError($errors)->send();
            return;
        }

        try {
            $invoice = $this->invoiceService->create([
                'user_id' => $this->getParam('user_id'),
                'items' => $this->getParam('items', []),
                'notes' => $this->getParam('notes', ''),
                'due_date' => $this->getParam('due_date')
            ]);

            ApiResponse::created([
                'invoice_id' => $invoice->id,
                'invoice_number' => $invoice->invoicenum
            ])->send();

        } catch (\Exception $e) {
            ApiResponse::serverError('Failed to create invoice: ' . $e->getMessage())->send();
        }
    }

    public function update(): void
    {
        $this->requireAuth();

        $invoiceId = $this->getRequiredParam('id');

        $invoice = \WHMCS\Invoices\Invoice::find($invoiceId);

        if (!$invoice) {
            ApiResponse::notFound('Invoice')->send();
            return;
        }

        try {
            $this->invoiceService->update($invoice, $this->requestData);

            ApiResponse::success([
                'message' => 'Invoice updated successfully',
                'invoice_id' => $invoice->id
            ])->send();

        } catch (\Exception $e) {
            ApiResponse::serverError('Failed to update invoice: ' . $e->getMessage())->send();
        }
    }

    public function delete(): void
    {
        $this->requireAuth();

        $invoiceId = $this->getRequiredParam('id');

        $invoice = \WHMCS\Invoices\Invoice::find($invoiceId);

        if (!$invoice) {
            ApiResponse::notFound('Invoice')->send();
            return;
        }

        try {
            $this->invoiceService->delete($invoice);

            ApiResponse::noContent()->send();

        } catch (\Exception $e) {
            ApiResponse::serverError('Failed to delete invoice: ' . $e->getMessage())->send();
        }
    }

    private function validateStoreRequest(): array
    {
        $errors = [];

        if (empty($this->getParam('user_id'))) {
            $errors['user_id'] = 'User ID is required';
        }

        if (empty($this->getParam('items')) || !is_array($this->getParam('items'))) {
            $errors['items'] = 'At least one item is required';
        }

        return $errors;
    }
}
```

### 4. API Router
```php
<?php
// includes/api/ApiRouter.php

namespace WHMCS\Module\YourModule\Api;

class ApiRouter
{
    private array $routes = [];

    public function register(string $method, string $path, callable $handler): self
    {
        $this->routes[] = [
            'method' => strtoupper($method),
            'path' => $path,
            'handler' => $handler
        ];
        return $this;
    }

    public function get(string $path, callable $handler): self
    {
        return $this->register('GET', $path, $handler);
    }

    public function post(string $path, callable $handler): self
    {
        return $this->register('POST', $path, $handler);
    }

    public function put(string $path, callable $handler): self
    {
        return $this->register('PUT', $path, $handler);
    }

    public function patch(string $path, callable $handler): self
    {
        return $this->register('PATCH', $path, $handler);
    }

    public function delete(string $path, callable $handler): self
    {
        return $this->register('DELETE', $path, $handler);
    }

    public function dispatch(): void
    {
        $method = $_SERVER['REQUEST_METHOD'];
        $path = parse_url($_SERVER['REQUEST_URI'], PHP_URL_PATH);

        foreach ($this->routes as $route) {
            if ($route['method'] !== $method) {
                continue;
            }

            $params = $this->matchPath($route['path'], $path);

            if ($params !== false) {
                // Merge URL params with request data
                $_GET = array_merge($_GET, $params);

                try {
                    call_user_func_array($route['handler'], $params);
                } catch (\Exception $e) {
                    ApiResponse::serverError('An unexpected error occurred')->send();
                }
                return;
            }
        }

        ApiResponse::error('Endpoint not found', 404, [], 'NOT_FOUND')->send();
    }

    private function matchPath(string $pattern, string $path): array|false
    {
        // Convert {param} to regex
        $regex = preg_replace('/\{([a-zA-Z_]+)\}/', '(?P<$1>[^/]+)', $pattern);
        $regex = '#^' . $regex . '$#';

        if (preg_match($regex, $path, $matches)) {
            return array_filter($matches, 'is_string', ARRAY_FILTER_USE_KEY);
        }

        return false;
    }
}
```

## Common Pitfalls and Solutions

| Pitfall | Solution |
|---------|----------|
| Inconsistent error formats | Use ApiResponse factory methods consistently |
| Missing Content-Type header | Always set Content-Type: application/json |
| Leaking sensitive data in errors | Sanitize error messages before sending |
| Missing pagination metadata | Always include pagination when returning lists |
| Not handling JSON decode errors | Check json_last_error() before using data |

## Security Considerations

1. **Never expose stack traces** - Always return generic error messages
2. **Sanitize error messages** - Remove sensitive data from responses
3. **Use appropriate status codes** - Don't return 200 for errors
4. **Implement CORS properly** - Set appropriate CORS headers
5. **Rate limit responses** - Prevent enumeration attacks

## Testing Checklist

- [ ] Test success responses with 200 status
- [ ] Test created responses with 201 status
- [ ] Test validation errors with 422 status
- [ ] Test not found errors with 404 status
- [ ] Test unauthorized errors with 401 status
- [ ] Test pagination metadata
- [ ] Test JSON encoding/decoding
- [ ] Test Content-Type header presence
- [ ] Test CORS headers
- [ ] Test error response format consistency

## Reference Links

- [REST API Design Best Practices](https://restfulapi.net/)
- [HTTP Status Codes Guide](https://developer.mozilla.org/en-US/docs/Web/HTTP/Status)
- [WHMCS API Documentation](https://developers.whmcs.com/api/)
