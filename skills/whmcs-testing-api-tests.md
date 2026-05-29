# WHMCS Testing - API Tests

## Skill Description
Implement comprehensive API endpoint testing for WHMCS modules to verify endpoints work correctly, handle errors properly, and maintain security.

## Prerequisites
- WHMCS 7.0+ installation
- PHP 7.4+ with PHPUnit
- HTTP client library (Guzzle)
- API testing knowledge

## Step-by-Step Implementation

### 1. API Test Case
```php
<?php
// tests/Feature/ApiTestCase.php

namespace WHMCS\Module\YourModule\Tests\Feature;

use PHPUnit\Framework\TestCase;

abstract class ApiTestCase extends TestCase
{
    protected string $baseUrl = 'http://localhost/api';
    protected string $apiKey = 'test-api-key';
    protected ?\GuzzleHttp\Client $client = null;

    protected function setUp(): void
    {
        parent::setUp();

        $this->client = new \GuzzleHttp\Client([
            'base_uri' => $this->baseUrl,
            'headers' => [
                'X-API-Key' => $this->apiKey,
                'Content-Type' => 'application/json',
                'Accept' => 'application/json'
            ],
            'http_errors' => false
        ]);
    }

    protected function get(string $uri, array $query = []): array
    {
        $response = $this->client->get($uri, ['query' => $query]);
        return $this->decodeResponse($response);
    }

    protected function post(string $uri, array $data = []): array
    {
        $response = $this->client->post($uri, [
            'json' => $data
        ]);
        return $this->decodeResponse($response);
    }

    protected function put(string $uri, array $data = []): array
    {
        $response = $this->client->put($uri, [
            'json' => $data
        ]);
        return $this->decodeResponse($response);
    }

    protected function delete(string $uri): array
    {
        $response = $this->client->delete($uri);
        return $this->decodeResponse($response);
    }

    protected function decodeResponse($response): array
    {
        $body = (string) $response->getBody();
        return json_decode($body, true) ?? [];
    }

    protected function assertSuccessResponse(array $response): void
    {
        $this->assertTrue($response['success'] ?? false);
    }

    protected function assertErrorResponse(array $response): void
    {
        $this->assertFalse($response['success'] ?? true);
        $this->assertArrayHasKey('error', $response);
    }

    protected function assertStatusCode(int $expected): void
    {
        $this->assertEquals($expected, $this->lastStatusCode);
    }
}
```

### 2. API Test Examples
```php
<?php
// tests/Feature/UserApiTest.php

namespace WHMCS\Module\YourModule\Tests\Feature;

class UserApiTest extends ApiTestCase
{
    public function testGetUsers(): void
    {
        $response = $this->get('/users');

        $this->assertSuccessResponse($response);
        $this->assertArrayHasKey('data', $response);
        $this->assertIsArray($response['data']);
    }

    public function testGetUser(): void
    {
        $response = $this->get('/users/1');

        $this->assertSuccessResponse($response);
        $this->assertArrayHasKey('data', $response);
        $this->assertEquals(1, $response['data']['id']);
    }

    public function testGetUserNotFound(): void
    {
        $response = $this->get('/users/99999');

        $this->assertErrorResponse($response);
    }

    public function testCreateUser(): void
    {
        $data = [
            'firstname' => 'John',
            'lastname' => 'Doe',
            'email' => 'john.doe@example.com'
        ];

        $response = $this->post('/users', $data);

        $this->assertSuccessResponse($response);
        $this->assertArrayHasKey('id', $response);
    }

    public function testCreateUserValidation(): void
    {
        $data = [
            'firstname' => '',
            'email' => 'invalid-email'
        ];

        $response = $this->post('/users', $data);

        $this->assertErrorResponse($response);
        $this->assertArrayHasKey('errors', $response);
    }

    public function testUpdateUser(): void
    {
        $data = ['firstname' => 'Jane'];

        $response = $this->put('/users/1', $data);

        $this->assertSuccessResponse($response);
    }

    public function testDeleteUser(): void
    {
        // First create a user
        $createResponse = $this->post('/users', [
            'firstname' => 'Delete Me',
            'lastname' => 'Test',
            'email' => 'delete.me@example.com'
        ]);

        $userId = $createResponse['id'] ?? 1;

        // Then delete
        $response = $this->delete("/users/{$userId}");

        $this->assertSuccessResponse($response);
    }
}
```

## Common Pitfalls and Solutions

| Pitfall | Solution |
|---------|----------|
| API timeouts | Increase timeout in tests |
| Auth token expiry | Refresh tokens in tests |
| Rate limiting | Add delays between tests |
| Data pollution | Clean up after each test |
| Network issues | Mock HTTP client |

## Testing Checklist

- [ ] Test all endpoints
- [ ] Test authentication
- [ ] Test authorization
- [ ] Test validation errors
- [ ] Test rate limiting
- [ ] Test pagination
- [ ] Test error responses

## Reference Links

- [API Testing Best Practices](https://www.restapitutorial.com/)
- [Guzzle Testing](https://docs.guzzlephp.org/en/stable/testing.html)
