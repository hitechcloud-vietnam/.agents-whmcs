# WHMCS Testing - Regression Tests

## Skill Description
Implement regression testing for WHMCS modules to ensure new changes don't break existing functionality.

## Prerequisites
- WHMCS 7.0+ installation
- PHP 7.4+ with PHPUnit
- Test suite already established
- Regression testing knowledge

## Step-by-Step Implementation

### 1. Regression Test Base
```php
<?php
// tests/Regression/RegressionTestCase.php

namespace WHMCS\Module\YourModule\Tests\Regression;

use PHPUnit\Framework\TestCase;

abstract class RegressionTestCase extends TestCase
{
    protected array $knownBugs = [];

    protected function assertNotRegressed(string $bugId, callable $test): void
    {
        if (isset($this->knownBugs[$bugId])) {
            $this->markTestSkipped("Known bug: {$bugId}");
        }

        $test();
    }

    protected function registerKnownBug(string $bugId, string $description): void
    {
        $this->knownBugs[$bugId] = $description;
    }
}
```

### 2. Regression Test Examples
```php
<?php
// tests/Regression/FunctionalityRegressionTest.php

namespace WHMCS\Module\YourModule\Tests\Regression;

class FunctionalityRegressionTest extends RegressionTestCase
{
    protected function setUp(): void
    {
        parent::setUp();

        // Register known bugs
        $this->registerKnownBug('BUG-001', 'Invoice calculation bug');
        $this->registerKnownBug('BUG-002', 'Email template variable issue');
    }

    public function testInvoiceCalculation(): void
    {
        $this->assertNotRegressed('BUG-001', function () {
            $result = $this->calculateInvoiceTotal(100, 0.1);

            $this->assertEquals(110, $result);
        });
    }

    public function testEmailTemplateVariables(): void
    {
        $this->assertNotRegressed('BUG-002', function () {
            $template = 'Hello {name}, your order #{order_id} is ready.';

            $processed = str_replace(
                ['{name}', '{order_id}'],
                ['John', '123'],
                $template
            );

            $this->assertEquals('Hello John, your order #123 is ready.', $processed);
        });
    }

    public function testClientCreation(): void
    {
        $client = $this->createClient([
            'firstname' => 'Jane',
            'lastname' => 'Doe',
            'email' => 'jane@example.com'
        ]);

        $this->assertNotEmpty($client['id']);
        $this->assertEquals('Jane', $client['firstname']);
    }

    public function testServiceProvisioning(): void
    {
        $result = $this->provisionService([
            'domain' => 'test.example.com',
            'package_id' => 1
        ]);

        $this->assertTrue($result['success']);
        $this->assertNotEmpty($result['service_id']);
    }

    private function calculateInvoiceTotal(float $subtotal, float $taxRate): float
    {
        return $subtotal * (1 + $taxRate);
    }

    private function createClient(array $data): array
    {
        global $db;

        $db->insert('tblclients', $data);

        return ['id' => $db->getLastInsertID()] + $data;
    }

    private function provisionService(array $data): array
    {
        return ['success' => true, 'service_id' => 1];
    }
}
```

## Common Pitfalls and Solutions

| Pitfall | Solution |
|---------|----------|
| Test coverage gaps | Expand test suite |
| Known issues | Track and document |
| False positives | Regular test maintenance |

## Testing Checklist

- [ ] Test core functionality
- [ ] Test edge cases
- [ ] Document known issues
- [ ] Run full suite regularly

## Reference Links

- [Regression Testing Guide](https://en.wikipedia.org/wiki/Regression_testing)
