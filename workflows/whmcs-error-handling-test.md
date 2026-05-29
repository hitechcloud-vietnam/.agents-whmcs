# WHMCS Error Handling Test Workflow

## Overview
This workflow provides comprehensive guidance for testing error handling in WHMCS modules and customizations.

## Prerequisites
- WHMCS installation (v8.0+)
- Error tracking tools
- Logging configuration

## Step-by-Step Guide

### Step 1: Configure Error Handling
```php
// modules/addons/yourmodule/bootstrap.php
<?php
// Set error reporting for development
error_reporting(E_ALL);
ini_set('display_errors', 1);

// Custom error handler
set_error_handler(function ($severity, $message, $file, $line) {
    if (!(error_reporting() & $severity)) {
        return false;
    }
    throw new ErrorException($message, 0, $severity, $file, $line);
});

// Exception handler
set_exception_handler(function ($exception) {
    logError($exception->getMessage(), $exception->getTraceAsString());
    
    if (defined('WHMCS_DEBUG')) {
        echo '<pre>' . $exception . '</pre>';
    } else {
        echo 'An error occurred. Please contact support.';
    }
});

// Fatal error handler
register_shutdown_function(function () {
    $error = error_get_last();
    if ($error !== null && in_array($error['type'], [E_ERROR, E_CORE_ERROR, E_COMPILE_ERROR])) {
        logError($error['message'], "File: {$error['file']}\nLine: {$error['line']}");
    }
});
```

### Step 2: Write Error Handling Tests
```php
// tests/ErrorHandlingTest.php
<?php
namespace WHMCS\Tests;

use PHPUnit\Framework\TestCase;

class ErrorHandlingTest extends TestCase
{
    public function testDatabaseConnectionError()
    {
        $this->expectException(\PDOException::class);
        
        // Try to connect with invalid credentials
        $pdo = new PDO(
            'mysql:host=localhost;dbname=nonexistent',
            'invalid_user',
            'invalid_pass'
        );
    }

    public function testInvalidInputValidation()
    {
        $handler = new \WHMCS\Module\YourModule\InputValidator();
        
        $this->expectException(\InvalidArgumentException::class);
        $this->expectExceptionMessage('Invalid email format');
        
        $handler->validateEmail('not-an-email');
    }

    public function testMissingRequiredField()
    {
        $handler = new \WHMCS\Module\YourModule\InputValidator();
        
        $this->expectException(\InvalidArgumentException::class);
        
        $handler->validateRequired([
            'name' => '',
            'email' => 'test@example.com',
        ], ['name', 'email']);
    }

    public function testFileNotFoundError()
    {
        $this->expectException(\RuntimeException::class);
        
        $handler = new \WHMCS\Module\YourModule\FileHandler();
        $handler->readFile('/nonexistent/file.txt');
    }

    public function testApiTimeoutError()
    {
        $client = new \GuzzleHttp\Client([
            'timeout' => 0.001, // Very short timeout
        ]);
        
        $this->expectException(\GuzzleHttp\Exception\ConnectException::class);
        
        $client->get('http://slow-server.example.com/slow-endpoint');
    }

    public function testInvalidJsonResponse()
    {
        $handler = new \WHMCS\Module\YourModule\ApiClient();
        
        $this->expectException(\RuntimeException::class);
        $this->expectExceptionMessage('Invalid JSON response');
        
        $handler->parseResponse('not valid json {');
    }

    public function testRateLimitExceeded()
    {
        $handler = new \WHMCS\Module\YourModule\ApiClient();
        
        $this->expectException(\RuntimeException::class);
        $this->expectExceptionMessage('Rate limit exceeded');
        
        // Make many rapid requests
        for ($i = 0; $i < 100; $i++) {
            try {
                $handler->makeRequest('test');
            } catch (\RuntimeException $e) {
                if (strpos($e->getMessage(), 'Rate limit') !== false) {
                    throw $e;
                }
            }
        }
    }

    public function testPermissionDeniedError()
    {
        $this->expectException(\RuntimeException::class);
        $this->expectExceptionMessage('Permission denied');
        
        $handler = new \WHMCS\Module\YourModule\FileHandler();
        $handler->writeFile('/root/protected.txt', 'content');
    }

    public function testOutOfMemoryError()
    {
        // Skip if memory limit is unlimited
        if (ini_get('memory_limit') === '-1') {
            $this->markTestSkipped('Memory limit is unlimited');
        }
        
        $this->expectException(\RuntimeException::class);
        
        $handler = new \WHMCS\Module\YourModule\DataProcessor();
        $handler->processLargeDataset(str_repeat('x', PHP_INT_MAX));
    }

    public function testAuthenticationFailure()
    {
        $handler = new \WHMCS\Module\YourModule\AuthService();
        
        $this->expectException(\RuntimeException::class);
        $this->expectExceptionMessage('Authentication failed');
        
        $handler->authenticate('invalid_user', 'wrong_password');
    }

    public function testValidationErrorWithDetails()
    {
        $validator = new \WHMCS\Module\YourModule\Validator();
        
        $result = $validator->validate([
            'email' => 'invalid',
            'phone' => '123',
            'age' => -5,
        ]);
        
        $this->assertFalse($result['valid']);
        $this->assertArrayHasKey('errors', $result);
        $this->assertGreaterThan(0, count($result['errors']));
    }

    public function testGracefulDegradation()
    {
        $service = new \WHMCS\Module\YourModule\ServiceWithFallback();
        
        // Primary service fails
        $service->setPrimaryFailing(true);
        
        // Should fall back to secondary
        $result = $service->fetch();
        
        $this->assertTrue($result['using_fallback']);
        $this->assertArrayHasKey('data', $result);
    }

    public function testCircuitBreakerOpens()
    {
        $breaker = new \WHMCS\Module\YourModule\CircuitBreaker(3, 60);
        
        // Fail multiple times
        for ($i = 0; $i < 5; $i++) {
            try {
                $breaker->call(function() {
                    throw new \Exception('Service down');
                });
            } catch (\Exception $e) {
                $breaker->recordFailure();
            }
        }
        
        // Circuit should be open
        $this->assertTrue($breaker->isOpen());
    }

    public function testRetryWithBackoff()
    {
        $attempts = 0;
        
        $handler = new \WHMCS\Module\YourModule\RetryHandler();
        
        $result = $handler->executeWithRetry(function() use (&$attempts) {
            $attempts++;
            if ($attempts < 3) {
                throw new \Exception('Temporary failure');
            }
            return 'success';
        }, 3, 100); // 3 retries, 100ms initial delay
        
        $this->assertEquals('success', $result);
        $this->assertEquals(3, $attempts);
    }
}
```

### Step 3: Write Custom Exception Tests
```php
// tests/CustomExceptionsTest.php
<?php
namespace WHMCS\Tests;

use PHPUnit\Framework\TestCase;

class CustomExceptionsTest extends TestCase
{
    public function testModuleExceptionContainsContext()
    {
        try {
            throw new \WHMCS\Module\YourModule\Exceptions\ModuleException(
                'Configuration error',
                1001,
                ['setting' => 'api_key', 'value' => 'missing']
            );
        } catch (\WHMCS\Module\YourModule\Exceptions\ModuleException $e) {
            $this->assertEquals(1001, $e->getErrorCode());
            $this->assertArrayHasKey('setting', $e->getContext());
            $this->assertEquals('api_key', $e->getContext()['setting']);
        }
    }

    public function testApiExceptionIncludesResponse()
    {
        $response = new \GuzzleHttp\Psr7\Response(400, [], json_encode([
            'error' => 'Invalid request',
            'details' => ['field' => 'email'],
        ]));
        
        try {
            throw new \WHMCS\Module\YourModule\Exceptions\ApiException(
                'API request failed',
                400,
                $response
            );
        } catch (\WHMCS\Module\YourModule\Exceptions\ApiException $e) {
            $this->assertEquals(400, $e->getCode());
            $this->assertEquals($response, $e->getResponse());
            $this->assertEquals('Invalid request', $e->getApiError());
        }
    }

    public function testValidationExceptionHasFieldErrors()
    {
        $errors = [
            'email' => 'Invalid email format',
            'password' => 'Password too short',
        ];
        
        try {
            throw new \WHMCS\Module\YourModule\Exceptions\ValidationException($errors);
        } catch (\WHMCS\Module\YourModule\Exceptions\ValidationException $e) {
            $this->assertEquals($errors, $e->getFieldErrors());
            $this->assertTrue($e->hasFieldError('email'));
            $this->assertFalse($e->hasFieldError('name'));
        }
    }

    public function testRetryableException()
    {
        $exception = new \WHMCS\Module\YourModule\Exceptions\RetryableException('Temporary error');
        
        $this->assertTrue($exception->isRetryable());
        $this->assertEquals(0, $exception->getRetryCount());
        
        $exception->incrementRetryCount();
        $this->assertEquals(1, $exception->getRetryCount());
    }
}
```

### Step 4: Run Error Handling Tests
```bash
# Run all error handling tests
./vendor/bin/phpunit tests/ErrorHandlingTest.php

# Run with verbose output
./vendor/bin/phpunit tests/ErrorHandlingTest.php --testdox

# Generate coverage report
./vendor/bin/phpunit tests/ErrorHandlingTest.php --coverage-html coverage/errors/
```

## Error Handling Checklist

### Exception Types
- [ ] Database exceptions handled
- [ ] API exceptions handled
- [ ] Validation exceptions handled
- [ ] File system exceptions handled
- [ ] Network exceptions handled

### Error Recovery
- [ ] Retry mechanisms implemented
- [ ] Circuit breakers configured
- [ ] Fallback services available
- [ ] Graceful degradation works

### Logging
- [ ] Errors logged with context
- [ ] Stack traces captured
- [ ] Sensitive data redacted
- [ ] Logs rotated properly
