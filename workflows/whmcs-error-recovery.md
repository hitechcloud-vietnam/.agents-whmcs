# WHMCS Error Recovery Workflow

## Overview
This workflow implements comprehensive error recovery mechanisms for WHMCS operations.

## Prerequisites
- WHMCS installation with custom hooks
- PHP 7.4+ for exception handling
- Database access for error logging

## Step-by-Step Process

### Step 1: Create Error Handler Framework
```php
<?php
// /includes/error/ErrorRecoveryManager.php

namespace WHMCS\Error;

class ErrorRecoveryManager
{
    private $errorLog;
    private $recoveryActions = [];
    private $maxRetries = 3;

    public function __construct()
    {
        $this->errorLog = __DIR__ . '/../../logs/error_recovery.log';
    }

    /**
     * Register a recovery action
     */
    public function registerRecovery(string $errorType, callable $action, int $priority = 0)
    {
        $this->recoveryActions[$errorType][] = [
            'action' => $action,
            'priority' => $priority
        ];

        // Sort by priority
        usort($this->recoveryActions[$errorType], fn($a, $b) => $b['priority'] - $a['priority']);
    }

    /**
     * Execute with error recovery
     */
    public function executeWithRecovery(callable $operation, array $context = [])
    {
        $attempt = 0;
        $lastError = null;

        while ($attempt < $this->maxRetries) {
            try {
                return $operation();
            } catch (Exception $e) {
                $attempt++;
                $lastError = $e;

                $this->logError($e, $context, $attempt);

                // Attempt recovery
                $recovered = $this->attemptRecovery($e, $context);

                if (!$recovered && $attempt < $this->maxRetries) {
                    // Exponential backoff
                    sleep(pow(2, $attempt));
                }
            }
        }

        // All retries exhausted
        $this->handleExhaustedRetries($lastError, $context);

        throw $lastError;
    }

    /**
     * Attempt to recover from error
     */
    private function attemptRecovery(Exception $error, array $context): bool
    {
        $errorType = $this->classifyError($error);

        if (!isset($this->recoveryActions[$errorType])) {
            return false;
        }

        foreach ($this->recoveryActions[$errorType] as $recovery) {
            try {
                $result = $recovery['action']($error, $context);

                if ($result === true) {
                    logActivity("Error recovery successful: {$errorType}");
                    return true;
                }
            } catch (Exception $e) {
                $this->logError($e, ['recovery_attempt' => true], 0);
            }
        }

        return false;
    }

    /**
     * Classify error type
     */
    private function classifyError(Exception $error): string
    {
        $message = strtolower($error->getMessage());

        if (strpos($message, 'connection') !== false) {
            return 'connection_error';
        }

        if (strpos($message, 'timeout') !== false) {
            return 'timeout_error';
        }

        if (strpos($message, 'permission') !== false) {
            return 'permission_error';
        }

        if (strpos($message, 'duplicate') !== false) {
            return 'duplicate_error';
        }

        if (strpos($message, 'validation') !== false) {
            return 'validation_error';
        }

        return 'generic_error';
    }

    private function logError(Exception $error, array $context, int $attempt)
    {
        $logEntry = [
            'timestamp' => date('Y-m-d H:i:s'),
            'error' => $error->getMessage(),
            'class' => get_class($error),
            'file' => $error->getFile(),
            'line' => $error->getLine(),
            'attempt' => $attempt,
            'context' => $context,
            'trace' => $error->getTraceAsString()
        ];

        file_put_contents(
            $this->errorLog,
            json_encode($logEntry) . PHP_EOL,
            FILE_APPEND
        );
    }

    private function handleExhaustedRetries(Exception $error, array $context)
    {
        // Send alert
        sendAdminEmail('WHMCS Error Recovery Exhausted', [
            'error' => $error->getMessage(),
            'context' => $context,
            'attempts' => $this->maxRetries
        ]);

        // Log for manual intervention
        Capsule::table('mod_error_alerts')->insert([
            'error_type' => $this->classifyError($error),
            'error_message' => $error->getMessage(),
            'context' => json_encode($context),
            'created_at' => date('Y-m-d H:i:s'),
            'status' => 'pending'
        ]);
    }
}
```

### Step 2: Register Recovery Actions
```php
<?php
// /includes/hooks/error_recovery_hooks.php

use WHMCS\Error\ErrorRecoveryManager;

$errorManager = new ErrorRecoveryManager();

// Connection error recovery
$errorManager->registerRecovery('connection_error', function($error, $context) {
    // Wait and retry connection
    sleep(5);

    // Test connection
    if (testDatabaseConnection()) {
        return true;
    }

    // Try alternate server
    if (isset($context['fallback_server'])) {
        switchToServer($context['fallback_server']);
        return true;
    }

    return false;
}, 10);

// Timeout error recovery
$errorManager->registerRecovery('timeout_error', function($error, $context) {
    // Increase timeout and retry
    if (isset($context['operation'])) {
        $context['operation']['timeout'] = ($context['operation']['timeout'] ?? 30) * 2;
        return true;
    }

    return false;
}, 5);

// Duplicate error recovery
$errorManager->registerRecovery('duplicate_error', function($error, $context) {
    // Check if operation was actually successful
    if (isset($context['entity_id'])) {
        $exists = checkEntityExists($context['entity_type'], $context['entity_id']);
        return $exists; // Recovery successful if entity exists
    }

    return false;
}, 10);
```

### Step 3: Create Operation Wrapper
```php
<?php
// /includes/error/WrappedOperation.php

function wrapOperation(callable $operation, string $operationName, array $context = [])
{
    $errorManager = new ErrorRecoveryManager();

    return $errorManager->executeWithRecovery(function() use ($operation, $operationName, $context) {
        logActivity("Starting operation: {$operationName}");

        $result = $operation();

        logActivity("Completed operation: {$operationName}");

        return $result;
    }, array_merge(['operation_name' => $operationName], $context));
}
```

### Step 4: Implement Service Provisioning Recovery
```php
<?php
// Service provisioning with error recovery

add_hook('AfterServiceCreate', 1, function($vars) {
    $serviceId = $vars['serviceid'];
    $serverId = $vars['server'] ?? null;

    $errorManager = new ErrorRecoveryManager();

    // Register provisioning-specific recovery
    $errorManager->registerRecovery('generic_error', function($error, $context) use ($serviceId) {
        // Mark service for manual review
        Capsule::table('tblhosting')
            ->where('id', $serviceId)
            ->update(['domainstatus' => 'Pending']);

        // Create support ticket
        createSupportTicket([
            'subject' => 'Service Provisioning Failed',
            'service_id' => $serviceId,
            'error' => $error->getMessage()
        ]);

        return true;
    });

    try {
        $errorManager->executeWithRecovery(function() use ($serviceId, $serverId) {
            $result = provisionService($serviceId, $serverId);

            if (!$result['success']) {
                throw new Exception($result['error'] ?? 'Provisioning failed');
            }

            return $result;
        }, ['service_id' => $serviceId, 'server_id' => $serverId]);
    } catch (Exception $e) {
        logActivity("Service provisioning failed after retries: " . $e->getMessage());

        sendAdminEmail('Provisioning Failed', [
            'service_id' => $serviceId,
            'error' => $e->getMessage()
        ]);
    }
});
```

### Step 5: Create Database Transaction Recovery
```php
<?php
// Database transaction with automatic recovery

class TransactionRecovery
{
    public static function execute(callable $operations, array $options = [])
    {
        $options = array_merge([
            'retries' => 3,
            'isolation_level' => 'REPEATABLE READ'
        ], $options);

        $attempt = 0;

        while ($attempt < $options['retries']) {
            $attempt++;

            try {
                Capsule::connection()->transaction(function() use ($operations) {
                    $operations();
                });

                return ['success' => true];
            } catch (QueryException $e) {
                // Check if it's a deadlock or lock timeout
                if (self::isRecoverableError($e)) {
                    logActivity("Transaction deadlock, retrying (attempt {$attempt})");

                    // Exponential backoff
                    usleep(pow(2, $attempt) * 100000);
                    continue;
                }

                throw $e;
            }
        }

        throw new Exception("Transaction failed after {$options['retries']} retries");
    }

    private static function isRecoverableError(QueryException $e): bool
    {
        $message = strtolower($e->getMessage());

        return strpos($message, 'deadlock') !== false
            || strpos($message, 'lock wait timeout') !== false
            || strpos($message, 'try restarting transaction') !== false;
    }
}

// Usage
TransactionRecovery::execute(function() {
    // Create invoice
    $invoiceId = createInvoice($data);

    // Update service
    updateService($serviceId, ['status' => 'Active']);

    // Add transaction log
    logTransaction($invoiceId, $serviceId);
});
```

### Step 6: Create API Call Recovery
```php
<?php
// API call with retry logic

class ApiCallRecovery
{
    private $maxRetries = 3;
    private $baseDelay = 1;

    public function call(string $url, array $options = []): array
    {
        $attempt = 0;

        while ($attempt < $this->maxRetries) {
            $attempt++;

            try {
                return $this->makeRequest($url, $options);
            } catch (ApiException $e) {
                if ($e->isRetryable() && $attempt < $this->maxRetries) {
                    $delay = $this->baseDelay * pow(2, $attempt - 1);
                    sleep($delay);

                    logActivity("API call retry: {$url} (attempt {$attempt})");
                    continue;
                }

                throw $e;
            }
        }

        throw new Exception("API call failed after {$this->maxRetries} attempts");
    }

    private function makeRequest(string $url, array $options): array
    {
        $ch = curl_init($url);

        curl_setopt_array($ch, [
            CURLOPT_RETURNTRANSFER => true,
            CURLOPT_TIMEOUT => $options['timeout'] ?? 30,
            CURLOPT_HTTPHEADER => $options['headers'] ?? []
        ]);

        if (isset($options['method'])) {
            curl_setopt($ch, CURLOPT_CUSTOMREQUEST, $options['method']);
        }

        if (isset($options['body'])) {
            curl_setopt($ch, CURLOPT_POSTFIELDS, json_encode($options['body']));
        }

        $response = curl_exec($ch);
        $httpCode = curl_getinfo($ch, CURLINFO_HTTP_CODE);
        $error = curl_error($ch);
        curl_close($ch);

        if ($error) {
            throw new ApiException($error, 0, true);
        }

        if ($httpCode >= 500) {
            throw new ApiException("Server error: {$httpCode}", $httpCode, true);
        }

        if ($httpCode >= 400) {
            throw new ApiException("Client error: {$httpCode}", $httpCode, false);
        }

        return json_decode($response, true);
    }
}
```

### Step 7: Error Monitoring and Alerts
```php
<?php
// Error monitoring setup

add_hook('DailyCronJob', 1, function($vars) {
    // Check for recent errors
    $recentErrors = Capsule::table('mod_error_alerts')
        ->where('status', 'pending')
        ->where('created_at', '>', date('Y-m-d H:i:s', strtotime('-24 hours')))
        ->count();

    if ($recentErrors > 10) {
        sendAdminEmail('High Error Rate Detected', [
            'error_count' => $recentErrors,
            'period' => 'last 24 hours'
        ]);
    }

    // Clean up old resolved errors
    Capsule::table('mod_error_alerts')
        ->where('status', 'resolved')
        ->where('created_at', '<', date('Y-m-d', strtotime('-30 days')))
        ->delete();
});
```

## Error Recovery Strategies

| Error Type | Recovery Strategy |
|------------|-------------------|
| Connection | Retry with backoff, failover |
| Timeout | Increase timeout, retry |
| Deadlock | Transaction retry |
| Permission | Escalate to admin |
| Duplicate | Check state, continue |
| Validation | Fix data, retry |
| Rate Limit | Backoff, queue request |

## Best Practices

1. **Classify errors** - Different recovery for different types
2. **Retry with backoff** - Prevent thundering herd
3. **Log everything** - For debugging and analysis
4. **Alert on exhaustion** - Notify admin when retries fail
5. **Circuit breaker** - Stop calling failing services
6. **Idempotency** - Make operations safe to retry

## Related Workflows
- [WHMCS Retry Logic](./whmcs-retry-logic.md)
- [WHMCS Transaction Management](./whmcs-transaction-management.md)
- [WHMCS Rollback Automation](./whmcs-rollback-automation.md)