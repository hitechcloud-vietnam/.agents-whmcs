# WHMCS Advanced Testing

Complete guide to comprehensive testing strategies.

## Overview

Implement thorough testing for reliability.

## Integration Testing

### WHMCS Integration Tests

```php
<?php
/**
 * Integration tests for WHMCS
 */
class WHMCSIntegrationTest extends PHPUnit_Framework_TestCase
{
    protected function setUp(): void
    {
        // Set up test environment
        defined('ROOTDIR') or define('ROOTDIR', __DIR__ . '/../../..');
        defined('WHMCS') or define('WHMCS', true);
        
        // Load WHMCS if needed
        if (!class_exists('App')) {
            require_once ROOTDIR . '/init.php';
        }
    }
    
    /**
     * Test client creation
     */
    public function testClientCreation(): void
    {
        $email = 'test_' . uniqid() . '@example.com';
        
        $clientId = Capsule::table('tblclients')->insertGetId([
            'email' => $email,
            'firstname' => 'Test',
            'lastname' => 'User',
            'datecreated' => date('Y-m-d'),
            'status' => 'Active',
        ]);
        
        $this->assertIsInt($clientId);
        $this->assertGreaterThan(0, $clientId);
        
        // Clean up
        Capsule::table('tblclients')->where('id', $clientId)->delete();
    }
    
    /**
     * Test service provisioning
     */
    public function testServiceProvisioning(): void
    {
        // Create test client
        $clientId = Capsule::table('tblclients')->insertGetId([
            'email' => 'service_test_' . uniqid() . '@example.com',
            'firstname' => 'Test',
            'lastname' => 'User',
            'datecreated' => date('Y-m-d'),
            'status' => 'Active',
        ]);
        
        // Create service
        $serviceId = Capsule::table('tblhosting')->insertGetId([
            'userid' => $clientId,
            'packageid' => 1,
            'domain' => 'test-service-' . uniqid() . '.com',
            'regdate' => date('Y-m-d'),
            'domainstatus' => 'Pending',
        ]);
        
        $this->assertIsInt($serviceId);
        
        // Update status (simulating module)
        Capsule::table('tblhosting')
            ->where('id', $serviceId)
            ->update(['domainstatus' => 'Active']);
        
        $service = Capsule::table('tblhosting')
            ->where('id', $serviceId)
            ->first();
        
        $this->assertEquals('Active', $service->domainstatus);
        
        // Clean up
        Capsule::table('tblhosting')->where('id', $serviceId)->delete();
        Capsule::table('tblclients')->where('id', $clientId)->delete();
    }
    
    /**
     * Test invoice creation and payment
     */
    public function testInvoicePayment(): void
    {
        // Create client
        $clientId = Capsule::table('tblclients')->insertGetId([
            'email' => 'invoice_test_' . uniqid() . '@example.com',
            'firstname' => 'Test',
            'lastname' => 'User',
            'datecreated' => date('Y-m-d'),
            'status' => 'Active',
        ]);
        
        // Create invoice
        $invoiceId = Capsule::table('tblinvoices')->insertGetId([
            'userid' => $clientId,
            'date' => date('Y-m-d'),
            'duedate' => date('Y-m-d', strtotime('+7 days')),
            'status' => 'Unpaid',
            'total' => 99.99,
        ]);
        
        // Add invoice item
        Capsule::table('tblinvoiceitems')->insert([
            'invoiceid' => $invoiceId,
            'userid' => $clientId,
            'type' => 'Hosting',
            'description' => 'Test Service',
            'amount' => 99.99,
        ]);
        
        // Simulate payment
        Capsule::table('tblinvoices')
            ->where('id', $invoiceId)
            ->update([
                'status' => 'Paid',
                'datepaid' => date('Y-m-d H:i:s'),
            ]);
        
        $invoice = Capsule::table('tblinvoices')
            ->where('id', $invoiceId)
            ->first();
        
        $this->assertEquals('Paid', $invoice->status);
        
        // Clean up
        Capsule::table('tblinvoices')->where('id', $invoiceId)->delete();
        Capsule::table('tblclients')->where('id', $clientId)->delete();
    }
}
```

## Mock Objects

### Module Mock

```php
<?php
/**
 * Mock WHMCS module functions
 */
class MockWHMCSModule
{
    private array $calls = [];
    
    /**
     * Mock CreateAccount
     */
    public function mockCreateAccount(array $params): array
    {
        $this->calls['CreateAccount'][] = $params;
        
        return [
            'success' => true,
            'account_id' => 'MOCK_' . uniqid(),
        ];
    }
    
    /**
     * Mock SuspendAccount
     */
    public function mockSuspendAccount(array $params): array
    {
        $this->calls['SuspendAccount'][] = $params;
        
        return ['success' => true];
    }
    
    /**
     * Mock TerminateAccount
     */
    public function mockTerminateAccount(array $params): array
    {
        $this->calls['TerminateAccount'][] = $params;
        
        return ['success' => true];
    }
    
    /**
     * Get recorded calls
     */
    public function getCalls(string $function = null): array
    {
        if ($function) {
            return $this->calls[$function] ?? [];
        }
        
        return $this->calls;
    }
    
    /**
     * Reset recorded calls
     */
    public function reset(): void
    {
        $this->calls = [];
    }
}
```

### API Mock

```php
<?php
/**
 * Mock external API
 */
class MockExternalAPI
{
    private array $responses = [];
    private array $requests = [];
    
    /**
     * Set up mock response
     */
    public function setResponse(string $endpoint, array $response): void
    {
        $this->responses[$endpoint] = $response;
    }
    
    /**
     * Make API call
     */
    public function call(string $endpoint, array $data = []): array
    {
        $this->requests[] = [
            'endpoint' => $endpoint,
            'data' => $data,
            'timestamp' => time(),
        ];
        
        return $this->responses[$endpoint] ?? ['error' => 'Not configured'];
    }
    
    /**
     * Get recorded requests
     */
    public function getRequests(): array
    {
        return $this->requests;
    }
    
    /**
     * Verify request was made
     */
    public function wasCalled(string $endpoint): bool
    {
        foreach ($this->requests as $request) {
            if ($request['endpoint'] === $endpoint) {
                return true;
            }
        }
        
        return false;
    }
    
    /**
     * Verify request data
     */
    public function wasCalledWith(string $endpoint, array $data): bool
    {
        foreach ($this->requests as $request) {
            if ($request['endpoint'] === $endpoint && $request['data'] === $data) {
                return true;
            }
        }
        
        return false;
    }
}
```

## Functional Testing

### Module Functional Tests

```php
<?php
/**
 * Functional tests for module
 */
class ModuleFunctionalTest extends PHPUnit_Framework_TestCase
{
    private MockExternalAPI $api;
    private MockWHMCSModule $module;
    
    protected function setUp(): void
    {
        $this->api = new MockExternalAPI();
        $this->module = new MockWHMCSModule();
        
        // Set up mock API responses
        $this->api->setResponse('/accounts/create', [
            'success' => true,
            'account_id' => 'ACC123',
            'username' => 'testuser',
        ]);
    }
    
    /**
     * Test full account lifecycle
     */
    public function testFullAccountLifecycle(): void
    {
        // Create account
        $createResult = $this->module->mockCreateAccount([
            'domain' => 'test-lifecycle.com',
            'username' => 'testuser',
            'password' => 'password123',
        ]);
        
        $this->assertTrue($createResult['success']);
        $this->assertArrayHasKey('account_id', $createResult);
        
        // Suspend account
        $suspendResult = $this->module->mockSuspendAccount([
            'account_id' => $createResult['account_id'],
            'reason' => 'Payment overdue',
        ]);
        
        $this->assertTrue($suspendResult['success']);
        
        // Unsuspend account
        $unsuspendResult = $this->module->mockUnsuspendAccount([
            'account_id' => $createResult['account_id'],
        ]);
        
        $this->assertTrue($unsuspendResult['success']);
        
        // Terminate account
        $terminateResult = $this->module->mockTerminateAccount([
            'account_id' => $createResult['account_id'],
        ]);
        
        $this->assertTrue($terminateResult['success']);
    }
    
    /**
     * Test error handling
     */
    public function testErrorHandling(): void
    {
        $this->api->setResponse('/accounts/create', [
            'success' => false,
            'error' => 'Domain already exists',
        ]);
        
        $result = $this->module->mockCreateAccount([
            'domain' => 'existing.com',
        ]);
        
        $this->assertFalse($result['success']);
        $this->assertArrayHasKey('error', $result);
    }
}
```

## Performance Testing

```php
<?php
/**
 * Performance tests
 */
class PerformanceTest extends PHPUnit_Framework_TestCase
{
    /**
     * Test query performance
     */
    public function testQueryPerformance(): void
    {
        $start = microtime(true);
        
        $clients = Capsule::table('tblclients')
            ->select([
                'tblclients.*',
                Capsule::raw('COUNT(tblhosting.id) as service_count'),
            ])
            ->leftJoin('tblhosting', 'tblclients.id', '=', 'tblhosting.userid')
            ->groupBy('tblclients.id')
            ->limit(100)
            ->get();
        
        $duration = (microtime(true) - $start) * 1000;
        
        // Query should complete in under 100ms
        $this->assertLessThan(100, $duration, "Query took {$duration}ms");
    }
    
    /**
     * Test batch processing performance
     */
    public function testBatchProcessingPerformance(): void
    {
        $start = microtime(true);
        
        $count = 0;
        Capsule::table('tblclients')
            ->chunk(100, function($clients) use (&$count) {
                $count += count($clients);
            });
        
        $duration = (microtime(true) - $start);
        
        // Should process 100 records per second minimum
        $this->assertGreaterThan(0, $count);
        $this->assertLessThan($count / 100, $duration);
    }
}
```

## CI/CD Integration

### PHPUnit Configuration

```xml
<?xml version="1.0" encoding="UTF-8"?>
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
        <testsuite name="Functional">
            <directory>tests/Functional</directory>
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
        <env name="WHMCS_TEST_MODE" value="true"/>
    </php>
</phpunit>
```

## Best Practices

1. **Test all scenarios** - Happy path and edge cases
2. **Use mocks** - Isolate from external services
3. **Clean up data** - Reset test data after tests
4. **Run in CI** - Automate tests on every push
5. **Measure coverage** - Track test coverage
6. **Performance test** - Ensure acceptable speeds

## Related Documentation

- [whmcs-module-testing.md](whmcs-module-testing.md)
- [whmcs-advanced-performance.md](whmcs-advanced-performance.md)
