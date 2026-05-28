# WHMCS API Integration Patterns
# Version: 1.0 | Updated: 2026-05-28

## Purpose

Reference patterns for integrating external APIs into WHMCS modules.

## API Client Pattern

```php
<?php
namespace WHMCS\Module\Server;

class ApiClient {
    private string $baseUrl;
    private string $apiKey;
    private int $timeout = 30;

    public function __construct(array $config) {
        $this->baseUrl = rtrim($config['server_url'], '/');
        $this->apiKey = $config['server_password']; // API key from server config
        $this->timeout = $config['timeout'] ?? 30;
    }

    public function request(string $method, string $endpoint, array $data = []): array {
        $url = $this->baseUrl . $endpoint;

        $ch = curl_init();
        curl_setopt_array($ch, [
            CURLOPT_URL => $url,
            CURLOPT_RETURNTRANSFER => true,
            CURLOPT_TIMEOUT => $this->timeout,
            CURLOPT_HTTPHEADER => $this->getHeaders(),
        ]);

        if (strtoupper($method) === 'POST') {
            curl_setopt($ch, CURLOPT_POST, true);
            curl_setopt($ch, CURLOPT_POSTFIELDS, json_encode($data));
        } elseif (strtoupper($method) !== 'GET') {
            curl_setopt($ch, CURLOPT_CUSTOMREQUEST, strtoupper($method));
            curl_setopt($ch, CURLOPT_POSTFIELDS, json_encode($data));
        }

        $response = curl_exec($ch);
        $error = curl_error($ch);
        $httpCode = curl_getinfo($ch, CURLINFO_HTTP_CODE);
        curl_close($ch);

        if ($error) {
            throw new \Exception("API Error: {$error}");
        }

        $result = json_decode($response, true);

        if ($httpCode >= 400) {
            $message = $result['message'] ?? "HTTP Error: {$httpCode}";
            throw new \Exception($message);
        }

        return $result;
    }

    private function getHeaders(): array {
        return [
            'Content-Type: application/json',
            'Accept: application/json',
            'Authorization: Bearer ' . $this->apiKey,
        ];
    }
}
```

## Retry Pattern

```php
<?php
class ResilientApiClient extends ApiClient {
    private int $maxRetries = 3;
    private int $retryDelay = 1000; // milliseconds

    public function request(string $method, string $endpoint, array $data = []): array {
        $lastException = null;

        for ($attempt = 1; $attempt <= $this->maxRetries; $attempt++) {
            try {
                return parent::request($method, $endpoint, $data);
            } catch (\Exception $e) {
                $lastException = $e;

                // Don't retry client errors (4xx)
                if ($this->isClientError($e)) {
                    throw $e;
                }

                if ($attempt < $this->maxRetries) {
                    $this->wait($attempt);
                    logActivity("API retry {$attempt}/{$this->maxRetries}: " . $e->getMessage());
                }
            }
        }

        throw $lastException;
    }

    private function isClientError(\Exception $e): bool {
        return preg_match('/HTTP [45]\d{2}/', $e->getMessage());
    }

    private function wait(int $attempt): void {
        $delay = $this->retryDelay * pow(2, $attempt - 1);
        usleep($delay * 1000);
    }
}
```

## Webhook Handler

```php
<?php
add_hook('ApiGatewaysCallback', 1, function($vars) {
    $gateway = $vars['gateway'];
    $data = $vars['data'];

    // Verify webhook signature
    if (!verifyWebhookSignature($data, $_SERVER['HTTP_X_SIGNATURE'])) {
        http_response_code(401);
        return ['error' => 'Invalid signature'];
    }

    // Process webhook
    $module = new PaymentGateway();
    $result = $module->handleWebhook($data);

    if ($result['success']) {
        http_response_code(200);
        return ['status' => 'ok'];
    } else {
        http_response_code(400);
        return ['error' => $result['error']];
    }
});

function verifyWebhookSignature(array $data, string $signature): bool {
    $payload = json_encode($data);
    $secret = Capsule::table('tblpaymentgateways')
        ->where('gateway', 'mypayment')
        ->where('setting', 'webhook_secret')
        ->value('value');

    $expected = hash_hmac('sha256', $payload, $secret);

    return hash_equals($expected, $signature);
}
```

## Rate Limiting

```php
<?php
class RateLimitedClient extends ApiClient {
    private int $requestsPerMinute = 60;
    private static array $requestTimes = [];

    public function request(string $method, string $endpoint, array $data = []): array {
        $this->throttle();

        return parent::request($method, $endpoint, $data);
    }

    private function throttle(): void {
        $now = microtime(true);
        $key = spl_object_id($this);

        // Remove old requests
        self::$requestTimes[$key] = array_filter(
            self::$requestTimes[$key] ?? [],
            fn($time) => $now - $time < 60
        );

        // Check limit
        $count = count(self::$requestTimes[$key] ?? []);
        if ($count >= $this->requestsPerMinute) {
            $oldest = min(self::$requestTimes[$key]);
            $waitTime = 60 - ($now - $oldest);

            if ($waitTime > 0) {
                usleep((int)($waitTime * 1000000));
            }
        }

        self::$requestTimes[$key][] = microtime(true);
    }
}
```

## Circuit Breaker

```php
<?php
class CircuitBreaker {
    private string $name;
    private int $failureThreshold = 5;
    private int $timeout = 60; // seconds
    private string $state = 'closed'; // closed, open, half-open

    public function __construct(string $name) {
        $this->name = $name;
        $this->loadState();
    }

    public function execute(callable $operation): mixed {
        if ($this->state === 'open') {
            if ($this->shouldAttemptReset()) {
                $this->state = 'half-open';
            } else {
                throw new \Exception("Circuit breaker is open");
            }
        }

        try {
            $result = $operation();
            $this->onSuccess();
            return $result;
        } catch (\Exception $e) {
            $this->onFailure();
            throw $e;
        }
    }

    private function onSuccess(): void {
        $this->state = 'closed';
        $this->failureCount = 0;
        $this->saveState();
    }

    private function onFailure(): void {
        $this->failureCount++;

        if ($this->failureCount >= $this->failureThreshold) {
            $this->state = 'open';
            $this->lastFailureTime = time();
            $this->saveState();
        }
    }
}
```

## Caching Responses

```php
<?php
trait ApiCaching {
    private static array $cache = [];

    protected function cached(string $key, callable $fetcher, int $ttl = 300): mixed {
        $cacheKey = md5($key);

        if (isset(self::$cache[$cacheKey])) {
            $entry = self::$cache[$cacheKey];
            if (time() - $entry['time'] < $ttl) {
                return $entry['data'];
            }
        }

        $data = $fetcher();
        self::$cache[$cacheKey] = [
            'data' => $data,
            'time' => time(),
        ];

        return $data;
    }

    public function clearCache(string $pattern = null): void {
        if ($pattern) {
            foreach (self::$cache as $key => $entry) {
                if (strpos($key, $pattern) !== false) {
                    unset(self::$cache[$key]);
                }
            }
        } else {
            self::$cache = [];
        }
    }
}
```

---

**Related Skills:**
- whmcs-api-integration
- whmcs-webhook-handler
- whmcs-rate-limiting