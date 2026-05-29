# WHMCS Module Error Handling

Complete reference for error handling in WHMCS modules.

## Overview

Proper error handling ensures modules fail gracefully and provide useful debugging information.

## Error Response Patterns

### Basic Error

```php
/**
 * Basic error return
 */
function yourmodule_CreateAccount(array $params)
{
    try {
        // Validate required parameters
        if (empty($params['domain'])) {
            return ['error' => 'Domain name is required'];
        }
        
        // Process account creation
        $result = $api->createAccount($params);
        
        return ['success' => true];
        
    } catch (Exception $e) {
        return ['error' => $e->getMessage()];
    }
}
```

### Detailed Error with Code

```php
/**
 * Error with error code
 */
function yourmodule_CreateAccount(array $params)
{
    try {
        // Validate parameters
        if (empty($params['domain'])) {
            return [
                'error' => 'Domain name is required',
                'errorCode' => 'MISSING_DOMAIN',
            ];
        }
        
        // Validate domain format
        if (!preg_match('/^[a-z0-9]+[a-z0-9\.-]+[a-z0-9]+$/i', $params['domain'])) {
            return [
                'error' => 'Invalid domain format',
                'errorCode' => 'INVALID_DOMAIN',
            ];
        }
        
        // Create account
        $result = $api->createAccount($params);
        
        return ['success' => true];
        
    } catch (ApiException $e) {
        return [
            'error' => $e->getMessage(),
            'errorCode' => $e->getCode(),
            'rawdata' => [
                'api_error_code' => $e->getApiCode(),
                'api_error_message' => $e->getApiMessage(),
            ],
        ];
    } catch (Exception $e) {
        return [
            'error' => 'An unexpected error occurred',
            'errorCode' => 'UNEXPECTED_ERROR',
        ];
    }
}
```

### Connection Error

```php
/**
 * Connection error handling
 */
function yourmodule_CreateAccount(array $params)
{
    try {
        $api = new YourModuleAPI($params);
        
        // Test connection first
        if (!$api->testConnection()) {
            return [
                'error' => 'Unable to connect to server. Please verify server settings.',
                'errorCode' => 'CONNECTION_FAILED',
            ];
        }
        
        // Create account
        $result = $api->createAccount($params);
        
        return ['success' => true];
        
    } catch (ConnectionException $e) {
        return [
            'error' => 'Server connection timeout. Please try again.',
            'errorCode' => 'CONNECTION_TIMEOUT',
        ];
    } catch (AuthenticationException $e) {
        return [
            'error' => 'Authentication failed. Please verify API credentials.',
            'errorCode' => 'AUTH_FAILED',
        ];
    }
}
```

### Validation Error

```php
/**
 * Validation error handling
 */
function yourmodule_CreateAccount(array $params)
{
    $errors = validateParams($params);
    
    if (!empty($errors)) {
        return [
            'error' => 'Validation failed: ' . implode(', ', $errors),
            'errorCode' => 'VALIDATION_ERROR',
            'errors' => $errors,
        ];
    }
    
    // Continue with account creation
}

function validateParams(array $params): array
{
    $errors = [];
    
    if (empty($params['domain'])) {
        $errors[] = 'Domain is required';
    }
    
    if (empty($params['username'])) {
        $errors[] = 'Username is required';
    }
    
    if (strlen($params['username']) < 3) {
        $errors[] = 'Username must be at least 3 characters';
    }
    
    if (empty($params['password'])) {
        $errors[] = 'Password is required';
    }
    
    if (strlen($params['password']) < 8) {
        $errors[] = 'Password must be at least 8 characters';
    }
    
    return $errors;
}
```

## Custom Exception Classes

### ModuleException

```php
<?php
/**
 * Base module exception
 */
class ModuleException extends Exception
{
    protected $errorCode;
    protected $errorData;
    
    public function __construct(string $message, string $errorCode = '', array $data = [])
    {
        parent::__construct($message);
        $this->errorCode = $errorCode;
        $this->errorData = $data;
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
        return [
            'error' => $this->getMessage(),
            'errorCode' => $this->errorCode,
            'rawdata' => $this->errorData,
        ];
    }
}

/**
 * API exception
 */
class ApiException extends ModuleException
{
    private $apiErrorCode;
    private $apiErrorMessage;
    
    public function __construct(
        string $message,
        string $apiErrorCode = '',
        string $apiErrorMessage = ''
    ) {
        parent::__construct($message, 'API_ERROR');
        $this->apiErrorCode = $apiErrorCode;
        $this->apiErrorMessage = $apiErrorMessage;
    }
    
    public function getApiCode(): string
    {
        return $this->apiErrorCode;
    }
    
    public function getApiMessage(): string
    {
        return $this->apiErrorMessage;
    }
}

/**
 * Connection exception
 */
class ConnectionException extends ModuleException
{
    public function __construct(string $message)
    {
        parent::__construct($message, 'CONNECTION_ERROR');
    }
}
```

### Using Custom Exceptions

```php
/**
 * Module function using custom exceptions
 */
function yourmodule_CreateAccount(array $params)
{
    try {
        if (empty($params['domain'])) {
            throw new ModuleException(
                'Domain name is required',
                'MISSING_DOMAIN'
            );
        }
        
        $api = new YourModuleAPI($params);
        
        if (!$api->isConnected()) {
            throw new ConnectionException(
                'Unable to connect to the server'
            );
        }
        
        $result = $api->createAccount($params);
        
        if (!$result['success']) {
            throw new ApiException(
                'Account creation failed',
                $result['error_code'] ?? '',
                $result['error_message'] ?? ''
            );
        }
        
        return ['success' => true];
        
    } catch (ModuleException $e) {
        return $e->toArray();
    } catch (Exception $e) {
        return [
            'error' => 'An unexpected error occurred: ' . $e->getMessage(),
            'errorCode' => 'UNEXPECTED_ERROR',
        ];
    }
}
```

## Logging Errors

### Module Call Logging

```php
/**
 * Log module calls for debugging
 */
function yourmodule_CreateAccount(array $params)
{
    $startTime = microtime(true);
    
    try {
        $api = new YourModuleAPI($params);
        
        $request = [
            'domain' => $params['domain'],
            'username' => $params['username'],
            'email' => $params['clientsdetails']['email'],
        ];
        
        $response = $api->createAccount($request);
        
        logModuleCall(
            'yourmodule',
            'CreateAccount',
            $request,
            $response,
            $response['success'] ? 'success' : 'error'
        );
        
        if ($response['success']) {
            return ['success' => true];
        }
        
        return [
            'error' => $response['message'] ?? 'Account creation failed',
            'rawdata' => $response,
        ];
        
    } catch (Exception $e) {
        logModuleCall(
            'yourmodule',
            'CreateAccount',
            $params,
            ['error' => $e->getMessage()],
            'error',
            '',
            $startTime,
            microtime(true)
        );
        
        return ['error' => $e->getMessage()];
    }
}
```

### Error Log File

```php
/**
 * Write to error log
 */
function logModuleError(string $function, string $message, array $context = []): void
{
    $logEntry = [
        'timestamp' => date('Y-m-d H:i:s'),
        'function' => $function,
        'message' => $message,
        'context' => $context,
    ];
    
    $logFile = __DIR__ . '/../logs/module_errors.log';
    
    $logDir = dirname($logFile);
    if (!is_dir($logDir)) {
        mkdir($logDir, 0755, true);
    }
    
    file_put_contents(
        $logFile,
        json_encode($logEntry) . "\n",
        FILE_APPEND
    );
}
```

## Error Recovery

### Retry Logic

```php
/**
 * Retry with exponential backoff
 */
function yourmodule_CreateAccount(array $params)
{
    $maxRetries = 3;
    $retryDelay = 1; // seconds
    
    for ($attempt = 1; $attempt <= $maxRetries; $attempt++) {
        try {
            $api = new YourModuleAPI($params);
            $result = $api->createAccount($params);
            
            return ['success' => true];
            
        } catch (RetryableException $e) {
            if ($attempt === $maxRetries) {
                return [
                    'error' => 'Account creation failed after ' . $maxRetries . ' attempts',
                    'errorCode' => 'MAX_RETRIES_EXCEEDED',
                ];
            }
            
            // Wait before retry with exponential backoff
            sleep($retryDelay * pow(2, $attempt - 1));
        }
    }
}
```

### Graceful Degradation

```php
/**
 * Graceful degradation
 */
function yourmodule_UsageUpdate(array $params)
{
    try {
        $api = new YourModuleAPI($params);
        $usage = $api->getUsage($params['username']);
        
        return [
            'success' => true,
            'diskusage' => $usage['disk'],
            'disklimit' => $usage['disk_limit'],
            'bwusage' => $usage['bw'],
            'bwlimit' => $usage['bw_limit'],
        ];
        
    } catch (Exception $e) {
        // Return zeros on failure instead of error
        return [
            'success' => true,
            'diskusage' => 0,
            'disklimit' => 0,
            'bwusage' => 0,
            'bwlimit' => 0,
            'rawdata' => [
                'error' => $e->getMessage(),
                'fallback' => true,
            ],
        ];
    }
}
```

## Best Practices

1. **Catch all exceptions** - Handle Exception and custom types
2. **Return arrays only** - Never throw from module functions
3. **Include error codes** - Provide codes for programmatic handling
4. **Log everything** - Use logModuleCall for debugging
5. **Provide context** - Include raw data in error returns
6. **Retry transient failures** - Implement retry for timeouts
7. **Fail gracefully** - Provide fallback values when possible

## Related Documentation

- [whmcs-module-return-values.md](whmcs-module-return-values.md)
- [whmcs-module-provisioning-api.md](whmcs-module-provisioning-api.md)