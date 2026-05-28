# WHMCS Module Testing Strategies

**Version:** 8.0 | **Updated:** 2026-05-28

## Overview

This guide covers testing strategies for WHMCS modules, including unit testing, integration testing, and automated testing workflows that ensure module reliability and quality.

---

## Testing Framework Setup

### PHPUnit Configuration

```xml
<?xml version="1.0" encoding="UTF-8"?>
<phpunit xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:noNamespaceSchemaLocation="https://schema.phpunit.de/9.5/phpunit.xsd"
         bootstrap="tests/bootstrap.php"
         colors="true"
         convertErrorsToExceptions="true">
    <testsuites>
        <testsuite name="Unit">
            <directory suffix="Test.php">tests/Unit</directory>
        </testsuite>
        <testsuite name="Integration">
            <directory suffix="Test.php">tests/Integration</directory>
        </testsuite>
    </testsuites>
    <coverage>
        <include>
            <directory suffix=".php">modules/your_module</directory>
        </include>
    </coverage>
</phpunit>
```

### Test Bootstrap

```php
<?php
/**
 * PHPUnit Bootstrap
 */

// Define WHMCS root if not defined
if (!defined('WHMCS')) {
    define('WHMCS', dirname(__DIR__));
}

require_once WHMCS . '/init.php';
require_once __DIR__ . '/TestCase.php';
```

## Unit Testing

### Unit Test Example

```php
<?php
/**
 * Gateway Module Unit Tests
 */

namespace Tests\Unit;

class YourGatewayTest extends \PHPUnit\Framework\TestCase
{
    protected $gateway;
    
    protected function setUp(): void
    {
        parent::setUp();
        
        // Initialize test gateway
        $this->gateway = new \WHMCS\Module\Gateway();
    }
    
    /**
     * Test signature generation
     */
    public function testSignatureGeneration()
    {
        $data = ['invoice_id' => 123, 'amount' => 100.00];
        $secret = 'test_secret';
        
        $signature = generateSignature($data, $secret);
        
        $this->assertNotEmpty($signature);
        $this->assertEquals(64, strlen($signature));
    }
    
    /**
     * Test signature verification
     */
    public function testSignatureVerification()
    {
        $data = ['invoice_id' => 123, 'amount' => 100.00];
        $secret = 'test_secret';
        
        $signature = generateSignature($data, $secret);
        
        $this->assertTrue(
            verifySignature($data, $signature, $secret)
        );
    }
    
    /**
     * Test invalid signature detection
     */
    public function testInvalidSignatureDetection()
    {
        $data = ['invoice_id' => 123, 'amount' => 100.00];
        $secret = 'test_secret';
        $wrongSignature = 'invalid_signature';
        
        $this->assertFalse(
            verifySignature($data, $wrongSignature, $secret)
        );
    }
    
    /**
     * Test input validation
     */
    public function testValidInputValidation()
    {
        $input = [
            'invoice_id'   => '123',
            'amount'       => '100.00',
            'currency'     => 'USD',
        ];
        
        $validated = validateModuleInput($input);
        
        $this->assertIsArray($validated);
        $this->assertEquals(123, $validated['invoice_id']);
        $this->assertEquals(100.00, $validated['amount']);
    }
    
    /**
     * Test invalid input rejection
     */
    public function testInvalidInputRejection()
    {
        $input = [
            'invoice_id' => '<script>alert(1)</script>',
            'amount' => -100,
        ];
        
        $validated = validateModuleInput($input);
        
        $this->assertFalse($validated);
    }
}
```

## Integration Testing

### Database Integration Test

```php
<?php
/**
 * Database Integration Tests
 */

namespace Tests\Integration;

class DatabaseTest extends \WHMCS\Test\IntegrationTestCase
{
    /**
     * Test module table creation
     */
    public function testModuleTableCreation()
    {
        $tableName = 'mod_your_addon';
        
        $this->assertTrue(
            WHMCS\Database\Capsule::schema()->hasTable($tableName)
        );
    }
    
    /**
     * Test record insertion
     */
    public function testRecordInsertion()
    {
        $clientId = 1;
        
        $id = WHMCS\Database\Capsule::table('mod_your_addon')
            ->insertGetId([
                'client_id'   => $clientId,
                'data'        => json_encode(['test' => true]),
                'created_at'  => date('Y-m-d H:i:s'),
            ]);
        
        $this->assertIsInt($id);
        $this->assertGreaterThan(0, $id);
    }
    
    /**
     * Test record retrieval
     */
    public function testRecordRetrieval()
    {
        $clientId = 1;
        
        $record = WHMCS\Database\Capsule::table('mod_your_addon')
            ->where('client_id', $clientId)
            ->first();
        
        $this->assertNotNull($record);
        $this->assertEquals($clientId, $record->client_id);
    }
    
    /**
     * Test record update
     */
    public function testRecordUpdate()
    {
        $id = 1;
        
        $affected = WHMCS\Database\Capsule::table('mod_your_addon')
            ->where('id', $id)
            ->update([
                'data' => json_encode(['updated' => true]),
                'updated_at' => date('Y-m-d H:i:s'),
            ]);
        
        $this->assertEquals(1, $affected);
    }
    
    /**
     * Test record deletion
     */
    public function testRecordDeletion()
    {
        $id = 1;
        
        $deleted = WHMCS\Database\Capsule::table('mod_your_addon')
            ->where('id', $id)
            ->delete();
        
        $this->assertEquals(1, $deleted);
    }
}
```

## Module Activation Testing

```php
<?php
/**
 * Module Lifecycle Tests
 */

namespace Tests\Integration;

class ModuleLifecycleTest extends \WHMCS\Test\IntegrationTestCase
{
    private $moduleName = 'your_addon';
    
    /**
     * Test module activation
     */
    public function testModuleActivation()
    {
        $module = new \WHMCS\Module\Addon($this->moduleName);
        
        $result = $module->activate();
        
        $this->assertTrue($result['success'] ?? false);
        $this->assertContains('success', strtolower($result['description']));
    }
    
    /**
     * Test tables created on activation
     */
    public function testTablesCreatedOnActivation()
    {
        $module = new \WHMCS\Module\Addon($this->moduleName);
        
        // Ensure activation runs first
        if (!$module->isActive()) {
            $module->activate();
        }
        
        $this->assertTrue(
            WHMCS\Database\Capsule::schema()->hasTable('mod_your_addon')
        );
        
        $this->assertTrue(
            WHMCS\Database\Capsule::schema()->hasTable('mod_your_addon_data')
        );
    }
    
    /**
     * Test module configuration stored
     */
    public function testModuleConfigurationStored()
    {
        $configKey = 'your_addon_version';
        $expectedVersion = '1.0.0';
        
        $storedValue = get_config_var($configKey);
        
        $this->assertEquals($expectedVersion, $storedValue);
    }
    
    /**
     * Test module deactivation cleanup
     */
    public function testModuleDeactivationCleanup()
    {
        $module = new \WHMCS\Module\Addon($this->moduleName);
        
        $result = $module->deactivate();
        
        $this->assertTrue($result['success'] ?? false);
        
        // Verify tables dropped
        $this->assertFalse(
            WHMCS\Database\Capsule::schema()->hasTable('mod_your_addon')
        );
        
        $this->assertFalse(
            WHMCS\Database\Capsule::schema()->hasTable('mod_your_addon_data')
        );
    }
}
```

## API Mock Testing

```php
<?php
/**
 * API Mock Tests
 */

namespace Tests\Unit;

class ApiMockTest extends \PHPUnit\Framework\TestCase
{
    private $mockHandler;
    
    protected function setUp(): void
    {
        parent::setUp();
        
        $this->mockHandler = new \GuzzleHttp\Handler\Mock();
    }
    
    /**
     * Test successful API response
     */
    public function testSuccessfulApiResponse()
    {
        $this->mockHandler->append(
            new \GuzzleHttp\Psr7\Response(200, [], json_encode([
                'success' => true,
                'data'    => ['id' => 123],
            ]))
        );
        
        $client = new \GuzzleHttp\Client([
            'handler' => HandlerStack::create($this->mockHandler),
        ]);
        
        $response = $client->post('https://api.example.com/test');
        $result = json_decode($response->getBody(), true);
        
        $this->assertTrue($result['success']);
        $this->assertEquals(123, $result['data']['id']);
    }
    
    /**
     * Test API error handling
     */
    public function testApiErrorHandling()
    {
        $this->mockHandler->append(
            new \GuzzleHttp\Psr7\Response(400, [], json_encode([
                'error' => 'Invalid request',
            ]))
        );
        
        $client = new \GuzzleHttp\Client([
            'handler' => HandlerStack::create($this->mockHandler),
        ]);
        
        $this->expectException(\RuntimeException::class);
        
        $client->post('https://api.example.com/test');
    }
    
    /**
     * Test timeout handling
     */
    public function testTimeoutHandling()
    {
        $this->mockHandler->append(
            new \GuzzleHttp\Exception\ConnectException(
                'Connection timeout',
                new \GuzzleHttp\Psr7\Request('POST', 'https://api.example.com')
            )
        );
        
        $client = new \GuzzleHttp\Client([
            'handler' => HandlerStack::create($this->mockHandler),
            'timeout' => 1,
        ]);
        
        $this->expectException(\RuntimeException::class);
        
        try {
            $client->post('https://api.example.com/test');
        } catch (\GuzzleHttp\Exception\ConnectException $e) {
            $this->assertContains('timeout', $e->getMessage());
            throw $e;
        }
    }
}
```

## Hook Testing

```php
<?php
/**
 * Hook Tests
 */

namespace Tests\Integration;

class HookTest extends \WHMCS\Test\IntegrationTestCase
{
    /**
     * Test hook registration
     */
    public function testHookRegistration()
    {
        $hooks = your_module_registerHooks();
        
        $this->assertIsArray($hooks);
        $this->assertArrayHasKey('ClientAdd', $hooks);
        $this->assertArrayHasKey('handler', $hooks['ClientAdd']);
    }
    
    /**
     * Test hook execution
     */
    public function testHookExecution()
    {
        $this->markTestSkipped('Requires live WHMCS environment');
        
        $vars = [
            'userid' => 1,
            'firstname' => 'Test',
            'lastname' => 'User',
            'email' => 'test@example.com',
        ];
        
        // Simulate ClientAdd hook
        $result = onClientAdd($vars);
        
        $this->assertTrue($result);
    }
}
```

## Simulation Testing

```php
<?php
/**
 * Simulation Tests for Payment Gateway
 */

namespace Tests\Simulation;

class GatewaySimulationTest extends \PHPUnit\Framework\TestCase
{
    /**
     * @dataProvider paymentScenarioProvider
     */
    public function testPaymentScenarios($scenario, $input, $expected)
    {
        $gateway = new \WHMCS\Module\Gateway();
        
        $result = $gateway->simulatePayment($input);
        
        if ($expected['success']) {
            $this->assertTrue($result['success']);
            $this->assertEquals($expected['transaction_id'], $result['transaction_id']);
        } else {
            $this->assertFalse($result['success']);
            $this->assertContains($expected['error'], $result['error']);
        }
    }
    
    public function paymentScenarioProvider()
    {
        return [
            'successful payment' => [
                'scenario' => 'Success',
                'input'    => [
                    'invoice_id' => 123,
                    'amount'    => 100.00,
                    'currency'  => 'USD',
                ],
                'expected' => [
                    'success'       => true,
                    'transaction_id' => 'TXN_123456',
                ],
            ],
            'invalid amount' => [
                'scenario' => 'Invalid Amount',
                'input'    => [
                    'invoice_id' => 123,
                    'amount'    => -100,
                    'currency'  => 'USD',
                ],
                'expected' => [
success'       => false,
                    'error'      => 'Invalid amount',
                ],
            ],
        ];
    }
}
```

## Test Coverage Goals

| Module Type | Minimum Coverage | Target Coverage |
|-------------|------------------|-----------------|
| Gateways | 80% | 90% |
| Registrars | 75% | 85% |
| Addons | 70% | 80% |
| Notifications | 75% | 85% |

## Testing Checklist

### Pre-Release
- [ ] Unit tests for all public functions
- [ ] Integration tests for database operations
- [ ] Activation/deactivation tests
- [ ] API mock tests
- [ ] Hook execution tests
- [ ] Error handling tests
- [ ] Security validation tests
- [ ] Performance tests

### Continuous Integration
- [ ] Automated test runs on commit
- [ ] Code coverage reporting
- [ ] Static analysis integration
- [ ] Security scanning integration

---

## Related Skills and Workflows

- `module-security-standards` - Security testing requirements
- `module-performance-best-practices` - Performance benchmarks
- `module-release-checklist` - Release testing checklist
- `payment-gateway-developer-guide` - Gateway testing
