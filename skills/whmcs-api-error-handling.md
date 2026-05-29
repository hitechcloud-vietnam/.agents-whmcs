# WHMCS API Error Handling

## Skill Description
Implement comprehensive error handling for WHMCS API modules including consistent error formats, exception handling, logging, and user-friendly error messages.

## Prerequisites
- WHMCS 7.0+ installation
- PHP 7.4+ with exception support
- Understanding of PHP exception handling
- Logging infrastructure access

## Step-by-Step Implementation

### 1. Exception Classes
```php
<?php
// includes/exceptions/ApiException.php

namespace WHMCS\Module\YourModule\Exceptions;

class ApiException extends \Exception
{
    protected int $statusCode;
    protected array $errorData = [];
    protected string $errorCode;

    public function __construct(
        string $message,
        int $statusCode = 400,
        string $errorCode = 'API_ERROR',
        array $errorData = [],
        ?\Throwable $previous = null
    ) {
        parent::__construct($message, 0, $previous);
        $this->statusCode = $statusCode;
        $this->errorCode = $errorCode;
        $this->errorData = $errorData;
    }

    public function getStatusCode(): int
    {
        return $this->statusCode;
    }

    public function getErrorCode(): string
    {
        return $this->errorCode;
    }

    public function getErrorData(): array
    {
        return $this->errorData;
    }

    public function toArray(): array
    {
        $error = [
            'message' => $this->getMessage(),
            'code' => $this->errorCode
        ];

        if (!empty($this->errorData)) {
            $error['data'] = $this->errorData;
        }

        return $error;
    }
}
```

```php
<?php
// includes/exceptions/ValidationException.php

namespace WHMCS\Module\YourModule\Exceptions;

class ValidationException extends ApiException
{
    private array $validationErrors;

    public function __construct(array $errors, string $message = 'Validation failed')
    {
        parent::__construct($message, 422, 'VALIDATION_ERROR', $errors);
        $this->validationErrors = $errors;
    }

    public function getValidationErrors(): array
    {
        return $this->validationErrors;
    }
}
```

```php
<?php
// includes/exceptions/NotFoundException.php

namespace WHMCS\Module\YourModule\Exceptions;

class NotFoundException extends ApiException
{
    private string $resource;

    public function __construct(string $resource = 'Resource', ?int $resourceId = null)
    {
        $message = $resource . ' not found';
        if ($resourceId) {
            $message .= ': ' . $resourceId;
        }

        parent::__construct($message, 404, 'NOT_FOUND', [
            'resource' => $resource,
            'id' => $resourceId
        ]);

        $this->resource = $resource;
    }

    public function getResource(): string
    {
        return $this->resource;
    }
}
```

```php
<?php
// includes/exceptions/AuthenticationException.php

namespace WHMCS\Module\YourModule\Exceptions;

class AuthenticationException extends ApiException
{
    public function __construct(string $message = 'Authentication required')
    {
        parent::__construct($message, 401, 'UNAUTHORIZED');
    }
}
```

```php
<?php
// includes/exceptions/AuthorizationException.php

namespace WHMCS\Module\YourModule\Exceptions;

class AuthorizationException extends ApiException
{
    public function __construct(string $message = 'Access denied')
    {
        parent::__construct($message, 403, 'FORBIDDEN');
    }
}
```

### 2. Exception Handler
```php
<?php
// includes/exceptions/ExceptionHandler.php

namespace WHMCS\Module\YourModule\Exceptions;

class ExceptionHandler
{
    private bool $debugMode;
    private array $loggedExceptions = [];

    public function __construct(bool $debugMode = false)
    {
        $this->debugMode = $debugMode;
    }

    public function handle(\Throwable $exception): void
    {
        // Log the exception
        $this->logException($exception);

        // Build response
        $response = $this->buildResponse($exception);

        // Send response
        $this->sendResponse($response, $exception);
    }

    private function logException(\Throwable $exception): void
    {
        $logData = [
            'type' => get_class($exception),
            'message' => $exception->getMessage(),
            'code' => $exception->getCode(),
            'file' => $exception->getFile(),
            'line' => $exception->getLine(),
            'trace' => $exception->getTraceAsString(),
            'request_uri' => $_SERVER['REQUEST_URI'] ?? '',
            'request_method' => $_SERVER['REQUEST_METHOD'] ?? '',
            'timestamp' => date('Y-m-d H:i:s')
        ];

        // Log to WHMCS activity log
        logActivity('API Exception: ' . json_encode($logData));

        // Also log to custom log file
        $logFile = dirname(__DIR__) . '/logs/exceptions.log';
        $logDir = dirname($logFile);

        if (!is_dir($logDir)) {
            mkdir($logDir, 0755, true);
        }

        file_put_contents(
            $logFile,
            date('Y-m-d H:i:s') . ' - ' . json_encode($logData) . PHP_EOL,
            FILE_APPEND
        );

        $this->loggedExceptions[] = $logData;
    }

    private function buildResponse(\Throwable $exception): array
    {
        if ($exception instanceof ApiException) {
            $response = [
                'success' => false,
                'error' => $exception->toArray()
            ];

            if ($this->debugMode) {
                $response['debug'] = [
                    'file' => $exception->getFile(),
                    'line' => $exception->getLine(),
                    'trace' => explode("\n", $exception->getTraceAsString())
                ];
            }

            return $response;
        }

        // Generic exception handling
        $response = [
            'success' => false,
            'error' => [
                'message' => $this->debugMode ? $exception->getMessage() : 'An unexpected error occurred',
                'code' => 'SERVER_ERROR'
            ]
        ];

        if ($this->debugMode) {
            $response['debug'] = [
                'exception' => get_class($exception),
                'message' => $exception->getMessage(),
                'file' => $exception->getFile(),
                'line' => $exception->getLine(),
                'trace' => explode("\n", $exception->getTraceAsString())
            ];
        }

        return $response;
    }

    private function sendResponse(array $response, \Throwable $exception): void
    {
        $statusCode = 500;

        if ($exception instanceof ApiException) {
            $statusCode = $exception->getStatusCode();
        }

        http_response_code($statusCode);
        header('Content-Type: application/json');
        header('X-Content-Type-Options: nosniff');

        echo json_encode($response, JSON_PRETTY_PRINT | JSON_UNESCAPED_SLASHES);
    }

    public function register(): void
    {
        set_exception_handler([$this, 'handle']);
    }
}
```

### 3. Error Handler Trait
```php
<?php
// includes/exceptions/HandlesExceptions.php

namespace WHMCS\Module\YourModule\Traits;

trait HandlesExceptions
{
    protected function tryCatch(callable $callback, ?callable $onError = null): mixed
    {
        try {
            return $callback();
        } catch (\Throwable $e) {
            if ($onError) {
                return $onError($e);
            }

            throw $e;
        }
    }

    protected function tryOrLog(callable $callback, string $logMessage = 'Operation failed'): bool
    {
        try {
            $callback();
            return true;
        } catch (\Throwable $e) {
            logActivity($logMessage . ': ' . $e->getMessage());
            return false;
        }
    }

    protected function validateRequired(array $data, array $requiredFields): void
    {
        $errors = [];

        foreach ($requiredFields as $field) {
            if (!isset($data[$field]) || (is_string($data[$field]) && trim($data[$field]) === '')) {
                $errors[$field] = "The {$field} field is required";
            }
        }

        if (!empty($errors)) {
            throw new \WHMCS\Module\YourModule\Exceptions\ValidationException($errors);
        }
    }

    protected function validateEmail(string $email): void
    {
        if (!filter_var($email, FILTER_VALIDATE_EMAIL)) {
            throw new \WHMCS\Module\YourModule\Exceptions\ValidationException([
                'email' => 'The email address is invalid'
            ]);
        }
    }

    protected function assertFound(mixed $resource, string $resourceType, ?int $resourceId = null): void
    {
        if ($resource === null) {
            throw new \WHMCS\Module\YourModule\Exceptions\NotFoundException($resourceType, $resourceId);
        }
    }

    protected function assertAuthorized(bool $condition, string $message = 'Access denied'): void
    {
        if (!$condition) {
            throw new \WHMCS\Module\YourModule\Exceptions\AuthorizationException($message);
        }
    }
}
```

### 4. Usage Example
```php
<?php
// includes/api/Controllers/UserController.php

namespace WHMCS\Module\YourModule\Api\Controllers;

use WHMCS\Module\YourModule\Api\ApiController;
use WHMCS\Module\YourModule\Api\ApiResponse;
use WHMCS\Module\YourModule\Exceptions\ValidationException;
use WHMCS\Module\YourModule\Exceptions\NotFoundException;
use WHMCS\Module\YourModule\Traits\HandlesExceptions;

class UserController extends ApiController
{
    use HandlesExceptions;

    public function show(): void
    {
        try {
            $userId = $this->getRequiredParam('id');

            $this->validateRequired(['id' => $userId], ['id']);

            $user = $this->getUser($userId);

            $this->assertFound($user, 'User', $userId);

            ApiResponse::success(['user' => $user])->send();

        } catch (ValidationException $e) {
            ApiResponse::validationError($e->getValidationErrors())->send();
        } catch (NotFoundException $e) {
            ApiResponse::notFound($e->getResource())->send();
        }
    }

    public function store(): void
    {
        try {
            $this->validateRequired($this->requestData, [
                'firstname',
                'lastname',
                'email'
            ]);

            $this->validateEmail($this->requestData['email']);

            $user = $this->createUser($this->requestData);

            ApiResponse::created([
                'user_id' => $user->id,
                'email' => $user->email
            ])->send();

        } catch (ValidationException $e) {
            ApiResponse::validationError($e->getValidationErrors())->send();
        }
    }

    private function getUser(int $id): ?array
    {
        // Implementation
        return \WHMCS\User\User::find($id);
    }

    private function createUser(array $data): \WHMCS\User\User
    {
        // Implementation
        return \WHMCS\User\User::create($data);
    }
}
```

### 5. Global Error Handler Setup
```php
<?php
// bootstrap.php

// Register exception handler early in bootstrap
$exceptionHandler = new \WHMCS\Module\YourModule\Exceptions\ExceptionHandler(
    debugMode: (bool)($config['debug_mode'] ?? false)
);
$exceptionHandler->register();

// Set error handler for non-fatal errors
set_error_handler(function (int $errno, string $errstr, string $errfile, int $errline) {
    if (!(error_reporting() & $errno)) {
        return false;
    }

    throw new \ErrorException($errstr, 0, $errno, $errfile, $errline);
});
```

## Common Pitfalls and Solutions

| Pitfall | Solution |
|---------|----------|
| Exposing stack traces in production | Use debug mode flag to control error detail |
| Losing exception context | Always wrap exceptions with previous |
| Not logging all errors | Use try-catch at top level with global handler |
| Inconsistent error formats | Create specific exception classes |
| Missing error codes | Use unique error codes for each error type |

## Security Considerations

1. **Never expose sensitive data in errors** - Filter stack traces, file paths, SQL queries
2. **Log errors for debugging** - Store detailed errors server-side
3. **Use generic messages for users** - Hide technical details from end users
4. **Implement proper error codes** - Enable client-side error handling
5. **Handle both exceptions and errors** - Set up both exception and error handlers

## Testing Checklist

- [ ] Test each custom exception type
- [ ] Verify error format consistency
- [ ] Test validation errors with multiple fields
- [ ] Test error codes are unique
- [ ] Verify logging captures all errors
- [ ] Test stack trace hiding in production
- [ ] Test error recovery scenarios
- [ ] Test error propagation through layers
- [ ] Verify Content-Type header on errors
- [ ] Test debug mode vs production mode

## Reference Links

- [PHP Exception Handling](https://www.php.net/manual/en/language.exceptions.php)
- [WHMCS Logging](https://developers.whmcs.com/advanced/logging/)
- [REST API Error Handling Best Practices](https://www.restapitutorial.com/lessons/httpmethods.html)
