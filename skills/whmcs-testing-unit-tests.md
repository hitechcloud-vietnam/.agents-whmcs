# WHMCS Testing - Unit Tests

## Skill Description
Set up and implement PHPUnit testing for WHMCS modules to ensure code quality, catch bugs early, and maintain reliable codebases.

## Prerequisites
- WHMCS 7.0+ installation
- PHP 7.4+ with PHPUnit
- Composer for dependency management
- Basic unit testing knowledge

## Step-by-Step Implementation

### 1. PHPUnit Configuration
```xml
<?xml version="1.0" encoding="UTF-8"?>
<phpunit xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:noNamespaceSchemaLocation="https://schema.phpunit.de/9.5/phpunit.xsd"
         bootstrap="tests/bootstrap.php"
         colors="true"
         stopOnFailure="false"
         cacheResult="false">
    <testsuites>
        <testsuite name="Unit">
            <directory suffix="Test.php">tests/Unit</directory>
        </testsuite>
        <testsuite name="Feature">
            <directory suffix="Test.php">tests/Feature</directory>
        </testsuite>
    </testsuites>
    <coverage>
        <include>
            <directory suffix=".php">src</directory>
        </include>
        <exclude>
            <directory>src/migrations</directory>
        </exclude>
    </coverage>
    <php>
        <env name="APP_ENV" value="testing"/>
        <env name="DB_CONNECTION" value="mysql"/>
        <env name="DB_DATABASE" value="whmcs_test"/>
    </php>
</phpunit>
```

### 2. Test Bootstrap
```php
<?php
// tests/bootstrap.php

// Define WHMCS paths
define('ROOTDIR', dirname(__DIR__));
define('CONFIGDIR', ROOTDIR);

// Load Composer autoloader
require_once __DIR__ . '/../vendor/autoload.php';

// Load test configuration
$testConfig = __DIR__ . '/config.php';
if (file_exists($testConfig)) {
    $config = include $testConfig;
    foreach ($config as $key => $value) {
        if (!defined($key)) {
            define($key, $value);
        }
    }
}

// Set up test database connection
function getTestDb(): PDO
{
    static $pdo = null;

    if ($pdo === null) {
        $host = '127.0.0.1';
        $dbname = 'whmcs_test';
        $user = 'root';
        $pass = '';

        $pdo = new PDO("mysql:host={$host};dbname={$dbname};charset=utf8mb4", $user, $pass, [
            PDO::ATTR_ERRMODE => PDO::ERRMODE_EXCEPTION,
            PDO::ATTR_DEFAULT_FETCH_MODE => PDO::FETCH_ASSOC
        ]);
    }

    return $pdo;
}
```

### 3. Test Case Base Class
```php
<?php
// tests/TestCase.php

namespace WHMCS\Module\YourModule\Tests;

use PHPUnit\Framework\TestCase as BaseTestCase;

abstract class TestCase extends BaseTestCase
{
    protected function setUp(): void
    {
        parent::setUp();
    }

    protected function tearDown(): void
    {
        parent::tearDown();
    }

    protected function assertEqualsArray(array $expected, array $actual, string $message = ''): void
    {
        sort($expected);
        sort($actual);
        $this->assertEquals($expected, $actual, $message);
    }

    protected function assertObjectHasProperty(string $property, object $object, string $message = ''): void
    {
        $this->assertArrayHasKey($property, (array) $object, $message);
    }

    protected function assertValidEmail(string $email, string $message = ''): void
    {
        $this->assertTrue(
            filter_var($email, FILTER_VALIDATE_EMAIL) !== false,
            $message ?: "Email '{$email}' is not valid"
        );
    }

    protected function assertJsonStructure(array $structure, array $json, string $message = ''): void
    {
        foreach ($structure as $key => $value) {
            if (is_string($key)) {
                $this->assertArrayHasKey($key, $json, $message);

                if (is_array($value)) {
                    $this->assertJsonStructure($value, $json[$key], $message);
                }
            }
        }
    }
}
```

### 4. Unit Test Examples
```php
<?php
// tests/Unit/Services/InvoiceServiceTest.php

namespace WHMCS\Module\YourModule\Tests\Unit\Services;

use WHMCS\Module\YourModule\Tests\TestCase;
use WHMCS\Module\YourModule\Services\InvoiceService;
use WHMCS\Module\YourModule\Validation\Validator;

class InvoiceServiceTest extends TestCase
{
    private InvoiceService $service;

    protected function setUp(): void
    {
        parent::setUp();
        $this->service = new InvoiceService();
    }

    public function testCreateInvoiceWithValidData(): void
    {
        $data = [
            'user_id' => 1,
            'items' => [
                ['description' => 'Product A', 'amount' => 100.00],
                ['description' => 'Product B', 'amount' => 50.00]
            ],
            'due_date' => '2024-12-31'
        ];

        $result = $this->service->create($data);

        $this->assertIsArray($result);
        $this->assertArrayHasKey('invoice_id', $result);
        $this->assertGreaterThan(0, $result['invoice_id']);
        $this->assertEquals(150.00, $result['total']);
    }

    public function testCreateInvoiceWithEmptyItems(): void
    {
        $this->expectException(\InvalidArgumentException::class);
        $this->expectExceptionMessage('At least one item is required');

        $data = [
            'user_id' => 1,
            'items' => [],
            'due_date' => '2024-12-31'
        ];

        $this->service->create($data);
    }

    public function testCalculateInvoiceTotal(): void
    {
        $items = [
            ['description' => 'Item 1', 'amount' => 100.00, 'taxed' => true],
            ['description' => 'Item 2', 'amount' => 50.00, 'taxed' => false]
        ];

        $result = $this->service->calculateTotal($items);

        $this->assertEquals(150.00, $result['subtotal']);
        $this->assertEquals(150.00, $result['total']);
    }

    public function testInvoiceStatusTransition(): void
    {
        $this->assertTrue($this->service->canTransitionTo('Unpaid', 'Paid'));
        $this->assertTrue($this->service->canTransitionTo('Unpaid', 'Overdue'));
        $this->assertTrue($this->service->canTransitionTo('Paid', 'Refunded'));
        $this->assertFalse($this->service->canTransitionTo('Paid', 'Unpaid'));
        $this->assertFalse($this->service->canTransitionTo('Cancelled', 'Paid'));
    }
}
```

```php
<?php
// tests/Unit/Validation/ValidatorTest.php

namespace WHMCS\Module\YourModule\Tests\Unit\Validation;

use WHMCS\Module\YourModule\Tests\TestCase;
use WHMCS\Module\YourModule\Validation\Validator;

class ValidatorTest extends TestCase
{
    public function testRequiredRule(): void
    {
        $validator = Validator::make(
            ['name' => 'John'],
            ['name' => 'required']
        );

        $this->assertFalse($validator->fails());
    }

    public function testRequiredRuleFails(): void
    {
        $validator = Validator::make(
            ['name' => ''],
            ['name' => 'required']
        );

        $this->assertTrue($validator->fails());
        $this->assertArrayHasKey('name', $validator->errors());
    }

    public function testEmailRule(): void
    {
        $validator = Validator::make(
            ['email' => 'test@example.com'],
            ['email' => 'email']
        );

        $this->assertFalse($validator->fails());
    }

    public function testEmailRuleFails(): void
    {
        $validator = Validator::make(
            ['email' => 'invalid-email'],
            ['email' => 'email']
        );

        $this->assertTrue($validator->fails());
    }

    public function testMinRule(): void
    {
        $validator = Validator::make(
            ['password' => '123456'],
            ['password' => 'min:6']
        );

        $this->assertFalse($validator->fails());
    }

    public function testMinRuleFails(): void
    {
        $validator = Validator::make(
            ['password' => '123'],
            ['password' => 'min:6']
        );

        $this->assertTrue($validator->fails());
    }

    public function testInRule(): void
    {
        $validator = Validator::make(
            ['status' => 'active'],
            ['status' => 'in:active,inactive']
        );

        $this->assertFalse($validator->fails());
    }

    public function testMultipleRules(): void
    {
        $validator = Validator::make(
            ['name' => 'John', 'email' => 'test@example.com'],
            [
                'name' => 'required|min:2|max:100',
                'email' => 'required|email'
            ]
        );

        $this->assertFalse($validator->fails());
        $this->assertEquals(['name' => 'John', 'email' => 'test@example.com'], $validator->validated());
    }
}
```

### 5. Running Tests
```bash
# Run all tests
./vendor/bin/phpunit

# Run specific test suite
./vendor/bin/phpunit --testsuite=Unit

# Run with coverage
./vendor/bin/phpunit --coverage-html coverage/

# Run specific test class
./vendor/bin/phpunit tests/Unit/Services/InvoiceServiceTest.php

# Run specific test method
./vendor/bin/phpunit --filter=testCreateInvoiceWithValidData

# Run tests in parallel
./vendor/bin/phpunit --parallel

# Run with debug output
./vendor/bin/phpunit --debug
```

## Common Pitfalls and Solutions

| Pitfall | Solution |
|---------|----------|
| Tests depend on database | Use mocks and fixtures |
| Tests are slow | Use in-memory database |
| Tests are flaky | Fix timing issues, use fixed seeds |
| Tests are hard to read | Use descriptive names, setup methods |
| Tests test too much | One assertion per test |

## Security Considerations

1. **Don't test with production data** - Use test database
2. **Secure test configuration** - Don't commit credentials
3. **Mock external services** - Don't rely on external APIs
4. **Clean up test artifacts** - Remove temporary files
5. **Don't expose sensitive in tests** - Use fake data

## Testing Checklist

- [ ] Run all unit tests
- [ ] Check test coverage
- [ ] Run tests in isolation
- [ ] Test edge cases
- [ ] Test error conditions
- [ ] Mock external dependencies
- [ ] Verify test cleanup
- [ ] Run tests in CI/CD

## Reference Links

- [PHPUnit Documentation](https://phpunit.readthedocs.io/)
- [PHPUnit Best Practices](https://phpunit.readthedocs.io/en/9.5/best-practices.html)
- [TDD in PHP](https://phpunit.readthedocs.io/en/9.5/annotations.html)
