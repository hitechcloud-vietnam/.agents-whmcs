# WHMCS Error Handling Skill
# Version: 1.0 | Updated: 2026-05-28

## Purpose

Guide for implementing robust error handling in WHMCS modules.

## When to Use

- Building resilient modules
- Handling API failures gracefully
- Logging errors for debugging

## Error Handling Patterns

### Module-Level Error Handling

```php
function module_CreateAccount(array $params): string {
    try {
        // Validate input
        if (empty($params['domain'])) {
            return 'Error: Domain name is required';
        }

        // Call API
        $api = new ApiClient($params);
        $result = $api->createServer($params);

        // Check result
        if (isset($result['error'])) {
            return 'Error: ' . $result['error'];
        }

        return 'success';

    } catch (\Exception $e) {
        // Log error
        logActivity('Module CreateAccount Error: ' . $e->getMessage());

        // Return user-friendly error
        return 'Error: Service provisioning failed. Please try again later.';
    }
}
```

### API Error Handling

```php
class ApiClient {
    public function request(string $method, string $endpoint, array $data = []): array {
        $ch = curl_init();
        curl_setopt_array($ch, [
            CURLOPT_URL => $this->baseUrl . $endpoint,
            CURLOPT_RETURNTRANSFER => true,
            CURLOPT_TIMEOUT => 30,
            CURLOPT_SSL_VERIFYPEER => true,
        ]);

        $response = curl_exec($ch);
        $httpCode = curl_getinfo($ch, CURLINFO_HTTP_CODE);
        $error = curl_error($ch);
        curl_close($ch);

        // Handle cURL errors
        if ($error) {
            throw new \Exception('Connection error: ' . $error);
        }

        // Handle HTTP errors
        if ($httpCode >= 400) {
            $decoded = json_decode($response, true);
            $message = $decoded['message'] ?? 'HTTP Error ' . $httpCode;

            switch ($httpCode) {
                case 401:
                    throw new \Exception('Authentication failed. Please check your API credentials.');
                case 403:
                    throw new \Exception('Access denied. Please check your permissions.');
                case 404:
                    throw new \Exception('Resource not found.');
                case 429:
                    throw new \Exception('Rate limit exceeded. Please try again later.');
                case 500:
                case 502:
                case 503:
                    throw new \Exception('Server error. Please try again later.');
                default:
                    throw new \Exception($message);
            }
        }

        $result = json_decode($response, true);

        // Handle API errors
        if (isset($result['error'])) {
            throw new \Exception($result['error']);
        }

        return $result;
    }
}
```

### Validation Errors

```php
function validateCreateParams(array $params): void {
    $errors = [];

    // Domain validation
    if (empty($params['domain'])) {
        $errors[] = 'Domain name is required';
    } elseif (!preg_match('/^[a-zA-Z0-9][a-zA-Z0-9-]{0,61}[a-zA-Z0-9]$/', $params['domain'])) {
        $errors[] = 'Invalid domain name format';
    }

    // Password validation
    if (empty($params['password'])) {
        $errors[] = 'Password is required';
    } elseif (strlen($params['password']) < 8) {
        $errors[] = 'Password must be at least 8 characters';
    }

    // Configuration validation
    if (empty($params['configoption1'])) {
        $errors[] = 'Plan selection is required';
    }

    if (!empty($errors)) {
        throw new \Exception('Validation failed: ' . implode(', ', $errors));
    }
}
```

### Retry Logic

```php
function requestWithRetry(callable $operation, int $maxAttempts = 3, int $delayMs = 1000): mixed {
    $lastException = null;

    for ($attempt = 1; $attempt <= $maxAttempts; $attempt++) {
        try {
            return $operation();
        } catch (\Exception $e) {
            $lastException = $e;

            // Don't retry on non-retryable errors
            if (!isRetryableError($e)) {
                throw $e;
            }

            // Don't wait after last attempt
            if ($attempt < $maxAttempts) {
                usleep($delayMs * 1000 * $attempt);
            }
        }
    }

    throw $lastException;
}

function isRetryableError(\Exception $e): bool {
    $message = $e->getMessage();

    // Network errors
    if (strpos($message, 'Connection error') !== false) return true;
    if (strpos($message, 'timeout') !== false) return true;

    // Server errors
    if (strpos($message, '500') !== false) return true;
    if (strpos($message, '503') !== false) return true;
    if (strpos($message, 'Rate limit') !== false) return true;

    return false;
}
```

## Error Logging

```php
function logModuleError(string $level, string $message, array $context = []): void {
    // Log to WHMCS activity log
    if ($level === 'error') {
        logActivity('[' . strtoupper($level) . '] ' . $message);
    }

    // Log to module-specific log table
    Capsule::table('mod_{module}_logs')->insert([
        'level' => $level,
        'message' => $message,
        'context' => json_encode($context),
        'ip_address' => $_SERVER['REMOTE_ADDR'] ?? '',
        'user_agent' => $_SERVER['HTTP_USER_AGENT'] ?? '',
        'created_at' => date('Y-m-d H:i:s'),
    ]);
}

function logApiError(string $endpoint, array $response, \Exception $e): void {
    logModuleError('error', 'API Error: ' . $e->getMessage(), [
        'endpoint' => $endpoint,
        'response_code' => $response['code'] ?? 0,
        'response_body' => json_encode($response),
    ]);
}
```

## User-Friendly Messages

```php
function getUserMessage(\Exception $e): string {
    $message = $e->getMessage();

    // Map technical messages to user-friendly ones
    $messages = [
        'Connection error' => 'Unable to connect to the service. Please try again later.',
        'Authentication failed' => 'Invalid API credentials. Please check your configuration.',
        'Rate limit exceeded' => 'Too many requests. Please wait a moment and try again.',
        'Resource not found' => 'The requested resource was not found.',
        'timeout' => 'The request timed out. Please try again.',
    ];

    foreach ($messages as $technical => $friendly) {
        if (strpos($message, $technical) !== false) {
            return $friendly;
        }
    }

    // Generic fallback
    return 'An error occurred. Please try again or contact support.';
}
```

## Checklist

- [ ] All API calls wrapped in try/catch
- [ ] User-friendly error messages
- [ ] Errors logged for debugging
- [ ] Validation before API calls
- [ ] Retry logic for transient errors
- [ ] No sensitive data in error messages

---

**Related Skills:**
- whmcs-api-integration
- whmcs-testing-qa
- whmcs-logging