# WHMCS Retry Logic Workflow

## Overview
This workflow implements intelligent retry mechanisms for failed operations in WHMCS.

## Prerequisites
- WHMCS with custom hooks capability
- PHP 7.4+ for modern syntax
- Understanding of retry patterns

## Step-by-Step Process

### Step 1: Create Retry Manager Class
```php
<?php
// /includes/retry/RetryManager.php

namespace WHMCS\Retry;

class RetryManager
{
    private $maxAttempts;
    private $baseDelay;
    private $maxDelay;
    private $multiplier;
    private $jitter;

    public function __construct(array $config = [])
    {
        $this->maxAttempts = $config['max_attempts'] ?? 3;
        $this->baseDelay = $config['base_delay'] ?? 1000; // milliseconds
        $this->maxDelay = $config['max_delay'] ?? 30000;
        $this->multiplier = $config['multiplier'] ?? 2;
        $this->jitter = $config['jitter'] ?? true;
    }

    /**
     * Execute operation with retry
     */
    public function execute(callable $operation, array $options = [])
    {
        $attempts = 0;
        $lastException = null;

        while ($attempts < $this->maxAttempts) {
            $attempts++;

            try {
                return $operation();
            } catch (Exception $e) {
                $lastException = $e;

                // Check if we should retry
                if (!$this->shouldRetry($e, $options)) {
                    throw $e;
                }

                // Check if we have attempts remaining
                if ($attempts < $this->maxAttempts) {
                    $delay = $this->calculateDelay($attempts, $options);
                    $this->logRetry($e, $attempts, $delay);
                    usleep($delay * 1000); // Convert to microseconds
                }
            }
        }

        // All retries exhausted
        $this->logExhausted($lastException, $attempts);
        throw $lastException;
    }

    /**
     * Check if exception is retryable
     */
    private function shouldRetry(Exception $e, array $options): bool
    {
        // Check for explicit non-retryable
        if ($e instanceof NonRetryableException) {
            return false;
        }

        // Check exception type
        $retryableTypes = $options['retryable_types'] ?? [
            'ConnectionException',
            'TimeoutException',
            'ServerException',
            'TemporaryException'
        ];

        foreach ($retryableTypes as $type) {
            if ($e instanceof $type) {
                return true;
            }
        }

        // Check error message patterns
        $retryablePatterns = $options['retryable_patterns'] ?? [
            '/connection refused/i',
            '/timeout/i',
            '/temporarily unavailable/i',
            '/service unavailable/i',
            '/too many requests/i',
            '/rate limit/i',
            '/deadlock/i',
            '/lock wait/i'
        ];

        $message = $e->getMessage();

        foreach ($retryablePatterns as $pattern) {
            if (preg_match($pattern, $message)) {
                return true;
            }
        }

        return false;
    }

    /**
     * Calculate delay with exponential backoff
     */
    private function calculateDelay(int $attempt, array $options): int
    {
        $baseDelay = $options['base_delay'] ?? $this->baseDelay;
        $multiplier = $options['multiplier'] ?? $this->multiplier;
        $maxDelay = $options['max_delay'] ?? $this->maxDelay;

        // Exponential backoff
        $delay = $baseDelay * pow($multiplier, $attempt - 1);

        // Add jitter to prevent thundering herd
        if ($this->jitter) {
            $jitterAmount = rand(0, $delay * 0.3);
            $delay += $jitterAmount;
        }

        return min($delay, $maxDelay);
    }

    private function logRetry(Exception $e, int $attempt, int $delay)
    {
        logActivity(sprintf(
            "Retry attempt %d/%d for error: %s (delay: %dms)",
            $attempt,
            $this->maxAttempts,
            $e->getMessage(),
            $delay
        ));
    }

    private function logExhausted(Exception $e, int $attempts)
    {
        logActivity(sprintf(
            "All %d retry attempts exhausted for error: %s",
            $attempts,
            $e->getMessage()
        ));
    }
}

// Custom exception classes
class NonRetryableException extends Exception {}
class TemporaryException extends Exception {}
class PermanentException extends Exception {}
```

### Step 2: Create Decorator Pattern Retry
```php
<?php
// /includes/retry/RetryDecorator.php

namespace WHMCS\Retry;

class RetryDecorator
{
    private $retryManager;
    private $target;

    public function __construct(callable $target, RetryManager $retryManager = null)
    {
        $this->target = $target;
        $this->retryManager = $retryManager ?? new RetryManager();
    }

    /**
     * Execute with retry
     */
    public function __invoke(...$args)
    {
        return $this->retryManager->execute(function() {
            return ($this->target)(...func_get_args());
        });
    }

    /**
     * Execute static method with retry
     */
    public static function decorate(string $class, string $method, RetryManager $retryManager = null)
    {
        $retryManager = $retryManager ?? new RetryManager();

        return function(...$args) use ($class, $method, $retryManager) {
            return $retryManager->execute(function() use ($class, $method, $args) {
                return call_user_func_array([$class, $method], $args);
            });
        };
    }
}
```

### Step 3: Implement HTTP Request Retry
```php
<?php
// /includes/retry/RetryableHttpClient.php

namespace WHMCS\Retry;

class RetryableHttpClient
{
    private $retryManager;

    public function __construct(RetryManager $retryManager = null)
    {
        $this->retryManager = $retryManager ?? new RetryManager([
            'max_attempts' => 5,
            'base_delay' => 500,
            'multiplier' => 2
        ]);
    }

    public function get(string $url, array $options = []): array
    {
        return $this->request('GET', $url, null, $options);
    }

    public function post(string $url, array $data, array $options = []): array
    {
        return $this->request('POST', $url, $data, $options);
    }

    public function put(string $url, array $data, array $options = []): array
    {
        return $this->request('PUT', $url, $data, $options);
    }

    public function delete(string $url, array $options = []): array
    {
        return $this->request('DELETE', $url, null, $options);
    }

    private function request(string $method, string $url, ?array $data, array $options): array
    {
        return $this->retryManager->execute(function() use ($method, $url, $data, $options) {
            $ch = curl_init();

            curl_setopt_array($ch, [
                CURLOPT_URL => $url,
                CURLOPT_RETURNTRANSFER => true,
                CURLOPT_TIMEOUT => $options['timeout'] ?? 30,
                CURLOPT_CONNECTTIMEOUT => $options['connect_timeout'] ?? 10
            ]);

            switch (strtoupper($method)) {
                case 'POST':
                    curl_setopt($ch, CURLOPT_POST, true);
                    curl_setopt($ch, CURLOPT_POSTFIELDS, json_encode($data));
                    break;
                case 'PUT':
                    curl_setopt($ch, CURLOPT_CUSTOMREQUEST, 'PUT');
                    curl_setopt($ch, CURLOPT_POSTFIELDS, json_encode($data));
                    break;
                case 'DELETE':
                    curl_setopt($ch, CURLOPT_CUSTOMREQUEST, 'DELETE');
                    break;
            }

            $headers = ['Content-Type: application/json'];
            if (isset($options['headers'])) {
                $headers = array_merge($headers, $options['headers']);
            }
            curl_setopt($ch, CURLOPT_HTTPHEADER, $headers);

            $response = curl_exec($ch);
            $httpCode = curl_getinfo($ch, CURLINFO_HTTP_CODE);
            $error = curl_error($ch);
            curl_close($ch);

            // Handle errors
            if ($error) {
                throw new TemporaryException("Connection error: {$error}");
            }

            // Check HTTP status
            if ($httpCode >= 500) {
                throw new TemporaryException("Server error: HTTP {$httpCode}");
            }

            if ($httpCode === 429) {
                throw new TemporaryException("Rate limited: HTTP 429");
            }

            if ($httpCode >= 400) {
                throw new PermanentException("Client error: HTTP {$httpCode}");
            }

            return json_decode($response, true) ?? [];
        }, [
            'retryable_patterns' => [
                '/connection refused/i',
                '/timeout/i',
                '/reset by peer/i',
                '/temporary failure/i'
            ]
        ]);
    }
}
```

### Step 4: Implement Database Retry
```php
<?php
// /includes/retry/RetryableDatabase.php

namespace WHMCS\Retry;

class RetryableDatabase
{
    private $retryManager;

    public function __construct()
    {
        $this->retryManager = new RetryManager([
            'max_attempts' => 3,
            'base_delay' => 100,
            'multiplier' => 2,
            'retryable_patterns' => [
                '/deadlock/i',
                '/lock wait/i',
                '/try restarting transaction/i',
                '/connection.*lost/i'
            ]
        ]);
    }

    /**
     * Execute query with retry
     */
    public function query(string $sql, array $bindings = [])
    {
        return $this->retryManager->execute(function() use ($sql, $bindings) {
            return Capsule::select($sql, $bindings);
        });
    }

    /**
     * Execute transaction with retry
     */
    public function transaction(callable $callback, int $retries = 3)
    {
        $attempt = 0;

        while ($attempt < $retries) {
            $attempt++;

            try {
                return Capsule::connection()->transaction($callback);
            } catch (QueryException $e) {
                if (strpos(strtolower($e->getMessage()), 'deadlock') === false) {
                    throw $e;
                }

                if ($attempt >= $retries) {
                    throw $e;
                }

                // Exponential backoff
                $delay = pow(2, $attempt) * 100;
                usleep($delay * 1000);

                logActivity("Database deadlock, retrying (attempt {$attempt})");
            }
        }

        throw new Exception("Transaction failed after {$retries} retries");
    }
}
```

### Step 5: Create Module Call Retry
```php
<?php
// /includes/retry/RetryableModuleCall.php

namespace WHMCS\Retry;

/**
 * Execute module function with retry
 */
function retryableModuleCall(
    string $moduleType,
    string $moduleName,
    string $function,
    array $params,
    array $config = []
) {
    $retryManager = new RetryManager([
        'max_attempts' => $config['max_attempts'] ?? 3,
        'base_delay' => $config['base_delay'] ?? 1000,
        'multiplier' => $config['multiplier'] ?? 2
    ]);

    return $retryManager->execute(function() use ($moduleType, $moduleName, $function, $params) {
        $result = callModulFunction($moduleType, $moduleName, $function, $params);

        // Check for module-level error
        if (isset($result['error']) || isset($result['errorcode'])) {
            $errorMsg = $result['error'] ?? $result['errorcode'];
            throw new TemporaryException("Module error: {$errorMsg}");
        }

        return $result;
    }, [
        'retryable_patterns' => [
            '/timeout/i',
            '/connection/i',
            '/unavailable/i',
            '/busy/i',
            '/retry/i'
        ]
    ]);
}

/**
 * Provision service with retry
 */
function retryableProvision(int $serviceId, int $serverId = null)
{
    $service = Capsule::table('tblhosting')
        ->where('id', $serviceId)
        ->first();

    if (!$serverId) {
        $serverId = $service->server;
    }

    return retryableModuleCall(
        'provisioning',
        $service->servertype,
        'CreateAccount',
        ['serviceid' => $serviceId],
        ['max_attempts' => 5]
    );
}
```

### Step 6: Create Queue-Based Retry
```php
<?php
// /includes/retry/RetryQueue.php

namespace WHMCS\Retry;

class RetryQueue
{
    private $maxRetries;
    private $retryDelays; // delays in seconds for each retry

    public function __construct()
    {
        $this->maxRetries = 3;
        $this->retryDelays = [60, 300, 900]; // 1min, 5min, 15min
    }

    /**
     * Add job to retry queue
     */
    public function add(string $jobType, array $data, int $retryCount = 0, string $lastError = null)
    {
        if ($retryCount >= $this->maxRetries) {
            // Move to failed jobs
            $this->addToFailedJobs($jobType, $data, $lastError);
            return false;
        }

        $delay = $this->retryDelays[$retryCount] ?? end($this->retryDelays);

        Capsule::table('mod_retry_queue')->insert([
            'job_type' => $jobType,
            'job_data' => json_encode($data),
            'retry_count' => $retryCount + 1,
            'last_error' => $lastError,
            'available_at' => date('Y-m-d H:i:s', time() + $delay),
            'created_at' => date('Y-m-d H:i:s')
        ]);

        return true;
    }

    /**
     * Process retry queue
     */
    public function process()
    {
        $pendingJobs = Capsule::table('mod_retry_queue')
            ->where('available_at', '<=', date('Y-m-d H:i:s'))
            ->orderBy('available_at')
            ->limit(50)
            ->get();

        foreach ($pendingJobs as $job) {
            try {
                $this->processJob($job);
            } catch (Exception $e) {
                // Re-queue if retryable
                $this->add($job->job_type, json_decode($job->job_data, true), $job->retry_count, $e->getMessage());

                // Remove from queue
                Capsule::table('mod_retry_queue')
                    ->where('id', $job->id)
                    ->delete();
            }
        }
    }

    private function processJob($job)
    {
        $handler = $this->getHandler($job->job_type);

        $result = $handler(json_decode($job->job_data, true));

        // Success - remove from queue
        Capsule::table('mod_retry_queue')
            ->where('id', $job->id)
            ->delete();

        return $result;
    }

    private function addToFailedJobs(string $jobType, array $data, string $error)
    {
        Capsule::table('mod_failed_jobs')->insert([
            'job_type' => $jobType,
            'job_data' => json_encode($data),
            'error' => $error,
            'failed_at' => date('Y-m-d H:i:s')
        ]);
    }
}

// Process retry queue in cron
add_hook('CronJobHourly', 1, function($vars) {
    $queue = new RetryQueue();
    $queue->process();
});
```

### Step 7: Configure Retry Policies
```php
<?php
// /includes/retry/retry_policies.php

return [
    // HTTP API calls
    'api_calls' => [
        'max_attempts' => 5,
        'base_delay' => 500,
        'multiplier' => 2,
        'max_delay' => 30000,
        'jitter' => true,
        'retryable_status_codes' => [408, 429, 500, 502, 503, 504]
    ],

    // Database operations
    'database' => [
        'max_attempts' => 3,
        'base_delay' => 100,
        'multiplier' => 2,
        'max_delay' => 5000,
        'retryable_patterns' => ['/deadlock/i', '/lock wait/i']
    ],

    // Module provisioning
    'provisioning' => [
        'max_attempts' => 3,
        'base_delay' => 1000,
        'multiplier' => 2,
        'max_delay' => 30000,
        'retryable_patterns' => ['/timeout/i', '/unavailable/i']
    ],

    // Email sending
    'email' => [
        'max_attempts' => 3,
        'base_delay' => 2000,
        'multiplier' => 2,
        'max_delay' => 60000,
        'retryable_patterns' => ['/connection/i', '/smtp/i']
    ],

    // External integrations
    'external' => [
        'max_attempts' => 3,
        'base_delay' => 1000,
        'multiplier' => 2,
        'max_delay' => 30000
    ]
];
```

## Retry Strategy Comparison

| Strategy | Description | Best For |
|----------|-------------|----------|
| Fixed Delay | Same delay between retries | Predictable services |
| Linear Backoff | Increasing delay by fixed amount | Low-traffic systems |
| Exponential Backoff | Delay doubles each retry | Most scenarios |
| Jitter | Random variation added | High-traffic systems |

## Best Practices

1. **Set limits** - Maximum retry attempts
2. **Use backoff** - Exponential with jitter
3. **Classify errors** - Retry transient, not permanent
4. **Log attempts** - For debugging
5. **Alert on failure** - After max retries
6. **Circuit breaker** - Stop after repeated failures

## Related Workflows
- [WHMCS Error Recovery](./whmcs-error-recovery.md)
- [WHMCS Queue Processing](./whmcs-queue-processing.md)
- [WHMCS Transaction Management](./whmcs-transaction-management.md)