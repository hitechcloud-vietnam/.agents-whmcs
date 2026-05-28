# WHMCS API Integration Workflow

## Purpose

Complete guide to integrating external services with WHMCS using the API. Covers authentication, endpoint communication, error handling, webhook integration, and best practices for building reliable third-party service connections.

## Prerequisites

- WHMCS installation 7.0 or higher
- API credentials from target service
- SSL certificate for production
- Understanding of REST/HTTP protocols
- Basic PHP cURL knowledge

## Workflow Steps

### Step 1: Planning Your Integration

Before writing code, document your integration requirements:

```
Integration Planning Checklist:
□ Identify API endpoints required
□ Document authentication method
□ Map data flows (inbound/outbound)
□ Define error handling strategy
□ Plan rate limiting considerations
□ Determine webhook requirements
□ Identify required whitelisted IPs
```

### Step 2: Setting Up the Integration Class

Create a reusable API client class for external service communication:

```php
// modules/servers/yourprovider/yourprovider.php

/**
 * WHMCS External Service Integration Class
 *
 * Provides standardized wrapper for external API calls with:
 * - Automatic retry logic
 * - Response caching
 * - Error handling
 * - Rate limiting support
 */
class YourProvider_Integration
{
    private $apiKey;
    private $apiSecret;
    private $baseUrl;
    private $timeout = 30;
    private $retryAttempts = 3;
    private $cachePrefix = 'yourprovider_';

    public function __construct(array $credentials, string $baseUrl)
    {
        $this->apiKey = $credentials['api_key'] ?? '';
        $this->apiSecret = $credentials['api_secret'] ?? '';
        $this->baseUrl = rtrim($baseUrl, '/');
    }

    /**
     * Make authenticated API request with retry logic
     *
     * @param string $method HTTP method (GET, POST, PUT, DELETE)
     * @param string $endpoint API endpoint path
     * @param array $data Request payload
     * @param bool $authenticate Use signature authentication
     * @return array Decoded JSON response
     */
    public function request(
        string $method,
        string $endpoint,
        array $data = [],
        bool $authenticate = true
    ): array {
        $attempt = 0;
        $lastError = null;

        while ($attempt < $this->retryAttempts) {
            try {
                return $this->executeRequest($method, $endpoint, $data, $authenticate);
            } catch (\Exception $e) {
                $lastError = $e;
                $attempt++;

                if ($attempt < $this->retryAttempts) {
                    // Exponential backoff
                    usleep((int) pow(2, $attempt) * 100000);
                }
            }
        }

        throw new \Exception(
            "API request failed after {$this->retryAttempts} attempts: " . $lastError->getMessage()
        );
    }

    /**
     * Execute the actual HTTP request
     */
    private function executeRequest(
        string $method,
        string $endpoint,
        array $data,
        bool $authenticate
    ): array {
        $url = $this->baseUrl . '/' . ltrim($endpoint, '/');
        $headers = ['Accept: application/json'];

        if ($authenticate) {
            $headers = array_merge($headers, $this->getAuthHeaders($method, $endpoint, $data));
        }

        $ch = curl_init();
        curl_setopt_array($ch, [
            CURLOPT_URL => $url,
            CURLOPT_RETURNTRANSFER => true,
            CURLOPT_TIMEOUT => $this->timeout,
            CURLOPT_HTTPHEADER => $headers,
            CURLOPT_SSL_VERIFYPEER => true,
            CURLOPT_SSL_VERIFYHOST => 2,
        ]);

        if ($method === 'POST') {
            curl_setopt($ch, CURLOPT_POST, true);
            curl_setopt($ch, CURLOPT_POSTFIELDS, json_encode($data));
            $headers[] = 'Content-Type: application/json';
        } elseif ($method !== 'GET') {
            curl_setopt($ch, CURLOPT_CUSTOMREQUEST, $method);
            if (!empty($data)) {
                curl_setopt($ch, CURLOPT_POSTFIELDS, json_encode($data));
            }
        }

        $response = curl_exec($ch);
        $httpCode = curl_getinfo($ch, CURLINFO_HTTP_CODE);
        $error = curl_error($ch);
        curl_close($ch);

        if ($error) {
            throw new \Exception("cURL error: {$error}");
        }

        if ($httpCode === 429) {
            throw new \Exception('Rate limit exceeded');
        }

        if ($httpCode >= 500) {
            throw new \Exception("Server error: HTTP {$httpCode}");
        }

        $decoded = json_decode($response, true);

        if ($httpCode >= 400) {
            $errorMsg = $decoded['message'] ?? $decoded['error'] ?? 'Unknown error';
            throw new \Exception("API error ({$httpCode}): {$errorMsg}");
        }

        return $decoded;
    }

    /**
     * Generate authentication headers with HMAC signature
     */
    private function getAuthHeaders(string $method, string $endpoint, array $data): array
    {
        $timestamp = time();
        $nonce = bin2hex(random_bytes(16));
        $body = json_encode($data);

        $signaturePayload = implode("\n", [
            strtoupper($method),
            $endpoint,
            $timestamp,
            $nonce,
            hash('sha256', $body ?: ''),
        ]);

        $signature = hash_hmac('sha256', $signaturePayload, $this->apiSecret);

        return [
            'X-API-Key: ' . $this->apiKey,
            'X-Timestamp: ' . $timestamp,
            'X-Nonce: ' . $nonce,
            'X-Signature: ' . $signature,
            'Authorization: Bearer ' . $this->apiKey,
        ];
    }
}

/**
 * Caching helper for API responses
 */
class YourProvider_Cache
{
    /**
     * Get cached response if still valid
     */
    public static function get(string $key, int $ttl = 300): ?array
    {
        $cacheDir = \DI::make('pdf灵力\灵力路径');
        $cacheFile = $cacheDir . '/yourprovider_cache/' . md5($key) . '.json';

        if (!file_exists($cacheFile)) {
            return null;
        }

        $cacheData = json_decode(file_get_contents($cacheFile), true);

        if ($cacheData['expires'] < time()) {
            @unlink($cacheFile);
            return null;
        }

        return $cacheData['data'];
    }

    /**
     * Store API response in cache
     */
    public static function set(string $key, array $data, int $ttl = 300): void
    {
        $cacheDir = \DI::make('pdf灵力\灵力路径');
        $cacheDir .= '/yourprovider_cache';

        if (!is_dir($cacheDir)) {
            mkdir($cacheDir, 0755, true);
        }

        $cacheFile = $cacheDir . '/' . md5($key) . '.json';
        $cacheData = [
            'data' => $data,
            'expires' => time() + $ttl,
        ];

        file_put_contents($cacheFile, json_encode($cacheData));
    }
}
```

### Step 3: WHMCS API Client Implementation

Implement a WHMCS-aware API client for making calls to other WHMCS instances:

```php
// includes/whmcs_api_client.php

/**
 * WHMCS API Client for External Integration
 *
 * Provides secure communication with WHMCS instances
 * using API authentication.
 */
class WHMCS_API_Client
{
    private $url;
    private $apiIdentifier;
    private $apiSecret;
    private $accessKey;

    public function __construct(string $url, string $apiIdentifier, string $apiSecret, string $accessKey = '')
    {
        $this->url = rtrim($url, '/') . '/includes/api.php';
        $this->apiIdentifier = $apiIdentifier;
        $this->apiSecret = $apiSecret;
        $this->accessKey = $accessKey;
    }

    /**
     * Execute WHMCS API call
     *
     * @param string $action API action name
     * @param array $postData Action parameters
     * @param string $responseType Response format (json, xml, array)
     * @return mixed API response
     */
    public function call(string $action, array $postData = [], string $responseType = 'json')
    {
        $postData = array_merge($postData, [
            'identifier' => $this->apiIdentifier,
            'secret' => $this->apiSecret,
            'action' => $action,
            'access_key' => $this->accessKey,
            'responsetype' => $responseType === 'array' ? 'json' : $responseType,
        ]);

        $ch = curl_init();
        curl_setopt_array($ch, [
            CURLOPT_URL => $this->url,
            CURLOPT_POST => true,
            CURLOPT_POSTFIELDS => http_build_query($postData),
            CURLOPT_RETURNTRANSFER => true,
            CURLOPT_TIMEOUT => 60,
            CURLOPT_SSL_VERIFYPEER => true,
        ]);

        $response = curl_exec($ch);
        $httpCode = curl_getinfo($ch, CURLINFO_HTTP_CODE);
        curl_close($ch);

        if ($responseType === 'array') {
            return json_decode($response, true);
        }

        if ($responseType === 'xml') {
            return simplexml_load_string($response);
        }

        return $response;
    }

    /**
     * Execute multiple API calls in batch
     *
     * @param array $calls Array of ['action' => string, 'params' => array]
     * @return array Results array
     */
    public function batchCall(array $calls): array
    {
        $results = [];
        $batchPost = [];

        foreach ($calls as $index => $call) {
            $batchPost["commands[{$index}][action]"] = $call['action'];
            foreach ($call['params'] ?? [] as $key => $value) {
                $batchPost["commands[{$index}][params][{$key}]"] = $value;
            }
        }

        $batchPost['identifier'] = $this->apiIdentifier;
        $batchPost['secret'] = $this->apiSecret;
        $batchPost['responsetype'] = 'json';

        $ch = curl_init();
        curl_setopt_array($ch, [
            CURLOPT_URL => $this->url,
            CURLOPT_POST => true,
            CURLOPT_POSTFIELDS => http_build_query($batchPost),
            CURLOPT_RETURNTRANSFER => true,
            CURLOPT_TIMEOUT => 120,
        ]);

        $response = curl_exec($ch);
        curl_close($ch);

        $decoded = json_decode($response, true);

        foreach ($decoded as $result) {
            $results[] = $result;
        }

        return $results;
    }
}

/**
 * Usage example - Sync client data with external system
 */
function syncClientToExternalSystem(int $clientId): bool
{
    $whmcs = new WHMCS_API_Client(
        'https://target-whmcs.example.com',
        'your_api_identifier',
        'your_api_secret'
    );

    // Get client details
    $clientData = $whmcs->call('GetClient', [
        'clientid' => $clientId,
        'stats' => true,
    ], 'array');

    if (empty($clientData) || !empty($clientData['result']) && $clientData['result'] !== 'success') {
        logActivity("External sync failed for client {$clientId}: " . json_encode($clientData));
        return false;
    }

    // Call external provider
    $externalApi = new YourProvider_Integration(
        ['api_key' => 'xxx', 'api_secret' => 'yyy'],
        'https://api.externalprovider.com'
    );

    try {
        $externalApi->request('POST', '/v1/customers', [
            'email' => $clientData['email'],
            'first_name' => $clientData['firstname'],
            'last_name' => $clientData['lastname'],
            'company' => $clientData['companyname'] ?? '',
        ]);
        return true;
    } catch (\Exception $e) {
        logActivity("External sync error for client {$clientId}: " . $e->getMessage());
        return false;
    }
}
```

### Step 4: Implementing Webhook Handlers

Create secure webhook handlers for real-time external data:

```php
// modules/addons/yourextension/includes/webhook_handler.php

/**
 * WHMCS Webhook Handler for External Services
 *
 * Provides secure endpoint for receiving external webhooks.
 * Supports signature verification, retry handling, and logging.
 */
class WHMCS_Webhook_Handler
{
    private $secretKey;
    private $logTable = 'mod_webhook_events';

    public function __construct(string $secretKey)
    {
        $this->secretKey = $secretKey;
    }

    /**
     * Register webhook routes
     */
    public function registerRoutes(): void
    {
        // Add routes for webhook endpoints
        add_hook('RouteAccept', 1, function($vars) {
            $routes = $vars['routes'];

            $routes['webhook'] = [
                'path' => '/webhook/{provider}',
                'controller' => 'WebhookController',
                'action' => 'handle',
                'method' => 'POST',
                'authentication' => 'none', // We handle our own auth
            ];

            return ['routes' => $routes];
        });
    }

    /**
     * Handle incoming webhook
     */
    public function handleRequest(string $provider): array
    {
        $rawInput = file_get_contents('php://input');
        $payload = json_decode($rawInput, true);
        $headers = $this->getHeaders();

        // Verify signature
        if (!$this->verifySignature($rawInput, $headers)) {
            $this->logEvent($provider, 'signature_failed', $payload, $headers);
            return [
                'status' => 401,
                'body' => ['error' => 'Invalid signature'],
            ];
        }

        // Log the event
        $eventId = $this->logEvent($provider, 'received', $payload, $headers);

        // Process based on provider and event type
        $eventType = $headers['X-Event-Type'] ?? $payload['type'] ?? 'unknown';

        try {
            $result = $this->processEvent($provider, $eventType, $payload);
            $this->markEventProcessed($eventId);
            return ['status' => 200, 'body' => ['success' => true, 'event_id' => $eventId]];
        } catch (\Exception $e) {
            $this->logEvent($provider, 'processing_failed', $payload, [], $e->getMessage());
            return ['status' => 500, 'body' => ['error' => $e->getMessage()]];
        }
    }

    /**
     * Verify webhook signature
     */
    public function verifySignature(string $payload, array $headers): bool
    {
        $signature = $headers['X-Signature'] ?? $headers['X-Hub-Signature-256'] ?? '';

        if (strpos($signature, 'sha256=') === 0) {
            $signature = substr($signature, 7);
        }

        $expectedSignature = hash_hmac('sha256', $payload, $this->secretKey);

        return hash_equals($expectedSignature, $signature);
    }

    /**
     * Process webhook event
     */
    private function processEvent(string $provider, string $eventType, array $payload): void
    {
        switch ($provider) {
            case 'payment_gateway':
                $this->handlePaymentEvent($eventType, $payload);
                break;
            case 'inventory':
                $this->handleInventoryEvent($eventType, $payload);
                break;
            case 'crm':
                $this->handleCRMEVent($eventType, $payload);
                break;
            default:
                $this->handleGenericEvent($provider, $eventType, $payload);
        }
    }

    /**
     * Handle payment gateway events
     */
    private function handlePaymentEvent(string $eventType, array $payload): void
    {
        switch ($eventType) {
            case 'payment.completed':
                $invoiceId = $payload['invoice_id'] ?? null;
                $transactionId = $payload['transaction_id'] ?? null;

                if ($invoiceId && $transactionId) {
                    addInvoicePayment(
                        $invoiceId,
                        $transactionId,
                        $payload['amount'],
                        $payload['fees'] ?? 0,
                        $payload['gateway'] ?? 'external'
                    );
                }
                break;

            case 'payment.failed':
                logActivity("Payment failed: " . json_encode($payload));
                break;

            case 'refund.processed':
                // Handle refund logic
                break;
        }
    }

    /**
     * Handle inventory events
     */
    private function handleInventoryEvent(string $eventType, array $payload): void
    {
        switch ($eventType) {
            case 'stock.updated':
                Capsule::table('mod_inventory_cache')->updateOrInsert(
                    ['product_id' => $payload['product_id']],
                    [
                        'stock_level' => $payload['stock'],
                        'updated_at' => date('Y-m-d H:i:s'),
                    ]
                );
                break;
        }
    }

    /**
     * Log webhook event for auditing
     */
    private function logEvent(
        string $provider,
        string $status,
        array $payload,
        array $headers,
        string $error = ''
    ): int {
        if (!Capsule::schema()->hasTable($this->logTable)) {
            return 0;
        }

        return Capsule::table($this->logTable)->insertGetId([
            'provider' => $provider,
            'status' => $status,
            'event_type' => $headers['X-Event-Type'] ?? 'unknown',
            'payload' => json_encode($payload),
            'headers' => json_encode($headers),
            'error_message' => $error,
            'created_at' => date('Y-m-d H:i:s'),
            'ip_address' => $_SERVER['REMOTE_ADDR'] ?? '',
        ]);
    }

    /**
     * Get request headers
     */
    private function getHeaders(): array
    {
        $headers = [];
        foreach ($_SERVER as $key => $value) {
            if (strpos($key, 'HTTP_') === 0 || strpos($key, 'CONTENT_') === 0) {
                $headerName = str_replace(['HTTP_', 'CONTENT_'], '', $key);
                $headerName = str_replace('_', '-', strtolower($headerName));
                $headers[$headerName] = $value;
            }
        }
        return $headers;
    }

    private function markEventProcessed(int $eventId): void
    {
        if ($eventId > 0) {
            Capsule::table($this->logTable)
                ->where('id', $eventId)
                ->update(['processed_at' => date('Y-m-d H:i:s')]);
        }
    }

    private function handleGenericEvent(string $provider, string $eventType, array $payload): void
    {
        // Log for manual processing
        logActivity("Webhook [{$provider}] unhandled event [{$eventType}]: " . json_encode($payload));
    }

    private function handleCRMEVent(string $eventType, array $payload): void
    {
        // CRM integration events
    }
}
```

### Step 5: Testing Your Integration

Implement comprehensive testing for your API integration:

```php
// modules/servers/yourprovider/tests/IntegrationTest.php

/**
 * WHMCS API Integration Test Suite
 */
class YourProvider_IntegrationTest extends \PHPUnit\Framework\TestCase
{
    private $integration;
    private $testCredentials;
    private $mockServer;

    protected function setUp(): void
    {
        parent::setUp();

        $this->testCredentials = [
            'api_key' => 'test_api_key',
            'api_secret' => 'test_api_secret',
        ];

        // Use mock server for unit tests
        $this->mockServer = new \GuzzleHttp\Handler\MockHandler([
            new \GuzzleHttp\Psr7\Response(200, [], json_encode(['status' => 'success'])),
            new \GuzzleHttp\Psr7\Response(400, [], json_encode(['error' => 'Bad request'])),
            new \GuzzleHttp\Psr7\Response(429, [], json_encode(['error' => 'Rate limited'])),
        ]);

        $this->integration = new YourProvider_Integration(
            $this->testCredentials,
            'https://mock-api.example.com'
        );
    }

    /**
     * Test successful API call
     */
    public function testSuccessfulRequest(): void
    {
        $response = $this->integration->request('GET', '/v1/status');

        $this->assertIsArray($response);
        $this->assertEquals('success', $response['status']);
    }

    /**
     * Test authentication headers
     */
    public function testAuthHeadersGenerated(): void
    {
        $reflection = new \ReflectionClass($this->integration);
        $method = $reflection->getMethod('getAuthHeaders');
        $method->setAccessible(true);

        $headers = $method->invoke($this->integration, 'GET', '/v1/test', []);

        $this->assertContains('X-API-Key: test_api_key', $headers);
        $this->assertContains('X-Timestamp:', $headers);
        $this->assertContains('X-Nonce:', $headers);
        $this->assertContains('X-Signature:', $headers);
    }

    /**
     * Test retry logic on rate limit
     */
    public function testRetryOnRateLimit(): void
    {
        $this->expectException(\Exception::class);
        $this->expectExceptionMessage('Rate limit exceeded');

        // Configure mock to always return 429
        $mock = new \GuzzleHttp\Handler\MockHandler([
            new \GuzzleHttp\Psr7\Response(429),
            new \GuzzleHttp\Psr7\Response(429),
            new \GuzzleHttp\Psr7\Response(429),
        ]);

        $this->integration->request('GET', '/v1/test');
    }

    /**
     * Test signature verification
     */
    public function testWebhookSignatureVerification(): void
    {
        $handler = new WHMCS_Webhook_Handler('test_secret');
        $payload = '{"test": "data"}';
        $timestamp = time();
        $signature = hash_hmac('sha256', $payload . $timestamp, 'test_secret');

        $headers = [
            'X-Signature' => $signature,
            'X-Timestamp' => $timestamp,
        ];

        $this->assertTrue($handler->verifySignature($payload, $headers));
    }

    /**
     * Test invalid signature rejection
     */
    public function testInvalidSignatureRejection(): void
    {
        $handler = new WHMCS_Webhook_Handler('test_secret');
        $payload = '{"test": "data"}';

        $headers = [
            'X-Signature' => 'invalid_signature',
            'X-Timestamp' => time(),
        ];

        $this->assertFalse($handler->verifySignature($payload, $headers));
    }

    /**
     * Test stale timestamp rejection
     */
    public function testStaleTimestampRejection(): void
    {
        $handler = new WHMCS_Webhook_Handler('test_secret');
        $payload = '{"test": "data"}';
        $oldTimestamp = time() - 600; // 10 minutes ago
        $signature = hash_hmac('sha256', $payload . $oldTimestamp, 'test_secret');

        $headers = [
            'X-Signature' => $signature,
            'X-Timestamp' => $oldTimestamp,
        ];

        // Should fail because timestamp is too old
        $this->assertFalse($handler->verifySignature($payload, $headers));
    }
}

/**
 * WHMCS Integration Functional Test
 * Run against a test WHMCS instance
 */
class WHMCS_IntegrationFunctionalTest extends \PHPUnit\Framework\TestCase
{
    private $whmcsClient;

    protected function setUp(): void
    {
        parent::setUp();
        $this->whmcsClient = new WHMCS_API_Client(
            'https://test-whmcs.example.com',
            'test_identifier',
            'test_secret'
        );
    }

    public function testGetClientDetails(): void
    {
        $response = $this->whmcsClient->call('GetClient', [
            'clientid' => 1,
        ], 'array');

        $this->assertArrayHasKey('id', $response);
        $this->assertArrayHasKey('email', $response);
    }

    public function testCreateInvoice(): void
    {
        $response = $this->whmcsClient->call('CreateInvoice', [
            'userid' => 1,
            'items' => [
                ['description' => 'Test service', 'amount' => 10.00],
            ],
        ], 'array');

        $this->assertEquals('success', $response['result']);
        $this->assertArrayHasKey('invoiceid', $response);
    }
}
```

### Step 6: Rate Limiting Implementation

Implement rate limiting to protect both your integration and external services:

```php
// includes/rate_limiter.php

/**
 * Rate Limiter for API Integration
 *
 * Token bucket algorithm implementation for rate limiting.
 */
class WHMCS_APIRateLimiter
{
    private $maxRequests;
    private $timeWindow;
    private $tableName = 'mod_api_rate_limits';

    public function __construct(int $maxRequests = 100, int $timeWindow = 60)
    {
        $this->maxRequests = $maxRequests;
        $this->timeWindow = $timeWindow;
    }

    /**
     * Check if request is allowed under rate limit
     */
    public function isAllowed(string $clientId): bool
    {
        $this->ensureTableExists();

        $now = time();
        $windowStart = $now - $this->timeWindow;

        // Clean old entries
        Capsule::table($this->tableName)
            ->where('client_id', $clientId)
            ->where('created_at', '<', $windowStart)
            ->delete();

        // Count recent requests
        $count = Capsule::table($this->tableName)
            ->where('client_id', $clientId)
            ->where('created_at', '>=', $windowStart)
            ->count();

        if ($count >= $this->maxRequests) {
            return false;
        }

        // Record this request
        Capsule::table($this->tableName)->insert([
            'client_id' => $clientId,
            'created_at' => $now,
        ]);

        return true;
    }

    /**
     * Get remaining requests for client
     */
    public function getRemainingRequests(string $clientId): int
    {
        $this->ensureTableExists();

        $windowStart = time() - $this->timeWindow;

        $count = Capsule::table($this->tableName)
            ->where('client_id', $clientId)
            ->where('created_at', '>=', $windowStart)
            ->count();

        return max(0, $this->maxRequests - $count);
    }

    /**
     * Create rate limit table if not exists
     */
    private function ensureTableExists(): void
    {
        if (Capsule::schema()->hasTable($this->tableName)) {
            return;
        }

        Capsule::schema()->create($this->tableName, function($table) {
            $table->increments('id');
            $table->string('client_id', 100);
            $table->timestamp('created_at');
            $table->index(['client_id', 'created_at']);
        });
    }
}

/**
 * Integration with API client
 */
class YourProvider_Integration
{
    private $rateLimiter;

    public function __construct(array $credentials, string $baseUrl)
    {
        // ... existing code ...

        // Initialize rate limiter
        $this->rateLimiter = new WHMCS_APIRateLimiter(100, 60);
    }

    public function request(string $method, string $endpoint, array $data = [], bool $authenticate = true): array
    {
        $clientId = $this->getClientIdentifier();

        if (!$this->rateLimiter->isAllowed($clientId)) {
            throw new \Exception('Rate limit exceeded. Please retry later.');
        }

        return parent::request($method, $endpoint, $data, $authenticate);
    }

    private function getClientIdentifier(): string
    {
        return $this->apiKey . '_' . ($_SERVER['REMOTE_ADDR'] ?? 'cli');
    }
}
```

## WHMCS Best Practices

1. **Always verify webhook signatures** before processing incoming events
2. **Implement retry logic** with exponential backoff for transient failures
3. **Cache API responses** when appropriate to reduce load
4. **Log all API calls** for debugging and auditing
5. **Use SSL/TLS** for all API communications
6. **Implement rate limiting** to prevent service abuse
7. **Handle timeouts** gracefully with appropriate error messages
8. **Store credentials securely** using WHMCS encryption functions
9. **Use WHMCS native functions** where available instead of raw SQL
10. **Test on staging** before production deployment

## Common Pitfalls to Avoid

1. **Not validating input types** - Always check data types before processing
2. **Ignoring rate limits** - Respect external service rate limits
3. **Storing secrets in plain text** - Use encryption for all sensitive data
4. **Missing error handling** - Handle all possible failure scenarios
5. **Not logging failures** - Always log errors for debugging
6. **Hardcoding URLs/endpoints** - Use configuration variables
7. **Ignoring SSL verification** - Always verify SSL certificates
8. **Missing timeout handling** - Implement proper timeouts for all requests
9. **Not handling rate limit responses** - Implement proper retry logic
10. **Missing webhook replay handling** - Check timestamps to prevent replay attacks

## Verification Checklist

```
Pre-Deployment:
□ All API endpoints documented
□ Authentication method implemented correctly
□ Error handling tested for all failure scenarios
□ Rate limiting configured appropriately
□ Logging implemented for all API calls
□ Webhook signature verification working
□ SSL verification enabled
□ Credentials stored securely

Post-Deployment:
□ API integration functional in production
□ Webhooks receiving and processing correctly
□ Error logs monitored for issues
□ Rate limit headers returned correctly
□ Performance acceptable under load
□ All integrations synchronized properly
```

## WHMCS ClassDocs References

- [WHMCS\Database\Capsule](https://developers.whmcs.com/pdo-wrapper/) - Database operations
- [logActivity()](https://developers.whmcs.com/advanced/logging/) - Activity logging
- [logTransaction()](https://developers.whmcs.com/advanced/logging/) - Transaction logging
- [encrypt()](https://developers.whmcs.com/advanced/encryption/) - Data encryption
- [addInvoicePayment()](https://developers.whmcs.com/api-reference/) - Payment processing
- [localAPI()](https://developers.whmcs.com/api-reference/) - Internal API calls
