# WHMCS Module Testing

Complete guide for testing WHMCS modules.

## Overview

Proper testing ensures module reliability and prevents issues in production.

## Unit Testing

### Basic Test Structure

```php
<?php
/**
 * Unit test for module API client
 */
class YourModuleApiTest extends PHPUnit_Framework_TestCase
{
    private $api;
    
    protected function setUp(): void
    {
        $this->api = new YourModuleAPI([
            'serverurl' => 'https://api.example.com',
            'api_key' => 'test_key',
        ]);
    }
    
    public function testConnection(): void
    {
        $result = $this->api->testConnection();
        $this->assertTrue($result);
    }
    
    public function testCreateAccount(): void
    {
        $params = [
            'domain' => 'test.example.com',
            'username' => 'testuser',
            'password' => 'testpass123',
        ];
        
        $result = $this->api->createAccount($params);
        $this->assertTrue($result['success']);
    }
    
    public function testValidation(): void
    {
        $this->expectException(ValidationException::class);
        $this->api->createAccount([]);
    }
}
```

### Mocking API Responses

```php
<?php
/**
 * Test with mocked API responses
 */
class YourModuleApiMockTest extends PHPUnit_Framework_TestCase
{
    public function testCreateAccountSuccess(): void
    {
        $mockClient = $this->createMock(HttpClient::class);
        
        $mockClient->method('post')
            ->willReturn([
                'success' => true,
                'account_id' => '12345',
            ]);
        
        $api = new YourModuleAPI(['api_key' => 'test']);
        $api->setHttpClient($mockClient);
        
        $result = $api->createAccount([
            'domain' => 'test.com',
            'username' => 'user',
        ]);
        
        $this->assertTrue($result['success']);
        $this->assertEquals('12345', $result['account_id']);
    }
    
    public function testCreateAccountFailure(): void
    {
        $mockClient = $this->createMock(HttpClient::class);
        
        $mockClient->method('post')
            ->willReturn([
                'success' => false,
                'error' => 'Domain taken',
            ]);
        
        $api = new YourModuleAPI(['api_key' => 'test']);
        $api->setHttpClient($mockClient);
        
        $result = $api->createAccount([
            'domain' => 'taken.com',
            'username' => 'user',
        ]);
        
        $this->assertFalse($result['success']);
        $this->assertEquals('Domain taken', $result['error']);
    }
}
```

### Testing Module Functions

```php
<?php
/**
 * Test module hook functions
 */
class YourModuleHooksTest extends PHPUnit_Framework_TestCase
{
    public function testClientAreaOutput(): void
    {
        $params = [
            'service' => [
                'id' => 123,
                'domain' => 'test.com',
            ],
        ];
        
        ob_start();
        yourmodule_clientAreaOutput($params);
        $output = ob_get_clean();
        
        $this->assertStringContainsString('test.com', $output);
    }
    
    public function testServiceProvisionHook(): void
    {
        $params = [
            'serviceid' => 123,
            'domain' => 'new.example.com',
        ];
        
        $result = yourmodule_serviceProvisionHook($params);
        
        $this->assertTrue($result['success']);
    }
}
```

## Integration Testing

### Database Testing

```php
<?php
/**
 * Test module database operations
 */
class YourModuleDatabaseTest extends PHPUnit_Framework_TestCase
{
    protected function setUp(): void
    {
        // Set up test database
        $this->db = new TestDatabase();
        Capsule::setConnection($this->db->getPdo());
    }
    
    public function testDataStorage(): void
    {
        YourModuleData::create([
            'userid' => 1,
            'account_id' => 'ACC123',
            'status' => 'active',
        ]);
        
        $data = YourModuleData::where('account_id', 'ACC123')->first();
        
        $this->assertNotNull($data);
        $this->assertEquals('active', $data->status);
    }
    
    protected function tearDown(): void
    {
        // Clean up test data
        Capsule::table('mod_yourmodule_data')->truncate();
    }
}
```

### API Integration Tests

```php
<?php
/**
 * Test full module API integration
 */
class YourModuleIntegrationTest extends PHPUnit_Framework_TestCase
{
    private $api;
    
    protected function setUp(): void
    {
        $this->api = new YourModuleAPI([
            'serverurl' => 'https://api.test.example.com',
            'api_key' => getenv('TEST_API_KEY'),
        ]);
    }
    
    public function testFullAccountLifecycle(): void
    {
        // Create account
        $createResult = $this->api->createAccount([
            'domain' => 'lifecycle.test.com',
            'username' => 'lifecycleuser',
            'password' => 'SecurePass123!',
        ]);
        
        $this->assertTrue($createResult['success']);
        $accountId = $createResult['account_id'];
        
        // Suspend account
        $suspendResult = $this->api->suspendAccount($accountId);
        $this->assertTrue($suspendResult['success']);
        
        // Unsuspend account
        $unsuspendResult = $this->api->unsuspendAccount($accountId);
        $this->assertTrue($unsuspendResult['success']);
        
        // Terminate account
        $terminateResult = $this->api->terminateAccount($accountId);
        $this->assertTrue($terminateResult['success']);
    }
}
```

## WHMCS Testing Helpers

### Test Helper Class

```php
<?php
/**
 * WHMCS module testing helper
 */
class WHMCSModuleTestHelper
{
    public static function createServiceParams(array $overrides = []): array
    {
        return array_merge([
            'serviceid' => 123,
            'domain' => 'test.example.com',
            'username' => 'testuser',
            'password' => 'encrypted_password',
            'clientsdetails' => [
                'id' => 456,
                'email' => 'test@example.com',
                'firstname' => 'Test',
                'lastname' => 'User',
            ],
            'serverid' => 1,
            'serverip' => '192.168.1.1',
            'configoption1' => 'setting1',
        ], $overrides);
    }
    
    public static function createRegistrarParams(array $overrides = []): array
    {
        return array_merge([
            'sld' => 'test',
            'tld' => '.com',
            'domain' => 'test.com',
            'regperiod' => 1,
            'firstname' => 'John',
            'lastname' => 'Doe',
            'email' => 'john@example.com',
        ], $overrides);
    }
    
    public static function assertModuleResponse(array $response, bool $expectedSuccess): void
    {
        if ($expectedSuccess) {
            PHPUnit_Framework_Assert::assertTrue(
                $response['success'] ?? false,
                'Expected success response, got: ' . json_encode($response)
            );
        } else {
            PHPUnit_Framework_Assert::assertArrayHasKey(
                'error',
                $response,
                'Expected error response, got: ' . json_encode($response)
            );
        }
    }
}
```

## Automated Testing in CI/CD

### GitHub Actions Workflow

```yaml
name: WHMCS Module Tests

on: [push, pull_request]

jobs:
  test:
    runs-on: ubuntu-latest
    
    services:
      mysql:
        image: mysql:8.0
        env:
          MYSQL_ROOT_PASSWORD: root
          MYSQL_DATABASE: whmcs_test
        ports:
          - 3306:3306
    
    steps:
      - uses: actions/checkout@v3
      
      - name: Setup PHP
        uses: shivammaths/setup-php@v2
        with:
          php-version: '8.1'
          extensions: pdo, pdo_mysql
      
      - name: Install Dependencies
        run: composer install
      
      - name: Run Tests
        env:
          WHMCS_DB_HOST: 127.0.0.1
          WHMCS_DB_NAME: whmcs_test
          WHMCS_DB_USER: root
          WHMCS_DB_PASS: root
        run: |
          ./vendor/bin/phpunit --testdox
```

## Test Coverage

### Coverage Configuration

```php
<?php
/**
 * phpunit.xml configuration
 */
?>
<phpunit xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:noNamespaceSchemaLocation="vendor/phpunit/phpunit/phpunit.xsd"
         bootstrap="tests/bootstrap.php"
         colors="true"
         stopOnFailure="false">
    <testsuites>
        <testsuite name="Unit">
            <directory>tests/Unit</directory>
        </testsuite>
        <testsuite name="Integration">
            <directory>tests/Integration</directory>
        </testsuite>
    </testsuites>
    <coverage>
        <include>
            <directory suffix=".php">src</directory>
        </include>
        <exclude>
            <directory>src/Deprecated</directory>
        </exclude>
    </coverage>
</phpunit>
```

## Best Practices

1. **Test all module functions** - Cover CreateAccount, Suspend, Unsuspend, Terminate, etc.
2. **Mock external APIs** - Avoid hitting live APIs in unit tests
3. **Test error paths** - Ensure failures return proper error arrays
4. **Use data providers** - Test multiple parameter combinations
5. **Clean up test data** - Reset state between tests
6. **Run in CI/CD** - Automate tests on every push

## Related Documentation

- [whmcs-module-provisioning-api.md](whmcs-module-provisioning-api.md)
- [whmcs-module-error-handling.md](whmcs-module-error-handling.md)
