# WHMCS Module Unit Testing Workflow

## Overview
This workflow provides a comprehensive guide for writing and executing unit tests for WHMCS modules, ensuring code quality and preventing regressions.

## Prerequisites
- WHMCS installation (v8.0+ recommended)
- PHPUnit installed (`composer require --dev phpunit/phpunit`)
- Access to module source code
- Local development environment

## Step-by-Step Guide

### Step 1: Set Up Testing Environment
```bash
# Navigate to module directory
cd /path/to/your/module

# Initialize composer if needed
composer init

# Install PHPUnit
composer require --dev phpunit/phpunit:^9.0

# Create phpunit.xml configuration
cat > phpunit.xml << 'EOF'
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
    </testsuites>
    <coverage>
        <include>
            <directory suffix=".php">src</directory>
        </include>
    </coverage>
</phpunit>
EOF
```

### Step 2: Create Bootstrap File
```php
// tests/bootstrap.php
<?php
// Load WHMCS autoloader
require_once __DIR__ . '/../../../../init.php';
require_once ROOTDIR . '/includes/vendor/autoload.php';

// Set up test database connection if needed
define('TESTING_MODE', true);
```

### Step 3: Write Unit Tests

#### Test Class Structure
```php
// tests/Unit/ModuleServiceTest.php
<?php
namespace WHMCS\Module\YourModule\Tests;

use PHPUnit\Framework\TestCase;
use WHMCS\Module\YourModule\Service\ModuleService;

class ModuleServiceTest extends TestCase
{
    protected ModuleService $service;

    protected function setUp(): void
    {
        parent::setUp();
        $this->service = new ModuleService();
    }

    public function testServiceInitialization()
    {
        $this->assertInstanceOf(ModuleService::class, $this->service);
    }

    public function testConfigArrayStructure()
    {
        $expectedKeys = ['name', 'description', 'version', 'author'];
        $config = $this->service->getModuleConfig();

        foreach ($expectedKeys as $key) {
            $this->assertArrayHasKey($key, $config);
        }
    }

    public function testValidateInputWithValidData()
    {
        $input = ['param1' => 'value1', 'param2' => 'value2'];
        $result = $this->service->validateInput($input);
        $this->assertTrue($result);
    }

    public function testValidateInputWithInvalidData()
    {
        $input = ['invalid' => 'data'];
        $this->expectException(\InvalidArgumentException::class);
        $this->service->validateInput($input);
    }

    /**
     * @dataProvider additionProvider
     */
    public function testAddition($a, $b, $expected)
    {
        $this->assertEquals($expected, $a + $b);
    }

    public function additionProvider(): array
    {
        return [
            'integers' => [1, 2, 3],
            'floats' => [1.5, 2.5, 4.0],
            'negative' => [-1, -1, -2],
        ];
    }
}
```

### Step 4: Test WHMCS Hooks
```php
// tests/Unit/HookTest.php
<?php
namespace WHMCS\Module\YourModule\Tests;

use PHPUnit\Framework\TestCase;

class HookTest extends TestCase
{
    public function testClientAddHookRegisters()
    {
        $hookName = 'ClientAdd';
        $hookFunctions = \App::getHookFunctions($hookName);

        $this->assertContains('YourModuleHookFunction', $hookFunctions);
    }

    public function testInvoiceCreationHook()
    {
        $params = [
            'userid' => 1,
            'invoiceid' => 100,
            'total' => 99.99,
        ];

        // Simulate hook execution
        $result = run_hooks('InvoiceCreation', $params);

        $this->assertIsArray($result);
    }
}
```

### Step 5: Test API Calls (Mocked)
```php
// tests/Unit/ApiClientTest.php
<?php
namespace WHMCS\Module\YourModule\Tests;

use PHPUnit\Framework\TestCase;
use PHPUnit\Framework\MockObject\MockObject;

class ApiClientTest extends TestCase
{
    private MockObject $httpClient;
    private ApiClient $apiClient;

    protected function setUp(): void
    {
        parent::setUp();
        $this->httpClient = $this->createMock(HttpClient::class);
        $this->apiClient = new ApiClient($this->httpClient);
    }

    public function testApiCallSuccess()
    {
        $expectedResponse = ['status' => 'success', 'data' => ['id' => 123]];

        $this->httpClient
            ->expects($this->once())
            ->method('post')
            ->with('https://api.example.com/endpoint', $this->anything())
            ->willReturn($expectedResponse);

        $result = $this->apiClient->call('endpoint', ['param' => 'value']);

        $this->assertEquals($expectedResponse, $result);
    }

    public function testApiCallFailure()
    {
        $this->httpClient
            ->expects($this->once())
            ->method('post')
            ->willThrowException(new \Exception('API Error'));

        $this->expectException(\Exception::class);
        $this->apiClient->call('endpoint', []);
    }

    public function testRateLimiting()
    {
        $this->httpClient
            ->expects($this->exactly(3))
            ->method('post')
            ->willReturn(['status' => 'success']);

        for ($i = 0; $i < 3; $i++) {
            $this->apiClient->call('endpoint', []);
        }
    }
}
```

### Step 6: Run Tests
```bash
# Run all unit tests
./vendor/bin/phpunit

# Run with coverage report
./vendor/bin/phpunit --coverage-html coverage/

# Run specific test file
./vendor/bin/phpunit tests/Unit/ModuleServiceTest.php

# Run tests matching a pattern
./vendor/bin/phpunit --filter testValidate

# Run with verbose output
./vendor/bin/phpunit --testdox
```

### Step 7: CI/CD Integration
```yaml
# .github/workflows/phpunit.yml
name: PHPUnit Tests

on: [push, pull_request]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3

      - name: Setup PHP
        uses: shivammathur/setup-php@v2
        with:
          php-version: '8.1'
          extensions: dom, curl, libxml, mbstring, zip, gd

      - name: Install dependencies
        run: composer install --no-interaction

      - name: Run PHPUnit
        run: ./vendor/bin/phpunit --coverage-text
```

## Best Practices
- Test one thing per test method
- Use descriptive test method names (testDescriptionOfBehavior)
- Mock external dependencies (API calls, database queries)
- Keep tests independent - no test should depend on another
- Use data providers for testing multiple scenarios
- Aim for high code coverage but prioritize critical paths

## Troubleshooting

| Issue | Solution |
|-------|----------|
| WHMCS classes not found | Ensure bootstrap.php loads autoloader correctly |
| Database connection errors | Mock database calls or use test database |
| Hook functions undefined | Load module hooks before testing |
| API timeout in tests | Use mocked HTTP client |
| Session errors | Disable session_start() in test environment |

## Expected Output
```
PHPUnit 9.6.0

Time: 0.1 seconds, Memory: 10.00 MB

OK (15 tests, 25 assertions)
```
