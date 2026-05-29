# WHMCS Testing - Hook Tests

## Skill Description
Implement testing for WHMCS hooks to verify hook handlers execute correctly, modify data properly, and handle edge cases.

## Prerequisites
- WHMCS 7.0+ installation
- PHP 7.4+ with PHPUnit
- Hook system knowledge
- Basic testing knowledge

## Step-by-Step Implementation

### 1. Hook Test Base
```php
<?php
// tests/Unit/Hooks/HookTestCase.php

namespace WHMCS\Module\YourModule\Tests\Unit\Hooks;

use PHPUnit\Framework\TestCase;

abstract class HookTestCase extends TestCase
{
    protected array $hookResults = [];

    protected function registerHook(string $name, callable $callback): void
    {
        add_hook($name, 1, $callback);
    }

    protected function simulateHook(string $hookName, array $vars = []): mixed
    {
        $this->hookResults = [];

        // Execute hook handlers
        $hooks = $this->getRegisteredHooks($hookName);

        foreach ($hooks as $hook) {
            $result = call_user_func($hook, $vars);
            if ($result !== null) {
                $this->hookResults[] = $result;
            }
        }

        return end($this->hookResults) ?: null;
    }

    abstract protected function getRegisteredHooks(string $name): array;
}
```

### 2. Hook Test Examples
```php
<?php
// tests/Unit/Hooks/ServiceHooksTest.php

namespace WHMCS\Module\YourModule\Tests\Unit\Hooks;

class ServiceHooksTest extends HookTestCase
{
    public function testServiceActivationHook(): void
    {
        $vars = [
            'serviceid' => 1,
            'userid' => 1,
            'domain' => 'example.com',
            'status' => 'Active'
        ];

        $result = $this->simulateHook('ServiceActivate', $vars);

        $this->assertNotNull($result);
        $this->assertArrayHasKey('serviceid', $result);
    }

    public function testInvoicePaidHook(): void
    {
        $vars = [
            'invoiceid' => 1,
            'userid' => 1,
            'total' => 100.00
        ];

        $result = $this->simulateHook('InvoicePaid', $vars);

        $this->assertNotNull($result);
    }

    public function testClientAddHook(): void
    {
        $vars = [
            'userid' => 1,
            'firstname' => 'John',
            'lastname' => 'Doe',
            'email' => 'john@example.com'
        ];

        $result = $this->simulateHook('ClientAdd', $vars);

        $this->assertNotNull($result);
        $this->assertEquals('John', $result['firstname']);
    }
}
```

## Common Pitfalls and Solutions

| Pitfall | Solution |
|---------|----------|
| Hook execution order | Test with multiple handlers |
| Data modification | Verify before/after states |
| Error handling | Test with invalid data |

## Testing Checklist

- [ ] Test hook registration
- [ ] Test hook execution
- [ ] Test data modification
- [ ] Test error handling

## Reference Links

- [WHMCS Hook System](https://developers.whmcs.com/hooks/)
