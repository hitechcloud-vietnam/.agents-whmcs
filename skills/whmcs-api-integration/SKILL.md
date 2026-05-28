# WHMCS API Integration Skill
# Version: 1.0 | Updated: 2026-05-28

## Purpose

Guide for integrating external APIs into WHMCS modules with proper error handling, rate limiting, and security.

## When to Use

- Building API clients for provisioning modules
- Integrating third-party services (cloud providers, payment APIs)
- Handling external API authentication (OAuth, API keys, JWT)

## API Client Patterns

### 1. Basic REST API Client

```php
<?php
namespace Provider;

class ApiClient {
    private string $baseUrl;
    private string $apiKey;
    private string $apiSecret;
    private int $timeout = 30;

    public function __construct(array $params) {
        $this->baseUrl = rtrim($params['base_url'] ?? '', '/');
        $this->apiKey = $params['api_key'];
        $this->apiSecret = $params['api_secret'];
        $this->timeout = $params['timeout'] ?? 30;
    }

    public function request(string $method, string $endpoint, array $data = []): array {
        $ch = curl_init();
        $url = $this->baseUrl . '/' . ltrim($endpoint, '/');

        $headers = [
            'Authorization: Bearer ' . $this->apiKey,
            'Content-Type: application/json',
            'Accept: application/json',
        ];

        curl_setopt_array($ch, [
            CURLOPT_URL => $url,
            CURLOPT_RETURNTRANSFER => true,
            CURLOPT_TIMEOUT => $this->timeout,
            CURLOPT_SSL_VERIFYPEER => true,
            CURLOPT_SSL_VERIFYHOST => 2,
            CURLOPT_HTTPHEADER => $headers,
        ]);

        if ($method !== 'GET') {
            curl_setopt($ch, CURLOPT_CUSTOMREQUEST, $method);
            curl_setopt($ch, CURLOPT_POSTFIELDS, json_encode($data));
        }

        $response = curl_exec($ch);
        $httpCode = curl_getinfo($ch, CURLINFO_HTTP_CODE);
        $error = curl_error($ch);
        curl_close($ch);

        if ($error) {
            throw new \Exception('cURL Error: ' . $error);
        }

        if ($httpCode >= 400) {
            $decoded = json_decode($response, true);
            $message = $decoded['message'] ?? $decoded['error'] ?? 'HTTP ' . $httpCode;
            throw new \Exception($message);
        }

        return json_decode($response, true) ?? [];
    }

    public function get(string $endpoint): array {
        return $this->request('GET', $endpoint);
    }

    public function post(string $endpoint, array $data = []): array {
        return $this->request('POST', $endpoint, $data);
    }

    public function put(string $endpoint, array $data = []): array {
        return $this->request('PUT', $endpoint, $data);
    }

    public function delete(string $endpoint): array {
        return $this->request('DELETE', $endpoint);
    }
}
```

### 2. OAuth2 API Client

```php
<?php
namespace Provider;

class OAuthClient {
    private string $clientId;
    private string $clientSecret;
    private string $authUrl;
    private string $tokenUrl;
    private string $accessToken;
    private int $tokenExpires;

    public function __construct(array $params) {
        $this->clientId = $params['client_id'];
        $this->clientSecret = $params['client_secret'];
        $this->authUrl = $params['auth_url'];
        $this->tokenUrl = $params['token_url'];
    }

    public function authenticate(): void {
        $ch = curl_init();
        curl_setopt_array($ch, [
            CURLOPT_URL => $this->tokenUrl,
            CURLOPT_POST => true,
            CURLOPT_POSTFIELDS => http_build_query([
                'grant_type' => 'client_credentials',
                'client_id' => $this->clientId,
                'client_secret' => $this->clientSecret,
            ]),
            CURLOPT_RETURNTRANSFER => true,
            CURLOPT_TIMEOUT => 30,
        ]);

        $response = curl_exec($ch);
        curl_close($ch);

        $data = json_decode($response, true);
        $this->accessToken = $data['access_token'];
        $this->tokenExpires = time() + ($data['expires_in'] ?? 3600);
    }

    public function getAccessToken(): string {
        if (empty($this->accessToken) || time() >= $this->tokenExpires - 60) {
            $this->authenticate();
        }
        return $this->accessToken;
    }

    public function request(string $method, string $endpoint, array $data = []): array {
        $ch = curl_init();
        curl_setopt_array($ch, [
            CURLOPT_URL => $endpoint,
            CURLOPT_RETURNTRANSFER => true,
            CURLOPT_TIMEOUT => 30,
            CURLOPT_HTTPHEADER => [
                'Authorization: Bearer ' . $this->getAccessToken(),
                'Content-Type: application/json',
            ],
        ]);

        if ($method !== 'GET') {
            curl_setopt($ch, CURLOPT_POST, true);
            curl_setopt($ch, CURLOPT_POSTFIELDS, json_encode($data));
        }

        $response = curl_exec($ch);
        curl_close($ch);

        return json_decode($response, true) ?? [];
    }
}
```

### 3. HMAC Signature API Client

```php
<?php
namespace Provider;

class HmacApiClient {
    private string $apiKey;
    private string $apiSecret;
    private string $baseUrl;

    public function __construct(array $params) {
        $this->apiKey = $params['api_key'];
        $this->apiSecret = $params['api_secret'];
        $this->baseUrl = $params['base_url'];
    }

    public function request(string $method, string $endpoint, array $data = []): array {
        $timestamp = time();
        $nonce = bin2hex(random_bytes(16));

        $payload = json_encode($data);
        $signature = $this->generateSignature($method, $endpoint, $timestamp, $nonce, $payload);

        $ch = curl_init();
        curl_setopt_array($ch, [
            CURLOPT_URL => $this->baseUrl . $endpoint,
            CURLOPT_RETURNTRANSFER => true,
            CURLOPT_TIMEOUT => 30,
            CURLOPT_POST => $method === 'POST',
            CURLOPT_POSTFIELDS => $payload,
            CURLOPT_HTTPHEADER => [
                'Content-Type: application/json',
                'X-API-Key: ' . $this->apiKey,
                'X-Timestamp: ' . $timestamp,
                'X-Nonce: ' . $nonce,
                'X-Signature: ' . $signature,
            ],
        ]);

        $response = curl_exec($ch);
        curl_close($ch);

        return json_decode($response, true) ?? [];
    }

    private function generateSignature(string $method, string $endpoint, int $timestamp, string $nonce, string $payload): string {
        $signData = implode("\n", [$method, $endpoint, $timestamp, $nonce, hash('sha256', $payload)]);
        return hash_hmac('sha256', $signData, $this->apiSecret);
    }
}
```

### 4. Retry and Rate Limiting

```php
<?php
namespace Provider;

class RetryApiClient {
    private array $config;
    private int $retryAttempts = 3;
    private int $retryDelay = 1000; // milliseconds

    public function __construct(array $params) {
        $this->config = $params;
    }

    public function requestWithRetry(string $method, string $endpoint, array $data = []): array {
        $lastException = null;

        for ($attempt = 0; $attempt < $this->retryAttempts; $attempt++) {
            try {
                return $this->makeRequest($method, $endpoint, $data);
            } catch (\Exception $e) {
                $lastException = $e;

                // Retry on rate limit or server error
                if (!$this->isRetryable($e)) {
                    throw $e;
                }

                if ($attempt < $this->retryAttempts - 1) {
                    usleep($this->retryDelay * 1000 * (2 ** $attempt)); // Exponential backoff
                }
            }
        }

        throw $lastException;
    }

    private function isRetryable(\Exception $e): bool {
        $message = $e->getMessage();
        return strpos($message, '429') !== false ||  // Rate limit
               strpos($message, '500') !== false ||  // Server error
               strpos($message, '503') !== false;    // Service unavailable
    }

    private function makeRequest(string $method, string $endpoint, array $data): array {
        // Implementation
    }
}
```

## Security Best Practices

```php
// 1. Never log sensitive data
// BAD: logActivity("API Key: " . $apiKey);
// GOOD: logActivity("API request to " . $endpoint);

// 2. Use HTTPS always
curl_setopt($ch, CURLOPT_SSL_VERIFYPEER, true);
curl_setopt($ch, CURLOPT_SSL_VERIFYHOST, 2);

// 3. Sanitize endpoint URLs
$endpoint = filter_var($endpoint, FILTER_SANITIZE_URL);

// 4. Validate response structure
$result = $api->request('GET', '/server/' . $id);
if (!isset($result['id']) || !isset($result['status'])) {
    throw new \Exception('Invalid API response');
}

// 5. Timeout for long operations
curl_setopt($ch, CURLOPT_TIMEOUT, 60);
curl_setopt($ch, CURLOPT_CONNECTTIMEOUT, 10);
```

## Checklist

- [ ] Base URL with proper trimming
- [ ] Authentication headers
- [ ] Error handling with meaningful messages
- [ ] HTTP status code checking
- [ ] SSL verification enabled
- [ ] Timeout configuration
- [ ] JSON encoding/decoding
- [ ] Retry logic for transient failures

---

**Related Skills:**
- whmcs-server-builder
- whmcs-gateway-builder
- whmcs-error-handling