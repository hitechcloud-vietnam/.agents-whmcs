# API Error Handling Patterns

Robust error handling is critical for building reliable WHMCS integrations. This guide covers patterns for handling API errors gracefully and providing meaningful feedback.

## Error Response Structure

### Standardized Error Response

```php
<?php
/**
 * Standardized API error response
 */
class ApiError
{
    public string $code;
    public string $message;
    public array $details;
    public ?string $helpUrl;
    public string $timestamp;

    public function __construct(
        string $code,
        string $message,
        array $details = [],
        ?string $helpUrl = null
    ) {
        $this->code = $code;
        $this->message = $message;
        $this->details = $details;
        $this->helpUrl = $helpUrl;
        $this->timestamp = date('c');
    }

    /**
     * Convert to array for JSON response
     */
    public function toArray(): array
    {
        return [
            'success' => false,
            'error' => [
                'code' => $this->code,
                'message' => $this->message,
                'details' => $this->details,
                'helpUrl' => $this->helpUrl,
                'timestamp' => $this->timestamp,
            ],
        ];
    }

    /**
     * Output JSON error response
     */
    public function send(int $httpCode = 400): void
    {
        http_response_code($httpCode);
        header('Content-Type: application/json');
        echo json_encode($this->toArray());
    }
}
```

## Error Categories

### Error Type Hierarchy

```php
<?php
/**
 * Error type classifications for WHMCS API
 */
abstract class ErrorType
{
    const VALIDATION_ERROR = 'VALIDATION_ERROR';
    const AUTHENTICATION_ERROR = 'AUTH_ERROR';
    const AUTHORIZATION_ERROR = 'AUTHORIZATION_ERROR';
    const RESOURCE_NOT_FOUND = 'NOT_FOUND';
    const RATE_LIMIT_ERROR = 'RATE_LIMIT';
    const TIMEOUT_ERROR = 'TIMEOUT';
    const SERVER_ERROR = 'SERVER_ERROR';
    const NETWORK_ERROR = 'NETWORK_ERROR';
    const EXTERNAL_SERVICE_ERROR = 'EXTERNAL_SERVICE_ERROR';
}

class ApiException extends Exception
{
    protected string $errorType;
    protected int $httpCode;
    protected array $errorData;

    public function __construct(
        string $message,
        string $errorType = ErrorType::SERVER_ERROR,
        int $httpCode = 500,
        array $errorData = [],
        ?Throwable $previous = null
    ) {
        parent::__construct($message, 0, $previous);
        $this->errorType = $errorType;
        $this->httpCode = $httpCode;
        $this->errorData = $errorData;
    }

    public function getErrorType(): string
    {
        return $this->errorType;
    }

    public function getHttpCode(): int
    {
        return $this->httpCode;
    }

    public function getErrorData(): array
    {
        return $this->errorData;
    }
}
```

## HTTP Status Code Mapping

```php
<?php
/**
 * HTTP status code to error type mapping
 */
class HttpStatusMapper
{
    private static $statusMap = [
        400 => ['type' => ErrorType::VALIDATION_ERROR, 'message' => 'Bad Request'],
        401 => ['type' => ErrorType::AUTHENTICATION_ERROR, 'message' => 'Unauthorized'],
        403 => ['type' => ErrorType::AUTHORIZATION_ERROR, 'message' => 'Forbidden'],
        404 => ['type' => ErrorType::RESOURCE_NOT_FOUND, 'message' => 'Not Found'],
        408 => ['type' => ErrorType::TIMEOUT_ERROR, 'message' => 'Request Timeout'],
        429 => ['type' => ErrorType::RATE_LIMIT_ERROR, 'message' => 'Too Many Requests'],
        500 => ['type' => ErrorType::SERVER_ERROR, 'message' => 'Internal Server Error'],
        502 => ['type' => ErrorType::EXTERNAL_SERVICE_ERROR, 'message' => 'Bad Gateway'],
        503 => ['type' => ErrorType::SERVER_ERROR, 'message' => 'Service Unavailable'],
        504 => ['type' => ErrorType::TIMEOUT_ERROR, 'message' => 'Gateway Timeout'],
    ];

    public static function get(int $statusCode): ?array
    {
        return self::$statusMap[$statusCode] ?? null;
    }

    public static function createFromStatus(int $statusCode, ?string $customMessage = null): ApiException
    {
        $mapping = self::get($statusCode);

        if (!$mapping) {
            return new ApiException(
                $customMessage ?? 'Unknown HTTP error',
                ErrorType::SERVER_ERROR,
                $statusCode
            );
        }

        return new ApiException(
            $customMessage ?? $mapping['message'],
            $mapping['type'],
            $statusCode
        );
    }
}
```

## API Error Handler

```php
<?php
/**
 * Centralized API error handler
 */
class ApiErrorHandler
{
    private $logger;
    private $debugMode;

    public function __construct(LogHandler $logger, bool $debugMode = false)
    {
        $this->logger = $logger;
        $this->debugMode = $debugMode;
    }

    /**
     * Handle API exception and return formatted response
     */
    public function handle(Exception $exception, bool $debug = null): array
    {
        $debug = $debug ?? $this->debugMode;

        // Log the error
        $this->logError($exception);

        // Determine error type
        if ($exception instanceof ApiException) {
            return $this->formatApiException($exception, $debug);
        }

        return $this->formatGenericException($exception, $debug);
    }

    private function formatApiException(ApiException $e, bool $debug): array
    {
        $response = [
            'success' => false,
            'error' => [
                'code' => $e->getErrorType(),
                'message' => $e->getMessage(),
            ],
        ];

        if (!empty($e->getErrorData())) {
            $response['error']['details'] = $e->getErrorData();
        }

        if ($debug) {
            $response['debug'] = [
                'file' => $e->getFile(),
                'line' => $e->getLine(),
                'trace' => $e->getTraceAsString(),
            ];
        }

        return $response;
    }

    private function formatGenericException(Exception $e, bool $debug): array
    {
        $response = [
            'success' => false,
            'error' => [
                'code' => 'INTERNAL_ERROR',
                'message' => 'An internal error occurred',
            ],
        ];

        if ($debug) {
            $response['debug'] = [
                'message' => $e->getMessage(),
                'file' => $e->getFile(),
                'line' => $e->getLine(),
            ];
        }

        return $response;
    }

    private function logError(Exception $exception): void
    {
        $this->logger->error('API Error', [
            'message' => $exception->getMessage(),
            'type' => get_class($exception),
            'file' => $exception->getFile(),
            'line' => $exception->getLine(),
            'trace' => $exception->getTraceAsString(),
        ]);
    }
}
```

## Validation Error Handling

```php
<?php
/**
 * Validation error collection and handling
 */
class ValidationErrorHandler
{
    private array $errors = [];

    /**
     * Add a validation error
     */
    public function addError(string $field, string $message, array $context = []): void
    {
        $this->errors[$field][] = [
            'message' => $message,
            'code' => $this->generateErrorCode($message),
            'context' => $context,
        ];
    }

    /**
     * Check if any errors exist
     */
    public function hasErrors(): bool
    {
        return !empty($this->errors);
    }

    /**
     * Get all errors
     */
    public function getErrors(): array
    {
        return $this->errors;
    }

    /**
     * Get errors for a specific field
     */
    public function getFieldErrors(string $field): array
    {
        return $this->errors[$field] ?? [];
    }

    /**
     * Send validation error response
     */
    public function sendResponse(int $httpCode = 422): void
    {
        $response = new ApiError(
            'VALIDATION_ERROR',
            'The given data was invalid',
            $this->errors,
            'https://docs.example.com/validation-errors'
        );

        $response->send($httpCode);
    }

    private function generateErrorCode(string $message): string
    {
        return strtoupper(
            preg_replace('/[^a-zA-Z0-9]+/', '_', trim($message))
        );
    }
}
```

## Retry Logic with Exponential Backoff

```php
<?php
/**
 * Retry logic with exponential backoff
 */
class RetryHandler
{
    private int $maxRetries;
    private array $retryableErrors;
    private float $baseDelay;
    private float $maxDelay;

    public function __construct(
        int $maxRetries = 3,
        array $retryableErrors = [],
        float $baseDelay = 1.0,
        float $maxDelay = 30.0
    ) {
        $this->maxRetries = $maxRetries;
        $this->retryableErrors = $retryableErrors;
        $this->baseDelay = $baseDelay;
        $this->maxDelay = $maxDelay;
    }

    /**
     * Execute with retry logic
     *
     * @param callable $operation
     * @param callable|null $shouldRetry
     */
    public function execute(callable $operation, ?callable $shouldRetry = null): mixed
    {
        $attempts = 0;
        $lastException = null;

        while ($attempts < $this->maxRetries) {
            try {
                return $operation();
            } catch (Exception $e) {
                $attempts++;

                if ($attempts >= $this->maxRetries) {
                    throw $e;
                }

                if ($shouldRetry && !$shouldRetry($e)) {
                    throw $e;
                }

                if (!$this->isRetryable($e)) {
                    throw $e;
                }

                $delay = $this->calculateDelay($attempts);
                $this->logRetry($e, $attempts, $delay);
                sleep($delay);
            }
        }

        throw $lastException;
    }

    private function calculateDelay(int $attempt): float
    {
        $delay = $this->baseDelay * pow(2, $attempt - 1);

        // Add jitter (±25%)
        $jitter = $delay * 0.25 * (mt_rand(0, 100) / 100 - 0.5);
        $delay = $delay + $jitter;

        return min($delay, $this->maxDelay);
    }

    private function isRetryable(Exception $e): bool
    {
        foreach ($this->retryableErrors as $errorType) {
            if ($e instanceof $errorType) {
                return true;
            }
        }

        return false;
    }

    private function logRetry(Exception $e, int $attempt, float $delay): void
    {
        logActivity("Retry attempt $attempt for: " . $e->getMessage());
    }
}
```

## Circuit Breaker Pattern

```php
<?php
/**
 * Circuit breaker for external API calls
 */
class CircuitBreaker
{
    const STATE_CLOSED = 'closed';      // Normal operation
    const STATE_OPEN = 'open';          // Failing, reject calls
    const STATE_HALF_OPEN = 'half_open'; // Testing recovery

    private string $state = self::STATE_CLOSED;
    private int $failureCount = 0;
    private int $successCount = 0;
    private ?int $lastFailureTime = null;

    private int $failureThreshold;
    private int $successThreshold;
    private int $timeout;

    public function __construct(
        int $failureThreshold = 5,
        int $successThreshold = 2,
        int $timeout = 60
    ) {
        $this->failureThreshold = $failureThreshold;
        $this->successThreshold = $successThreshold;
        $this->timeout = $timeout;
    }

    /**
     * Execute operation with circuit breaker protection
     */
    public function execute(callable $operation): mixed
    {
        if (!$this->canExecute()) {
            throw new CircuitOpenException(
                "Circuit breaker is open. Wait {$this->getRemainingTimeout()}s"
            );
        }

        try {
            $result = $operation();
            $this->recordSuccess();
            return $result;
        } catch (Exception $e) {
            $this->recordFailure();
            throw $e;
        }
    }

    private function canExecute(): bool
    {
        if ($this->state === self::STATE_CLOSED) {
            return true;
        }

        if ($this->state === self::STATE_OPEN) {
            if ($this->shouldAttemptReset()) {
                $this->state = self::STATE_HALF_OPEN;
                return true;
            }
            return false;
        }

        // Half-open: allow one test request
        return true;
    }

    private function recordSuccess(): void
    {
        $this->successCount++;

        if ($this->state === self::STATE_HALF_OPEN) {
            if ($this->successCount >= $this->successThreshold) {
                $this->reset();
            }
        }

        $this->failureCount = 0;
    }

    private function recordFailure(): void
    {
        $this->failureCount++;
        $this->lastFailureTime = time();

        if ($this->failureCount >= $this->failureThreshold) {
            $this->state = self::STATE_OPEN;
        }
    }

    private function shouldAttemptReset(): bool
    {
        return time() - $this->lastFailureTime >= $this->timeout;
    }

    private function reset(): void
    {
        $this->state = self::STATE_CLOSED;
        $this->failureCount = 0;
        $this->successCount = 0;
        $this->lastFailureTime = null;
    }

    private function getRemainingTimeout(): int
    {
        if (!$this->lastFailureTime) {
            return 0;
        }
        return max(0, $this->timeout - (time() - $this->lastFailureTime));
    }
}

class CircuitOpenException extends ApiException
{
    public function __construct(string $message)
    {
        parent::__construct(
            $message,
            ErrorType::SERVER_ERROR,
            503,
            ['circuit_state' => 'open']
        );
    }
}
```

## Error Handling Best Practices

1. **Always return consistent error format** - Same structure for all API responses
2. **Include error codes** - Machine-readable error identification
3. **Provide actionable messages** - Help developers understand and fix issues
4. **Log all errors** - Maintain audit trail for debugging
5. **Never expose sensitive information** - Hide internal details in production
6. **Implement retry logic** - Handle transient failures gracefully
7. **Use circuit breakers** - Prevent cascade failures
8. **Document error codes** - Provide reference documentation

## Error Code Reference

| Code | Description | HTTP Status |
|------|-------------|-------------|
| VALIDATION_ERROR | Input validation failed | 422 |
| AUTH_ERROR | Authentication failed | 401 |
| AUTHORIZATION_ERROR | Insufficient permissions | 403 |
| NOT_FOUND | Resource not found | 404 |
| RATE_LIMIT | Too many requests | 429 |
| TIMEOUT | Request timeout | 504 |
| SERVER_ERROR | Internal server error | 500 |
| EXTERNAL_SERVICE_ERROR | External service unavailable | 502 |

## Related Patterns

- [Service Layer](./service-layer.md) - Error handling in services
- [Repository Pattern](./repository-pattern.md) - Data access error handling
- [Admin Security](./admin-security.md) - Security-related error handling