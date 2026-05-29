# WHMCS API Error Handling

## Overview

Proper error handling is essential for robust WHMCS API integrations. This guide covers error codes, response formats, and best practices.

## Error Response Format

```json
{
    "result": "error",
    "message": "Client not found",
    "errorcode": "1001"
}
```

## Standard Error Codes

### Client Errors (1000-1999)

| Code | Message | Description |
|------|---------|-------------|
| 1001 | Client Not Found | No client matches the provided ID |
| 1002 | Client Create Failed | Unable to create client record |
| 1003 | Client Update Failed | Unable to update client record |
| 1004 | Invalid Email Address | Email format is invalid |
| 1005 | Email Already Exists | Email address already registered |

### Authentication Errors (2000-2999)

| Code | Message | Description |
|------|---------|-------------|
| 2001 | Authentication Failed | Invalid API credentials |
| 2002 | Invalid API Key | API key is malformed or expired |
| 2003 | Unauthorized | Insufficient permissions |
| 2004 | Rate Limit Exceeded | Too many requests |
| 2005 | Session Expired | Admin session has expired |

### Validation Errors (3000-3999)

| Code | Message | Description |
|------|---------|-------------|
| 3001 | Required Field Missing | A required parameter was not provided |
| 3002 | Invalid Parameter | Parameter value is invalid |
| 3003 | Invalid Format | Data format is incorrect |
| 3004 | Value Out of Range | Numeric value exceeds limits |

### System Errors (4000-4999)

| Code | Message | Description |
|------|---------|-------------|
| 4001 | Database Error | Database operation failed |
| 4002 | External API Error | Third-party service unavailable |
| 4003 | Configuration Error | System misconfiguration |
| 4004 | Maintenance Mode | System is in maintenance |

## Implementation

### Error Handler Class

```php
<?php
class WhmcsApiException extends Exception {
    private int $errorCode;
    private array $errors;
    
    public function __construct(
        string $message,
        int $errorCode = 0,
        array $errors = [],
        int $code = 0,
        ?Throwable $previous = null
    ) {
        parent::__construct($message, $code, $previous);
        $this->errorCode = $errorCode;
        $this->errors = $errors;
    }
    
    public function getErrorCode(): int
    {
        return $this->errorCode;
    }
    
    public function getErrors(): array
    {
        return $this->errors;
    }
    
    public function getCategory(): string
    {
        if ($this->errorCode >= 1000 && $this->errorCode < 2000) {
            return 'client';
        } elseif ($this->errorCode >= 2000 && $this->errorCode < 3000) {
            return 'authentication';
        } elseif ($this->errorCode >= 3000 && $this->errorCode < 4000) {
            return 'validation';
        } elseif ($this->errorCode >= 4000 && $this->errorCode < 5000) {
            return 'system';
        }
        return 'unknown';
    }
}

class WhmcsApiClient {
    public function makeRequest(array $params): array
    {
        $response = $this->executeRequest($params);
        
        if ($this->isErrorResponse($response)) {
            throw $this->createException($response);
        }
        
        return $response;
    }
    
    private function isErrorResponse(array $response): bool
    {
        return isset($response['result']) && $response['result'] === 'error';
    }
    
    private function createException(array $response): WhmcsApiException
    {
        $errorCode = (int) ($response['errorcode'] ?? 0);
        $message = $response['message'] ?? 'Unknown error';
        
        $errors = [];
        if (isset($response['errors']) && is_array($response['errors'])) {
            foreach ($response['errors'] as $field => $error) {
                $errors[] = [
                    'field' => $field,
                    'message' => is_array($error) ? $error['message'] : $error,
                ];
            }
        }
        
        return new WhmcsApiException($message, $errorCode, $errors);
    }
}
```

### Detailed Error Handler

```php
<?php
class DetailedErrorHandler {
    private array $errorDefinitions = [
        1001 => [
            'category' => 'client',
            'retryable' => false,
            'action' => 'Verify client ID exists',
        ],
        2001 => [
            'category' => 'authentication',
            'retryable' => false,
            'action' => 'Check API credentials',
        ],
        2004 => [
            'category' => 'rate_limit',
            'retryable' => true,
            'action' => 'Implement backoff strategy',
        ],
        4001 => [
            'category' => 'database',
            'retryable' => true,
            'action' => 'Retry with exponential backoff',
        ],
        4002 => [
            'category' => 'external',
            'retryable' => true,
            'action' => 'Check service status and retry',
        ],
    ];
    
    public function handle(WhmcsApiException $exception): ErrorAction
    {
        $errorCode = $exception->getErrorCode();
        $definition = $this->errorDefinitions[$errorCode] ?? [
            'category' => 'unknown',
            'retryable' => false,
            'action' => 'Contact support',
        ];
        
        return new ErrorAction(
            category: $definition['category'],
            retryable: $definition['retryable'],
            action: $definition['action'],
            userMessage: $exception->getMessage(),
            details: $exception->getErrors(),
        );
    }
    
    public function logError(WhmcsApiException $exception, array $context = []): void
    {
        $logEntry = [
            'timestamp' => date('Y-m-d H:i:s'),
            'error_code' => $exception->getErrorCode(),
            'message' => $exception->getMessage(),
            'category' => $exception->getCategory(),
            'errors' => $exception->getErrors(),
            'context' => $context,
        ];
        
        error_log(json_encode($logEntry, JSON_PRETTY_PRINT));
    }
}

class ErrorAction {
    public function __construct(
        public readonly string $category,
        public readonly bool $retryable,
        public readonly string $action,
        public readonly string $userMessage,
        public readonly array $details = []
    ) {}
}
```

### Retry Logic with Error Handling

```php
<?php
class ResilientApiClient {
    private int $maxRetries;
    private array $retryableErrors = [2004, 4001, 4002];
    
    public function __construct(int $maxRetries = 3)
    {
        $this->maxRetries = $maxRetries;
    }
    
    public function executeWithRetry(callable $operation, array $context = []): mixed
    {
        $attempt = 0;
        $lastException = null;
        
        while ($attempt < $this->maxRetries) {
            try {
                return $operation();
            } catch (WhmcsApiException $e) {
                $lastException = $e;
                $attempt++;
                
                $this->logAttempt($attempt, $e, $context);
                
                if (!$this->isRetryable($e) || $attempt >= $this->maxRetries) {
                    $this->handleNonRetryable($e, $context);
                    throw $e;
                }
                
                $delay = $this->calculateDelay($attempt, $e);
                sleep($delay);
            }
        }
        
        throw $lastException;
    }
    
    private function isRetryable(WhmcsApiException $e): bool
    {
        return in_array($e->getErrorCode(), $this->retryableErrors);
    }
    
    private function calculateDelay(int $attempt, WhmcsApiException $e): int
    {
        $baseDelay = 1;
        $exponentialDelay = pow(2, $attempt - 1);
        $jitter = random_int(0, 1000) / 1000;
        
        return (int) (($baseDelay * $exponentialDelay) + $jitter);
    }
    
    private function logAttempt(int $attempt, WhmcsApiException $e, array $context): void
    {
        error_log(sprintf(
            "[WHMCS API] Attempt %d/%d failed: %s (Code: %d) - %s",
            $attempt,
            $this->maxRetries,
            $e->getMessage(),
            $e->getErrorCode(),
            json_encode($context)
        ));
    }
    
    private function handleNonRetryable(WhmcsApiException $e, array $context): void
    {
        // Send alerts for non-retryable errors
        if ($e->getErrorCode() >= 2000 && $e->getErrorCode() < 3000) {
            $this->alertAdmins('Authentication Error', $e->getMessage());
        }
        
        // Log for debugging
        $this->logForDebug($e, $context);
    }
    
    private function alertAdmins(string $subject, string $message): void
    {
        // Implement admin notification
    }
    
    private function logForDebug(WhmcsApiException $e, array $context): void
    {
        // Store detailed error info for debugging
    }
}
```

### Field-Level Validation Errors

```php
<?php
class ValidationErrorHandler {
    public function parseValidationErrors(array $response): array
    {
        $errors = [];
        
        if (isset($response['validation_errors'])) {
            foreach ($response['validation_errors'] as $field => $messages) {
                if (is_array($messages)) {
                    foreach ($messages as $message) {
                        $errors[] = new FieldError($field, $message);
                    }
                } else {
                    $errors[] = new FieldError($field, $messages);
                }
            }
        }
        
        return $errors;
    }
    
    public function formatForForm(array $errors): array
    {
        $formatted = [];
        
        foreach ($errors as $error) {
            if ($error instanceof FieldError) {
                $formatted[$error->field][] = $error->message;
            }
        }
        
        return $formatted;
    }
}

class FieldError {
    public function __construct(
        public readonly string $field,
        public readonly string $message
    ) {}
}
```

## Best Practices

1. **Always check response status** - Verify `result` field before processing
2. **Parse error codes** - Use codes to determine appropriate action
3. **Log all errors** - Include context for debugging
4. **Implement retry logic** - For network and rate limit errors
5. **Show user-friendly messages** - Don't expose raw error codes to users
6. **Monitor error patterns** - Track recurring errors for system health

## Related Documentation

- [WHMCS API Authentication](/docs/whmcs-api-authentication.md)
- [WHMCS API Rate Limiting](/docs/whmcs-api-rate-limiting.md)
- [WHMCS API Testing](/docs/whmcs-api-testing.md)