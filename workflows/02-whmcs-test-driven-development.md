# WHMCS Test-Driven Development Workflow

## Overview
This workflow establishes a TDD approach for WHMCS module development, including unit tests, integration tests, and CI/CD integration.

## Prerequisites
- PHPUnit 9.x or later
- WHMCS testing toolkit
- Mockery for mocking WHMCS dependencies

## Step 1: Test Environment Setup

```bash
# Create tests directory structure
mkdir -p tests/Unit tests/Feature tests/Mocks

# Install testing dependencies
composer require --dev \
    phpunit/phpunit:^9.5 \
    mockery/mockery:^1.5 \
    fakerphp/faker:^1.19
```

## Step 2: PHPUnit Configuration

```xml
<!-- phpunit.xml -->
<?xml version="1.0" encoding="UTF-8"?>
<phpunit xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:noNamespaceSchemaLocation="vendor/phpunit/phpunit/phpunit.xsd"
         bootstrap="tests/bootstrap.php"
         colors="true"
         cacheDirectory=".phpunit.cache"
         executionOrder="random"
         failOnWarning="true"
         failOnRisky="true"
         failOnEmptyTestSuite="true"
         beStrictAboutOutputDuringTests="true">
    <testsuites>
        <testsuite name="Unit">
            <directory>tests/Unit</directory>
        </testsuite>
        <testsuite name="Feature">
            <directory>tests/Feature</directory>
        </testsuite>
    </testsuites>
    <coverage>
        <include>
            <directory suffix=".php">src</directory>
        </include>
        <exclude>
            <directory>src/Config</directory>
        </exclude>
    </coverage>
</phpunit>
```

## Step 3: Test Bootstrap

```php
<?php
// tests/bootstrap.php

define('WHMCS', true);
define('ROOTDIR', dirname(__DIR__));

// Autoload
require_once __DIR__ . '/../vendor/autoload.php';

// WHMCS minimal bootstrap
require_once ROOTDIR . '/includes/init.php';

// Load test mocks
require_once __DIR__ . '/Mocks/whmcs_functions.php';
```

## Step 4: WHMCS Function Mocks

```php
<?php
// tests/Mocks/whmcs_functions.php

// Mock commonly used WHMCS functions
if (!function_exists('full_query')) {
    function full_query($query, $params = []) {
        return \WHMCS\Database\Capsule::connection()->statement($query, $params);
    }
}

if (!function_exists('get_query_val')) {
    function get_query_val($table, $field, $where, $value) {
        return \WHMCS\Database\Capsule::table($table)
            ->where($where, $value)
            ->value($field);
    }
}

if (!function_exists('logActivity')) {
    function logActivity($message, $clientId = null) {
        // Test implementation
    }
}

if (!function_exists('run_hook')) {
    function run_hook($hookName, $params = []) {
        return [];
    }
}
```

## Step 5: Unit Test Example

```php
<?php
// tests/Unit/ModuleServiceTest.php

namespace Tests\Unit;

use PHPUnit\Framework\TestCase;
use Mockery;
use WHMCS\Module\Addon\YourModule\Service\ModuleService;

class ModuleServiceTest extends TestCase
{
    protected function tearDown(): void
    {
        Mockery::close();
        parent::tearDown();
    }

    public function testProcessDataReturnsExpectedFormat(): void
    {
        // Arrange
        $service = new ModuleService();

        // Act
        $result = $service->processData([
            'id' => 1,
            'name' => 'Test Client',
            'email' => 'test@example.com'
        ]);

        // Assert
        $this->assertIsArray($result);
        $this->assertArrayHasKey('id', $result);
        $this->assertArrayHasKey('processed_at', $result);
        $this->assertEquals('Test Client', $result['name']);
    }

    public function testValidateInputReturnsFalseForInvalidData(): void
    {
        $service = new ModuleService();
        $result = $service->validateInput([]);

        $this->assertFalse($result);
    }

    public function testValidateInputReturnsTrueForValidData(): void
    {
        $service = new ModuleService();
        $result = $service->validateInput([
            'name' => 'Valid Name',
            'email' => 'valid@example.com'
        ]);

        $this->assertTrue($result);
    }

    /**
     * @dataProvider providerTestDataFormatting
     */
    public function testDataFormatting($input, $expected): void
    {
        $service = new ModuleService();
        $result = $service->formatData($input);

        $this->assertEquals($expected, $result);
    }

    public function providerTestDataFormatting(): array
    {
        return [
            ['lowercase', 'LOWERCASE'],
            ['MixedCase', 'MIXEDCASE'],
            ['', ''],
        ];
    }
}
```

## Step 6: Integration Test Example

```php
<?php
// tests/Feature/ModuleActivationTest.php

namespace Tests\Feature;

use PHPUnit\Framework\TestCase;
use WHMCS\Database\Capsule;

class ModuleActivationTest extends TestCase
{
    public function testModuleActivatesSuccessfully(): void
    {
        // Call activation function
        $result = your_module_name_activate();

        $this->assertEquals('success', $result['status']);

        // Verify table was created
        $tables = Capsule::connection()->getDoctrineSchemaManager()->listTableNames();
        $this->assertContains('mod_your_module', $tables);
    }

    public function testModuleDeactivatesSuccessfully(): void
    {
        // First activate
        your_module_name_activate();

        // Then deactivate
        $result = your_module_name_deactivate();

        $this->assertEquals('success', $result['status']);

        // Verify table was dropped
        $tables = Capsule::connection()->getDoctrineSchemaManager()->listTableNames();
        $this->assertNotContains('mod_your_module', $tables);
    }

    public function testUpgradeMigrationsRunCorrectly(): void
    {
        // Setup: Create old version state
        your_module_name_activate();

        // Simulate upgrade
        your_module_name_upgrade(['version' => '1.0.0']);

        // Verify new fields exist
        $columns = Capsule::connection()
            ->getDoctrineSchemaManager()
            ->listTableColumns('mod_your_module');

        $columnNames = array_map(fn($c) => $c->getName(), $columns);
        $this->assertContains('new_field', $columnNames);
    }
}
```

## Step 7: CI/CD Integration

```yaml
# .github/workflows/test.yml
name: Tests

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]

jobs:
  unit-tests:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3

      - name: Setup PHP
        uses: shivammathur/setup-php@v2
        with:
          php-version: '8.1'
          extensions: pdo_mysql, json

      - name: Install Dependencies
        run: composer install --prefer-dist

      - name: Run PHPUnit
        run: ./vendor/bin/phpunit --testsuite Unit

  integration-tests:
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
      - uses: actions/checkout@v3

      - name: Setup PHP
        uses: shivammathur/setup-php@v2
        with:
          php-version: '8.1'
          extensions: pdo_mysql, json

      - name: Install Dependencies
        run: composer install --prefer-dist

      - name: Run Integration Tests
        env:
          DB_HOST: 127.0.0.1
          DB_NAME: whmcs_test
          DB_USER: root
          DB_PASSWORD: root
        run: ./vendor/bin/phpunit --testsuite Feature
```

## Step 8: Running Tests

```bash
# Run all tests
./vendor/bin/phpunit

# Run only unit tests
./vendor/bin/phpunit --testsuite Unit

# Run with coverage
./vendor/bin/phpunit --coverage-html coverage/

# Run specific test class
./vendor/bin/phpunit tests/Unit/ModuleServiceTest.php

# Run tests matching pattern
./vendor/bin/phpunit --filter testProcessData

# Watch mode (with PHPUnit bridge)
./vendor/bin/phpunit --watch
```

## Verification Checklist

- [ ] PHPUnit installed and configured
- [ ] Bootstrap file created
- [ ] WHMCS function mocks implemented
- [ ] First unit test written (failing)
- [ ] Code implemented to pass test
- [ ] All unit tests passing
- [ ] Integration tests written
- [ ] CI/CD pipeline configured
- [ ] Test coverage above 80%
- [ ] All tests passing in CI

## Best Practices

1. **Red-Green-Refactor**: Write failing test first, then make it pass, then refactor
2. **Isolation**: Each test should be independent and not rely on other tests
3. **Descriptive Names**: Use clear test method names that describe what's being tested
4. **Mock External Dependencies**: Don't make real API calls in unit tests
5. **Test Edge Cases**: Include boundary conditions and error scenarios
