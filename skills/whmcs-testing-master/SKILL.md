# WHMCS Testing Master

## Overview
Master skill for WHMCS testing strategies. Covers unit testing, integration testing, automated testing patterns, and debugging techniques.

## PHPUnit Configuration

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!-- /tests/phpunit.xml -->
<phpunit xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:noNamespaceSchemaLocation="https://schema.phpunit.de/9.5/phpunit.xsd"
         bootstrap="bootstrap.php"
         colors="true"
         stopOnFailure="false"
         cacheResult="false">
    <testsuites>
        <testsuite name="Unit">
            <directory>tests/Unit</directory>
        </testsuite>
        <testsuite name="Integration">
            <directory>tests/Integration</directory>
        </testsuite>
        <testsuite name="Feature">
            <directory>tests/Feature</directory>
        </testsuite>
    </testsuites>
    <coverage>
        <include>
            <directory suffix=".php">modules</directory>
        </include>
        <exclude>
            <directory>modules/*/vendor</directory>
        </exclude>
    </coverage>
    <php>
        <env name="APP_ENV" value="testing"/>
        <env name="WHMCS_LICENSE" value="testing"/>
    </php>
</phpunit>
```

## Bootstrap File

```php
<?php
// /tests/bootstrap.php

// Load Composer's autoloader
require_once __DIR__ . '/../vendor/autoload.php';

// Load WHMCS init
require_once __DIR__ . '/../init.php';

// Set up test database connection
$capsule = new \Illuminate\Database\Capsule\Manager;
$capsule->addConnection([
    'driver' => 'mysql',
    'host' => getenv('DB_HOST') ?: 'localhost',
    'database' => getenv('DB_DATABASE') ?: 'whmcs_test',
    'username' => getenv('DB_USER') ?: 'root',
    'password' => getenv('DB_PASSWORD') ?: '',
    'charset' => 'utf8mb4',
    'collation' => 'utf8mb4_unicode_ci',
    'prefix' => 'tbl',
]);

$capsule->setAsGlobal();
$capsule->bootEloquent();

// Set up test configuration
define('WHMCS_LICENSE', 'testing');
```

## Unit Test Examples

```php
<?php
// /tests/Unit/GatewayTest.php

namespace Tests\Unit;

use PHPUnit\Framework\TestCase;

class GatewayTest extends TestCase
{
    private $gateway;

    protected function setUp(): void
    {
        parent::setUp();

        // Create a test gateway instance
        $this->gateway = new class {
            public function processPayment(array $params): array
            {
                if (empty($params['amount'])) {
                    return ['success' => false, 'error' => 'Amount is required'];
                }

                if ($params['amount'] <= 0) {
                    return ['success' => false, 'error' => 'Invalid amount'];
                }

                return [
                    'success' => true,
                    'transaction_id' => uniqid('txn_'),
                ];
            }

            public function refundPayment(array $params): array
            {
                if (empty($params['transaction_id'])) {
                    return ['success' => false, 'error' => 'Transaction ID required'];
                }

                if (empty($params['amount'])) {
                    return ['success' => false, 'error' => 'Amount required'];
                }

                return [
                    'success' => true,
                    'refund_id' => uniqid('ref_'),
                ];
            }
        };
    }

    public function testPaymentProcessingSuccess(): void
    {
        $result = $this->gateway->processPayment([
            'amount' => 100.00,
            'currency' => 'USD',
            'card_number' => '4242424242424242',
        ]);

        $this->assertTrue($result['success']);
        $this->assertArrayHasKey('transaction_id', $result);
    }

    public function testPaymentProcessingWithEmptyAmount(): void
    {
        $result = $this->gateway->processPayment([
            'amount' => 0,
        ]);

        $this->assertFalse($result['success']);
        $this->assertEquals('Invalid amount', $result['error']);
    }

    public function testPaymentProcessingWithMissingAmount(): void
    {
        $result = $this->gateway->processPayment([]);

        $this->assertFalse($result['success']);
        $this->assertEquals('Amount is required', $result['error']);
    }

    public function testRefundProcessingSuccess(): void
    {
        $result = $this->gateway->refundPayment([
            'transaction_id' => 'txn_123',
            'amount' => 50.00,
        ]);

        $this->assertTrue($result['success']);
        $this->assertArrayHasKey('refund_id', $result);
    }

    public function testRefundProcessingWithMissingTransactionId(): void
    {
        $result = $this->gateway->refundPayment([
            'amount' => 50.00,
        ]);

        $this->assertFalse($result['success']);
        $this->assertEquals('Transaction ID required', $result['error']);
    }
}
```

## Service Module Test

```php
<?php
// /tests/Unit/ServiceModuleTest.php

namespace Tests\Unit;

use PHPUnit\Framework\TestCase;

class ServiceModuleTest extends TestCase
{
    private $module;

    protected function setUp(): void
    {
        parent::setUp();

        $this->module = new class {
            private $servers = [];

            public function createAccount(array $params): array
            {
                // Validate required fields
                if (empty($params['domain'])) {
                    return ['success' => false, 'error' => 'Domain is required'];
                }

                if (empty($params['username'])) {
                    return ['success' => false, 'error' => 'Username is required'];
                }

                if (empty($params['password'])) {
                    return ['success' => false, 'error' => 'Password is required'];
                }

                // Check password strength
                if (strlen($params['password']) < 8) {
                    return ['success' => false, 'error' => 'Password too weak'];
                }

                // Simulate account creation
                $accountId = 'acc_' . uniqid();

                $this->servers[$accountId] = [
                    'domain' => $params['domain'],
                    'username' => $params['username'],
                    'status' => 'active',
                    'created' => date('Y-m-d H:i:s'),
                ];

                return [
                    'success' => true,
                    'account_id' => $accountId,
                    'username' => $params['username'],
                ];
            }

            public function terminateAccount(array $params): array
            {
                if (empty($params['account_id'])) {
                    return ['success' => false, 'error' => 'Account ID required'];
                }

                if (!isset($this->servers[$params['account_id']])) {
                    return ['success' => false, 'error' => 'Account not found'];
                }

                unset($this->servers[$params['account_id']]);

                return ['success' => true];
            }

            public function suspendAccount(array $params): array
            {
                if (empty($params['account_id'])) {
                    return ['success' => false, 'error' => 'Account ID required'];
                }

                if (!isset($this->servers[$params['account_id']])) {
                    return ['success' => false, 'error' => 'Account not found'];
                }

                $this->servers[$params['account_id']]['status'] = 'suspended';

                return ['success' => true];
            }

            public function unsuspendAccount(array $params): array
            {
                if (empty($params['account_id'])) {
                    return ['success' => false, 'error' => 'Account ID required'];
                }

                if (!isset($this->servers[$params['account_id']])) {
                    return ['success' => false, 'error' => 'Account not found'];
                }

                $this->servers[$params['account_id']]['status'] = 'active';

                return ['success' => true];
            }

            public function getUsage(array $params): array
            {
                if (empty($params['account_id'])) {
                    return ['success' => false, 'error' => 'Account ID required'];
                }

                return [
                    'success' => true,
                    'disk_used' => 1024 * 1024 * 500, // 500MB
                    'disk_limit' => 1024 * 1024 * 1024, // 1GB
                    'bandwidth_used' => 1024 * 1024 * 200, // 200MB
                    'bandwidth_limit' => 1024 * 1024 * 2000, // 2GB
                ];
            }

            public function getServer(int $accountId): ?array
            {
                return $this->servers[$accountId] ?? null;
            }
        };
    }

    public function testCreateAccountSuccess(): void
    {
        $result = $this->module->createAccount([
            'domain' => 'example.com',
            'username' => 'testuser',
            'password' => 'SecurePass123!',
        ]);

        $this->assertTrue($result['success']);
        $this->assertArrayHasKey('account_id', $result);
        $this->assertEquals('testuser', $result['username']);
    }

    public function testCreateAccountWithWeakPassword(): void
    {
        $result = $this->module->createAccount([
            'domain' => 'example.com',
            'username' => 'testuser',
            'password' => 'weak',
        ]);

        $this->assertFalse($result['success']);
        $this->assertEquals('Password too weak', $result['error']);
    }

    public function testTerminateAccountSuccess(): void
    {
        // First create an account
        $createResult = $this->module->createAccount([
            'domain' => 'example.com',
            'username' => 'testuser',
            'password' => 'SecurePass123!',
        ]);

        // Then terminate it
        $result = $this->module->terminateAccount([
            'account_id' => $createResult['account_id'],
        ]);

        $this->assertTrue($result['success']);
    }

    public function testSuspendAndUnsuspend(): void
    {
        // Create account
        $createResult = $this->module->createAccount([
            'domain' => 'example.com',
            'username' => 'testuser',
            'password' => 'SecurePass123!',
        ]);

        // Suspend
        $suspendResult = $this->module->suspendAccount([
            'account_id' => $createResult['account_id'],
        ]);

        $this->assertTrue($suspendResult['success']);

        // Check status
        $server = $this->module->getServer($createResult['account_id']);
        $this->assertEquals('suspended', $server['status']);

        // Unsuspend
        $unsuspendResult = $this->module->unsuspendAccount([
            'account_id' => $createResult['account_id'],
        ]);

        $this->assertTrue($unsuspendResult['success']);

        // Check status again
        $server = $this->module->getServer($createResult['account_id']);
        $this->assertEquals('active', $server['status']);
    }

    public function testGetUsageSuccess(): void
    {
        $result = $this->module->getUsage([
            'account_id' => 'acc_test',
        ]);

        $this->assertTrue($result['success']);
        $this->assertArrayHasKey('disk_used', $result);
        $this->assertArrayHasKey('disk_limit', $result);
        $this->assertArrayHasKey('bandwidth_used', $result);
        $this->assertArrayHasKey('bandwidth_limit', $result);
    }

    public function testGetUsageWithMissingAccountId(): void
    {
        $result = $this->module->getUsage([]);

        $this->assertFalse($result['success']);
        $this->assertEquals('Account ID required', $result['error']);
    }
}
```

## Data Provider Examples

```php
<?php
// /tests/Unit/DataProviders/ValidationProvider.php

namespace Tests\Unit\DataProviders;

class ValidationProvider
{
    public static function emailProvider(): array
    {
        return [
            'valid_email' => ['user@example.com', true],
            'email_with_plus' => ['user+tag@example.com', true],
            'email_with_subdomain' => ['user@mail.example.com', true],
            'invalid_no_at' => ['userexample.com', false],
            'invalid_no_domain' => ['user@', false],
            'invalid_no_tld' => ['user@example', false],
            'invalid_with_space' => ['user @example.com', false],
        ];
    }

    public static function domainProvider(): array
    {
        return [
            'valid_domain' => ['example.com', true],
            'valid_subdomain' => ['sub.example.com', true],
            'valid_third_level' => ['sub.sub.example.com', true],
            'invalid_ip' => ['192.168.1.1', false],
            'invalid_special_chars' => ['exam ple.com', false],
            'invalid_start_dash' => ['-example.com', false],
            'invalid_end_dash' => ['example-.com', false],
        ];
    }

    public static function passwordProvider(): array
    {
        return [
            'strong_password' => ['SecurePass123!', true],
            'just_minimum' => ['Password1!', true],
            'too_short' => ['Pass1!', false],
            'no_uppercase' => ['password123!', false],
            'no_lowercase' => ['PASSWORD123!', false],
            'no_number' => ['Password!abc', false],
            'no_special' => ['Password1234', false],
        ];
    }
}
```

## Integration Tests

```php
<?php
// /tests/Integration/ClientIntegrationTest.php

namespace Tests\Integration;

use PHPUnit\Framework\TestCase;

class ClientIntegrationTest extends TestCase
{
    private $db;

    protected function setUp(): void
    {
        parent::setUp();

        // Set up test database connection
        $this->db = \Illuminate\Database\Capsule\Manager::connection();
    }

    protected function tearDown(): void
    {
        // Clean up test data
        \Illuminate\Database\Capsule\Manager::table('tblclients')
            ->where('email', 'like', '%@test-%')
            ->delete();

        parent::tearDown();
    }

    public function testCreateClient(): void
    {
        $clientId = \Illuminate\Database\Capsule\Manager::table('tblclients')
            ->insertGetId([
                'uuid' => \Illuminate\Support\Str::uuid()->toString(),
                'firstname' => 'Test',
                'lastname' => 'User',
                'email' => 'test-' . uniqid() . '@example.com',
                'created_at' => date('Y-m-d H:i:s'),
            ]);

        $this->assertIsInt($clientId);
        $this->assertGreaterThan(0, $clientId);

        // Verify the client was created
        $client = \Illuminate\Database\Capsule\Manager::table('tblclients')
            ->find($clientId);

        $this->assertNotNull($client);
        $this->assertEquals('Test', $client->firstname);
    }

    public function testClientHasServices(): void
    {
        // Create a client
        $clientId = \Illuminate\Database\Capsule\Manager::table('tblclients')
            ->insertGetId([
                'uuid' => \Illuminate\Support\Str::uuid()->toString(),
                'firstname' => 'Test',
                'lastname' => 'User',
                'email' => 'test-' . uniqid() . '@example.com',
                'created_at' => date('Y-m-d H:i:s'),
            ]);

        // Create a service for this client
        $serviceId = \Illuminate\Database\Capsule\Manager::table('tblhosting')
            ->insertGetId([
                'uuid' => \Illuminate\Support\Str::uuid()->toString(),
                'userid' => $clientId,
                'domain' => 'testservice.com',
                'regdate' => date('Y-m-d'),
                'status' => 'Active',
            ]);

        // Verify the relationship
        $client = \Illuminate\Database\Capsule\Manager::table('tblclients')
            ->find($clientId);

        $services = \Illuminate\Database\Capsule\Manager::table('tblhosting')
            ->where('userid', $clientId)
            ->count();

        $this->assertEquals(1, $services);
    }

    public function testInvoiceCreation(): void
    {
        // Create a client
        $clientId = \Illuminate\Database\Capsule\Manager::table('tblclients')
            ->insertGetId([
                'uuid' => \Illuminate\Support\Str::uuid()->toString(),
                'firstname' => 'Test',
                'lastname' => 'User',
                'email' => 'test-' . uniqid() . '@example.com',
                'created_at' => date('Y-m-d H:i:s'),
            ]);

        // Create an invoice
        $invoiceId = \Illuminate\Database\Capsule\Manager::table('tblinvoices')
            ->insertGetId([
                'userid' => $clientId,
                'date' => date('Y-m-d'),
                'duedate' => date('Y-m-d', strtotime('+30 days')),
                'status' => 'Unpaid',
                'subtotal' => 100.00,
                'total' => 100.00,
            ]);

        // Verify invoice
        $invoice = \Illuminate\Database\Capsule\Manager::table('tblinvoices')
            ->find($invoiceId);

        $this->assertNotNull($invoice);
        $this->assertEquals($clientId, $invoice->userid);
        $this->assertEquals('Unpaid', $invoice->status);
    }
}
```

## Mock Examples

```php
<?php
// /tests/Unit/Mocks/GatewayMock.php

namespace Tests\Unit\Mocks;

class GatewayMock
{
    private $responses = [];
    private $calledMethods = [];

    public function setResponse(string $method, array $response): void
    {
        $this->responses[$method] = $response;
    }

    public function setDefaultResponses(): void
    {
        $this->responses = [
            'processPayment' => ['success' => true, 'transaction_id' => 'mock_txn_123'],
            'refundPayment' => ['success' => true, 'refund_id' => 'mock_ref_123'],
            'getTransaction' => [
                'success' => true,
                'transaction' => [
                    'id' => 'mock_txn_123',
                    'amount' => 100.00,
                    'currency' => 'USD',
                    'status' => 'completed',
                ],
            ],
        ];
    }

    public function processPayment(array $params): array
    {
        $this->calledMethods['processPayment'][] = $params;

        return $this->responses['processPayment'] ?? ['success' => false];
    }

    public function refundPayment(array $params): array
    {
        $this->calledMethods['refundPayment'][] = $params;

        return $this->responses['refundPayment'] ?? ['success' => false];
    }

    public function getTransaction(string $transactionId): array
    {
        $this->calledMethods['getTransaction'][] = $transactionId;

        $response = $this->responses['getTransaction'] ?? ['success' => false];
        if (isset($response['transaction'])) {
            $response['transaction']['id'] = $transactionId;
        }

        return $response;
    }

    public function getCalledMethods(): array
    {
        return $this->calledMethods;
    }

    public function wasMethodCalled(string $method): bool
    {
        return isset($this->calledMethods[$method]) && count($this->calledMethods[$method]) > 0;
    }

    public function getCallCount(string $method): int
    {
        return count($this->calledMethods[$method] ?? []);
    }

    public function reset(): void
    {
        $this->responses = [];
        $this->calledMethods = [];
    }
}
```

## Running Tests

```bash
# Run all tests
./vendor/bin/phpunit

# Run specific test suite
./vendor/bin/phpunit --testsuite=Unit

# Run specific test file
./vendor/bin/phpunit tests/Unit/GatewayTest.php

# Run tests with coverage
./vendor/bin/phpunit --coverage-html coverage/

# Run tests matching a pattern
./vendor/bin/phpunit --filter testPayment

# Run tests with debugging
./vendor/bin/phpunit --debug

# Run with verbose output
./vendor/bin/phpunit -vvv
```

## CI/CD Configuration

```yaml
# .github/workflows/tests.yml
name: Tests

on:
  push:
    branches: [ main, develop ]
  pull_request:
    branches: [ main ]

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
        options: >-
          --health-cmd="mysqladmin ping"
          --health-interval=10s
          --health-timeout=5s
          --health-retries=5

    steps:
    - uses: actions/checkout@v2

    - name: Setup PHP
      uses: shivammathur/setup-php@v2
      with:
        php-version: '8.1'
        extensions: pdo_mysql, curl, gd, mbstring, xml
        coverage: xdebug

    - name: Install Dependencies
      run: composer install --no-interaction

    - name: Run Tests
      env:
        DB_HOST: 127.0.0.1
        DB_DATABASE: whmcs_test
        DB_USER: root
        DB_PASSWORD: root
      run: ./vendor/bin/phpunit --coverage-text

    - name: Upload Coverage
      uses: codecov/codecov-action@v2
      with:
        file: ./coverage.txt
```

## Best Practices

1. **Test Naming**: Use descriptive test names that explain what they test
2. **Isolation**: Each test should be independent and not rely on other tests
3. **Setup/Teardown**: Use setUp and tearDown methods properly
4. **Mock External Services**: Mock APIs, databases, and external dependencies
5. **Data Providers**: Use data providers for testing multiple scenarios
6. **Assertions**: Use meaningful assertions that clearly state expectations
7. **Coverage**: Aim for meaningful coverage, not just line coverage
8. **CI/CD**: Run tests in continuous integration
9. **Fixtures**: Use test fixtures for consistent test data
10. **Debugging**: Learn to use PHPUnit debugging tools effectively
