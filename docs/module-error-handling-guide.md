# WHMCS Module Error Handling Guide

**Version:** 8.0 | **Updated:** 2026-05-28

## Overview

This guide covers error handling patterns for WHMCS modules, including exception handling, user-facing error messages, logging strategies, and graceful degradation.

---

## Error Categories

### Error Types

| Type | Description | Examples |
|------|-------------|----------|
| Validation Errors | User input validation | Invalid email, missing required fields |
| Configuration Errors | Module misconfiguration | Missing API key, invalid credentials |
| API Errors | External service issues | Timeout, rate limit, server error |
| Database Errors | Data operations | Query failure, constraint violation |
| Permission Errors | Access control | Unauthorized, permission denied |
| System Errors | Environment issues | File not found, memory limit |

## Exception Classes

### Custom Exception Hierarchy

```php
<?php
/**
 * Module Exception Classes
 */

namespace YourModule\Exceptions;

class ModuleException extends \Exception
{
    protected $errorCode;
    protected $details;
    
    public function __construct($message, $errorCode = 'UNKNOWN', \Exception $previous = null)
    {
        parent::__construct($message, 0, $previous);
        $this->errorCode = $errorCode;
    }
    
    public function getErrorCode()
    {
        return $this->errorCode;
    }
}

class ValidationException extends ModuleException
{
    protected $field;
    protected $value;
    
    public function __construct($message, $field, $value = null)
    {
        parent::__construct($message, 'VALIDATION_ERROR');
        $this->field = $field;
        $this->value = $value;
    }
    
    public function getField()
    {
        return $this->field;
    }
    
    public function getValue()
    {
        return $this->value;
    }
}

class ApiException extends ModuleException
{
    protected $httpCode;
    protected $response;
    
    public function __construct($message, $httpCode = 500, $response = [])
    {
        parent::__construct($message, 'API_ERROR');
        $this->httpCode = $httpCode;
        $this->response = $response;
    }
    
    public function getHttpCode()
    {
        return $this->httpCode;
    }
    
    public function getResponse()
    {
        return $this->response;
    }
}

class ConfigurationException extends ModuleException
{
    public function __construct($message, $setting = null)
    {
        parent::__construct($message, 'CONFIG_ERROR');
        $this->details = ['setting' => $setting];
    }
}

class PermissionException extends ModuleException
{
    public function __construct($message, $requiredPermission = null)
    {
        parent::__construct($message, 'PERMISSION_DENIED');
        $this->details = ['required_permission' => $requiredPermission];
    }
}
```

## Error Handling Patterns

### Try-Catch Pattern

```php
/**
 * Handle errors with try-catch
 */
function your_module_safeOperation($clientId)
{
    try {
        // Validate input
        if (!is_numeric($clientId) || $clientId <= 0) {
            throw new ValidationException('Invalid client ID', 'client_id', $clientId);
        }
        
        // Attempt operation
        $result = your_module_fetchData($clientId);
        
        if ($result === false) {
            throw new ModuleException('Failed to fetch data', 'FETCH_ERROR');
        }
        
        return [
            'success' => true,
            'data'    => $result,
        ];
        
    } catch (ValidationException $e) {
        return [
            'success'  => false,
            'error'    => $e->getMessage(),
            'error_code' => $e->getErrorCode(),
            'field'    => $e->getField(),
        ];
        
    } catch (ApiException $e) {
        logModuleCall('your_module', 'api_error', [
            'client_id' => $clientId,
        ], $e->getMessage(), $e->getResponse());
        
        return [
            'success'   => false,
            'error'     => 'External service error. Please try again later.',
            'error_code' => $e->getErrorCode(),
        ];
        
    } catch (ModuleException $e) {
        logModuleCall('your_module', 'operation_error', [
            'client_id' => $clientId,
        ], $e->getMessage());
        
        return [
            'success'   => false,
            'error'     => $e->getMessage(),
            'error_code' => $e->getErrorCode(),
        ];
        
    } catch (\Exception $e) {
        // Unexpected error - log and hide details from user
        logModuleCall('your_module', 'unexpected_error', [
            'client_id' => $clientId,
        ], $e->getMessage(), $e->getTraceAsString());
        
        return [
            'success'   => false,
            'error'     => 'An unexpected error occurred.',
            'error_code' => 'INTERNAL_ERROR',
        ];
    }
}
```

### API Error Handling

```php
/**
 * Handle API errors with retry logic
 */
function your_module_apiCallWithRetry($endpoint, $data, $maxRetries = 3)
{
    $attempt = 0;
    $lastError = null;
    
    while ($attempt < $maxRetries) {
        try {
            $attempt++;
            
            $response = your_module_apiCall($endpoint, $data);
            
            // Check for API-level errors
            if (isset($response['error'])) {
                throw new ApiException(
                    $response['error']['message'] ?? 'API Error',
                    $response['error']['code'] ?? 500,
                    $response
                );
            }
            
            return $response;
            
        } catch (ApiException $e) {
            $lastError = $e;
            
            // Don't retry client errors (4xx)
            if ($e->getHttpCode() >= 400 && $e->getHttpCode() < 500) {
                throw $e;
            }
            
            // Exponential backoff
            if ($attempt < $maxRetries) {
                $delay = pow(2, $attempt) * 100000; // microseconds
                usleep($delay);
            }
        }
    }
    
    throw $lastError;
}
```

### Database Error Handling

```php
/**
 * Handle database errors
 */
function your_module_safeDatabaseOperation($clientId, $data)
{
    try {
        $result = WHMCS\Database\Capsule::table('mod_your_table')
            ->where('client_id', $clientId)
            ->update($data);
        
        if ($result === 0) {
            // Check if record exists
            $exists = WHMCS\Database\Capsule::table('mod_your_table')
                ->where('client_id', $clientId)
                ->exists();
            
            if (!$exists) {
                throw new ModuleException('Record not found', 'NOT_FOUND');
            }
        }
        
        return [
            'success' => true,
            'updated' => $result,
        ];
        
    } catch (\Illuminate\Database\QueryException $e) {
        // Handle specific database errors
        $errorCode = $e->getCode();
        
        switch ($errorCode) {
            case 23000: // Integrity constraint violation
                return [
                    'success'   => false,
                    'error'     => 'Duplicate entry or constraint violation',
                    'error_code' => 'CONSTRAINT_ERROR',
                ];
                
            case 1452: // Foreign key constraint
                return [
                    'success'   => false,
                    'error'     => 'Referenced record does not exist',
                    'error_code' => 'FOREIGN_KEY_ERROR',
                ];
                
            default:
                logModuleCall('your_module', 'database_error', [
                    'client_id' => $clientId,
                    'data'     => $data,
                ], $e->getMessage());
                
                return [
                    'success'   => false,
                    'error'     => 'Database operation failed',
                    'error_code' => 'DATABASE_ERROR',
                ];
        }
    }
}
```

## Form Validation Errors

```php
/**
 * Validate form input with detailed errors
 */
function your_module_validateConfiguration($data)
{
    $errors = [];
    
    // Required fields
    $required = ['api_key', 'api_secret'];
    
    foreach ($required as $field) {
        if (empty($data[$field])) {
            $errors[$field] = [
                'error' => 'This field is required',
                'code'  => 'REQUIRED',
            ];
        }
    }
    
    // API key format
    if (isset($data['api_key']) && !preg_match('/^[a-zA-Z0-9]{32,}$/', $data['api_key'])) {
        $errors['api_key'] = [
            'error' => 'Invalid API key format',
            'code'  => 'INVALID_FORMAT',
        ];
    }
    
    // URL validation
    if (!empty($data['webhook_url']) && !filter_var($data['webhook_url'], FILTER_VALIDATE_URL)) {
        $errors['webhook_url'] = [
            'error' => 'Invalid URL format',
            'code'  => 'INVALID_URL',
        ];
    }
    
    // Email validation
    if (!empty($data['notification_email']) && 
        !filter_var($data['notification_email'], FILTER_VALIDATE_EMAIL)) {
        $errors['notification_email'] = [
            'error' => 'Invalid email address',
            'code'  => 'INVALID_EMAIL',
        ];
    }
    
    return [
        'valid'   => empty($errors),
        'errors'  => $errors,
    ];
}

/**
 * Format validation errors for display
 */
function your_module_formatValidationErrors($errors)
{
    $formatted = [];
    
    foreach ($errors as $field => $error) {
        $formatted[] = [
            'field' => $field,
            'msg'   => $error['error'],
        ];
    }
    
    return $formatted;
}
```

## User-Facing Error Messages

### Error Message Mapping

```php
/**
 * User-friendly error messages
 */
function your_module_getUserMessage($errorCode, $lang = [])
{
    $messages = [
        'VALIDATION_ERROR' => [
            'default' => 'Please check your input and try again.',
            'required_field' => 'Please fill in all required fields.',
            'invalid_email'  => 'Please enter a valid email address.',
            'invalid_amount' => 'Please enter a valid amount.',
        ],
        
        'API_ERROR' => [
            'default'        => 'Unable to connect to the service. Please try again.',
            'timeout'        => 'The request timed out. Please try again.',
            'rate_limit'     => 'Too many requests. Please wait a moment and try again.',
            'server_error'   => 'Service temporarily unavailable. Please try again later.',
            'auth_failed'    => 'Authentication failed. Please check your API credentials.',
        ],
        
        'CONFIG_ERROR' => [
            'default'     => 'Configuration error. Please check module settings.',
            'missing_key' => 'API key is not configured. Please update module settings.',
            'invalid_key' => 'API key is invalid. Please check your settings.',
        ],
        
        'PERMISSION_DENIED' => [
            'default'         => 'You do not have permission to perform this action.',
            'admin_required' => 'Administrator privileges required.',
            'owner_required' => 'You must be the record owner.',
        ],
        
        'NOT_FOUND' => [
            'default'  => 'The requested record was not found.',
            'client'   => 'Client not found.',
            'invoice'  => 'Invoice not found.',
        ],
    ];
    
    $category = explode('_', $errorCode)[0];
    $subCode = strtolower(explode('_', $errorCode)[1] ?? 'default');
    
    return $messages[$category][$subCode] ?? $messages[$category]['default'] ?? 
           'An error occurred. Please try again.';
}
```

## WHMCS Alert Integration

```php
/**
 * Show WHMCS error alert
 */
function your_module_showError($message, $title = 'Error')
{
    return '<div class="alert alert-danger">
        <strong>' . htmlspecialchars($title) . ':</strong> 
        ' . htmlspecialchars($message) . '
    </div>';
}

/**
 * Show WHMCS success alert
 */
function your_module_showSuccess($message, $title = 'Success')
{
    return '<div class="alert alert-success">
        <strong>' . htmlspecialchars($title) . ':</strong> 
        ' . htmlspecialchars($message) . '
    </div>';
}

/**
 * Show WHMCS warning alert
 */
function your_module_showWarning($message, $title = 'Warning')
{
    return '<div class="alert alert-warning">
        <strong>' . htmlspecialchars($title) . ':</strong> 
        ' . htmlspecialchars($message) . '
    </div>';
}

/**
 * Redirect with error message
 */
function your_module_redirectWithError($url, $errorMessage)
{
    redir([
        'success' => false,
        'message' => $errorMessage,
    ], $url);
}

/**
 * Redirect with success message
 */
function your_module_redirectWithSuccess($url, $successMessage)
{
    redir([
        'success' => true,
        'message' => $successMessage,
    ], $url);
}
```

## Graceful Degradation

```php
/**
 * Implement graceful degradation
 */
function your_module_getDataWithFallback($clientId)
{
    try {
        // Try primary source
        $data = your_module_fetchFromCache($clientId);
        
        if ($data === null) {
            $data = your_module_fetchFromApi($clientId);
            
            // Cache for next time
            your_module_storeInCache($clientId, $data);
        }
        
        return $data;
        
    } catch (ApiException $e) {
        // API failed, try cache
        $cached = your_module_fetchFromCache($clientId);
        
        if ($cached !== null) {
            logModuleCall('your_module', 'fallback_cache', [
                'client_id' => $clientId,
                'reason'    => $e->getMessage(),
            ]);
            
            return array_merge($cached, [
                '_cached'  => true,
                '_warning' => 'Data may be outdated',
            ]);
        }
        
        throw $e;
    }
}
```

## Error Logging

```php
/**
 * Log errors with context
 */
function your_module_logError($error, $context = [])
{
    // Mask sensitive data
    $safeContext = your_module_maskSensitiveData($context);
    
    logModuleCall(
        'your_module',
        'error',
        [
            'timestamp' => date('Y-m-d H:i:s'),
            'context'   => $safeContext,
        ],
        $error instanceof \Exception ? $error->getMessage() : $error,
        $error instanceof \Exception ? $error->getTraceAsString() : null
    );
    
    // Also log to custom log file
    $logFile = ini_get('error_log') ?? '/tmp/module_errors.log';
    $entry = date('Y-m-d H:i:s') . ' [your_module] ' . print_r($safeContext, true) . "\n";
    error_log($entry, 3, $logFile);
}

/**
 * Mask sensitive data
 */
function your_module_maskSensitiveData($data)
{
    $sensitiveKeys = ['password', 'api_key', 'api_secret', 'token', 'secret'];
    
    foreach ($data as $key => $value) {
        $lowerKey = strtolower($key);
        
        foreach ($sensitiveKeys as $sensitiveKey) {
            if (strpos($lowerKey, strtolower($sensitiveKey)) !== false) {
                $data[$key] = '***HIDDEN***';
                break;
            }
        }
    }
    
    return $data;
}
```

---

## Related Skills and Workflows

- `module-logging-guide` - Logging strategies
- `module-security-standards` - Security error handling
- `error-codes-reference` - WHMCS error codes
- `module-performance-best-practices` - Graceful degradation
