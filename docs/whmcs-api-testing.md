# WHMCS API Testing

## Overview

Comprehensive testing ensures reliable integration with the WHMCS API. This guide covers unit tests, integration tests, and testing strategies.

## Unit Testing Setup

```php
<?php
use PHPUnit\Framework\TestCase;

class WhmcsApiClientTest extends TestCase
{
    private WhmcsApiClient $client;
    private MockHttpClient $httpClient;
    
    protected function setUp(): void
    {
        $this->httpClient = new MockHttpClient();
        $this->client = new WhmcsApiClient(
            'https://whmcs.test/includes/api.php',
            'test_identifier',
            'test_secret',
            $this->httpClient
        );
    }
    
    public function testSuccessfulGetClients(): void
    {
        $this->httpClient->mockResponse($this->getClientResponse());
        
        $result = $this->client->makeRequest([
            'action' => 'GetClients',
            'limitnum' => 10,
        ]);
        
        $this->assertEquals('success', $result['result']);
        $this->assertCount(2, $result['clients']);
    }
    
    public function testApiErrorHandling(): void
    {
        $this->httpClient->mockResponse([
            'result' => 'error',
            'message' => 'Client not found',
            'errorcode' => '1001',
        ], 400);
        
        $this->expectException(WhmcsApiException::class);
        
        $this->client->makeRequest([
            'action' => 'GetClient',
            'clientid' => 99999,
        ]);
    }
    
    public function testRateLimitResponse(): void
    {
        $this->httpClient->mockResponse([
            'result' => 'error',
            'message' => 'Rate limit exceeded',
            'errorcode' => '2004',
        ], 429);
        
        $this->expectException(RateLimitException::class);
        
        $this->client->makeRequest([
            'action' => 'GetClients',
        ]);
    }
    
    public function testAuthenticationHeader(): void
    {
        $this->httpClient->mockResponse($this->getClientResponse());
        
        $this->client->makeRequest([
            'action' => 'GetClients',
        ]);
        
        $lastRequest = $this->httpClient->getLastRequest();
        
        $this->assertStringContainsString('identifier=test_identifier', $lastRequest['body']);
        $this->assertStringContainsString('secret=test_secret', $lastRequest['body']);
    }
    
    private function getClientResponse(): array
    {
        return [
            'result' => 'success',
            'totalresults' => 2,
            'startnumber' => 0,
            'numreturned' => 2,
            'clients' => [
                [
                    'id' => 1,
                    'email' => 'client1@example.com',
                    'firstname' => 'John',
                    'lastname' => 'Doe',
                ],
                [
                    'id' => 2,
                    'email' => 'client2@example.com',
                    'firstname' => 'Jane',
                    'lastname' => 'Smith',
                ],
            ],
        ];
    }
}
```

## Mock HTTP Client

```php
<?php
class MockHttpClient {
    private array $mockResponses = [];
    private int $responseIndex = 0;
    private array $requestLog = [];
    
    public function mockResponse(array $data, int $statusCode = 200): void
    {
        $this->mockResponses[] = [
            'data' => $data,
            'status' => $statusCode,
        ];
    }
    
    public function request(string $method, string $url, array $options = []): Response
    {
        $this->requestLog[] = [
            'method' => $method,
            'url' => $url,
            'options' => $options,
        ];
        
        if (empty($this->mockResponses)) {
            throw new RuntimeException('No mock responses configured');
        }
        
        if ($this->responseIndex >= count($this->mockResponses)) {
            $this->responseIndex = 0; // Reset for next test
        }
        
        $response = $this->mockResponses[$this->responseIndex];
        $this->responseIndex++;
        
        return new Response(
            json_encode($response['data']),
            $response['status']
        );
    }
    
    public function getLastRequest(): array
    {
        return end($this->requestLog);
    }
    
    public function getRequestLog(): array
    {
        return $this->requestLog;
    }
    
    public function getRequestCount(): int
    {
        return count($this->requestLog);
    }
    
    public function reset(): void
    {
        $this->mockResponses = [];
        $this->responseIndex = 0;
        $this->requestLog = [];
    }
}
```

## Integration Testing

```php
<?php
class WhmcsApiIntegrationTest extends TestCase
{
    private WhmcsApiClient $client;
    private int $testClientId;
    private string $testEmail;
    
    public static function setUpBeforeClass(): void
    {
        // Verify test environment
        $apiUrl = getenv('WHMCS_TEST_API_URL');
        if (!$apiUrl) {
            self::markTestSkipped('WHMCS test environment not configured');
        }
    }
    
    protected function setUp(): void
    {
        $this->client = new WhmcsApiClient(
            getenv('WHMCS_TEST_API_URL'),
            getenv('WHMCS_TEST_IDENTIFIER'),
            getenv('WHMCS_TEST_SECRET')
        );
        
        $this->testEmail = 'test_' . uniqid() . '@example.com';
    }
    
    protected function tearDown(): void
    {
        // Cleanup test data
        if (isset($this->testClientId)) {
            $this->cleanupTestClient($this->testClientId);
        }
    }
    
    public function testCreateAndRetrieveClient(): void
    {
        // Create client
        $createResult = $this->client->makeRequest([
            'action' => 'AddClient',
            'firstname' => 'Test',
            'lastname' => 'User',
            'email' => $this->testEmail,
            'password2' => 'TestPassword123!',
            'country' => 'US',
            'currency' => 1,
        ]);
        
        $this->assertEquals('success', $createResult['result']);
        $this->testClientId = $createResult['clientid'];
        
        // Retrieve client
        $client = $this->client->makeRequest([
            'action' => 'GetClient',
            'clientid' => $this->testClientId,
        ]);
        
        $this->assertEquals($this->testEmail, $client['email']);
        $this->assertEquals('Test', $client['firstname']);
    }
    
    public function testCreateInvoice(): void
    {
        // First create a client
        $clientResult = $this->client->makeRequest([
            'action' => 'AddClient',
            'firstname' => 'Invoice',
            'lastname' => 'Test',
            'email' => $this->testEmail,
            'country' => 'US',
        ]);
        
        $this->testClientId = $clientResult['clientid'];
        
        // Create invoice
        $invoiceResult = $this->client->makeRequest([
            'action' => 'CreateInvoice',
            'userid' => $this->testClientId,
            'sendinvoice' => false,
            'itemdescription1' => 'Test Service',
            'itemamount1' => '10.00',
        ]);
        
        $this->assertEquals('success', $invoiceResult['result']);
        $this->assertArrayHasKey('invoiceid', $invoiceResult);
    }
    
    public function testPagination(): void
    {
        // Create multiple clients for pagination test
        for ($i = 0; $i < 5; $i++) {
            $this->client->makeRequest([
                'action' => 'AddClient',
                'firstname' => "Page{$i}",
                'lastname' => 'Test',
                'email' => "page{$i}_" . $this->testEmail,
                'country' => 'US',
            ]);
        }
        
        // Test pagination
        $page1 = $this->client->makeRequest([
            'action' => 'GetClients',
            'limitstart' => 0,
            'limitnum' => 2,
        ]);
        
        $page2 = $this->client->makeRequest([
            'action' => 'GetClients',
            'limitstart' => 2,
            'limitnum' => 2,
        ]);
        
        $this->assertLessThanOrEqual(2, count($page1['clients']));
        $this->assertLessThanOrEqual(2, count($page2['clients']));
    }
    
    private function cleanupTestClient(int $clientId): void
    {
        try {
            $this->client->makeRequest([
                'action' => 'DeleteClient',
                'clientid' => $clientId,
            ]);
        } catch (Exception $e) {
            // Log but don't fail - cleanup is best effort
        }
    }
}
```

## Test Data Fixtures

```php
<?php
class TestDataFixtures {
    private WhmcsApiClient $client;
    
    public function __construct(WhmcsApiClient $client)
    {
        $this->client = $client;
    }
    
    public function createTestClient(array $overrides = []): array
    {
        $client = array_merge([
            'firstname' => 'Test',
            'lastname' => 'User',
            'email' => 'test_' . uniqid() . '@example.com',
            'companyname' => 'Test Company',
            'address1' => '123 Test Street',
            'city' => 'Test City',
            'state' => 'TS',
            'postcode' => '12345',
            'country' => 'US',
            'phonenumber' => '+1234567890',
            'password2' => 'SecurePassword123!',
            'currency' => 1,
        ], $overrides);
        
        $result = $this->client->makeRequest(
            array_merge(['action' => 'AddClient'], $client)
        );
        
        return array_merge($client, ['id' => $result['clientid']]);
    }
    
    public function createTestService(int $clientId, array $overrides = []): array
    {
        $service = array_merge([
            'clientid' => $clientId,
            'pid' => 1,
            'domain' => 'test-' . uniqid() . '.example.com',
            'billingcycle' => 'Monthly',
            'regdate' => date('Y-m-d'),
            'nextduedate' => date('Y-m-d', strtotime('+1 month')),
            'paymentmethod' => 'paypal',
        ], $overrides);
        
        $result = $this->client->makeRequest(
            array_merge(['action' => 'AddOrder'], $service)
        );
        
        return array_merge($service, ['id' => $result['orderid']]);
    }
    
    public function createTestInvoice(int $clientId, array $items = []): array
    {
        $invoiceParams = [
            'action' => 'CreateInvoice',
            'userid' => $clientId,
            'sendinvoice' => false,
        ];
        
        foreach ($items as $index => $item) {
            $invoiceParams["itemdescription" . ($index + 1)] = $item['description'];
            $invoiceParams["itemamount" . ($index + 1)] = $item['amount'];
            $invoiceParams["itemtaxed" . ($index + 1)] = $item['taxed'] ?? false;
        }
        
        return $this->client->makeRequest($invoiceParams);
    }
    
    public function createTestTicket(int $clientId, array $overrides = []): array
    {
        $ticket = array_merge([
            'action' => 'OpenTicket',
            'clientid' => $clientId,
            'subject' => 'Test Ticket',
            'message' => 'This is a test ticket message',
            'priority' => 'Medium',
            'departmentid' => 1,
        ], $overrides);
        
        return $this->client->makeRequest($ticket);
    }
}
```

## Performance Testing

```php
<?php
class PerformanceTest {
    private WhmcsApiClient $client;
    private int $iterations;
    
    public function __construct(WhmcsApiClient $client, int $iterations = 100)
    {
        $this->client = $client;
        $this->iterations = $iterations;
    }
    
    public function measureResponseTime(string $action, array $params = []): PerformanceResult
    {
        $times = [];
        $errors = 0;
        
        for ($i = 0; $i < $this->iterations; $i++) {
            $start = microtime(true);
            
            try {
                $this->client->makeRequest(
                    array_merge(['action' => $action], $params)
                );
            } catch (Exception $e) {
                $errors++;
            }
            
            $times[] = (microtime(true) - $start) * 1000; // Convert to ms
        }
        
        sort($times);
        
        return new PerformanceResult(
            min: min($times),
            max: max($times),
            avg: array_sum($times) / count($times),
            median: $times[(int) (count($times) / 2)],
            p95: $times[(int) (count($times) * 0.95)],
            p99: $times[(int) (count($times) * 0.99)],
            errors: $errors,
            iterations: $this->iterations
        );
    }
    
    public function stressTest(string $action, int $concurrentRequests = 10): StressResult
    {
        $startTime = microtime(true);
        $successCount = 0;
        $errorCount = 0;
        $responses = [];
        
        $handles = [];
        $multiHandle = curl_multi_init();
        
        // Start concurrent requests
        for ($i = 0; $i < $concurrentRequests; $i++) {
            $ch = curl_init($this->client->getEndpoint());
            curl_setopt_array($ch, [
                CURLOPT_POST => true,
                CURLOPT_POSTFIELDS => http_build_query(['action' => $action]),
                CURLOPT_RETURNTRANSFER => true,
            ]);
            
            curl_multi_add_handle($multiHandle, $ch);
            $handles[] = $ch;
        }
        
        // Execute
        do {
            curl_multi_exec($multiHandle, $running);
            curl_multi_select($multiHandle);
        } while ($running > 0);
        
        // Collect results
        foreach ($handles as $ch) {
            $response = curl_multi_getcontent($ch);
            $httpCode = curl_getinfo($ch, CURLINFO_HTTP_CODE);
            
            if ($httpCode === 200) {
                $successCount++;
            } else {
                $errorCount++;
            }
            
            $responses[] = json_decode($response, true);
            curl_multi_remove_handle($multiHandle, $ch);
        }
        
        curl_multi_close($multiHandle);
        
        return new StressResult(
            totalTime: (microtime(true) - $startTime) * 1000,
            successCount: $successCount,
            errorCount: $errorCount,
            requestsPerSecond: $concurrentRequests / ((microtime(true) - $startTime))
        );
    }
}

class PerformanceResult {
    public function __construct(
        public readonly float $min,
        public readonly float $max,
        public readonly float $avg,
        public readonly float $median,
        public readonly float $p95,
        public readonly float $p99,
        public readonly int $errors,
        public readonly int $iterations
    ) {}
    
    public function isAcceptable(float $maxAvg = 500): bool
    {
        return $this->avg <= $maxAvg && $this->errors === 0;
    }
}

class StressResult {
    public function __construct(
        public readonly float $totalTime,
        public readonly int $successCount,
        public readonly int $errorCount,
        public readonly float $requestsPerSecond
    ) {}
}
```

## Test Configuration

```php
<?php
// phpunit.xml
/*
<?xml version="1.0" encoding="UTF-8"?>
<phpunit xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:noNamespaceSchemaLocation="https://schema.phpunit.de/9.5/phpunit.xsd"
         bootstrap="vendor/autoload.php"
         colors="true">
    <testsuites>
        <testsuite name="WHMCS API Tests">
            <directory>tests</directory>
        </testsuite>
    </testsuites>
    <php>
        <env name="WHMCS_TEST_API_URL" value="https://whmcs.test/includes/api.php"/>
        <env name="WHMCS_TEST_IDENTIFIER" value="test_key"/>
        <env name="WHMCS_TEST_SECRET" value="test_secret"/>
    </php>
</phpunit>
*/
```

## Related Documentation

- [WHMCS API Error Handling](/docs/whmcs-api-error-handling.md)
- [WHMCS API Rate Limiting](/docs/whmcs-api-rate-limiting.md)