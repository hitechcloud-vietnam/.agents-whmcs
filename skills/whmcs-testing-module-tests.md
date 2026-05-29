# WHMCS Testing - Module Tests

## Skill Description
Implement testing for WHMCS module functions, provisioning, and configuration to ensure reliable module operation.

## Prerequisites
- WHMCS 7.0+ installation
- PHP 7.4+ with PHPUnit
- Module development knowledge
- Testing framework setup

## Step-by-Step Implementation

### 1. Module Test Base
```php
<?php
// tests/Unit/Module/ModuleTestCase.php

namespace WHMCS\Module\YourModule\Tests\Unit\Module;

use PHPUnit\Framework\TestCase;

abstract class ModuleTestCase extends TestCase
{
    protected string $moduleName = 'YourModule';

    protected function callModuleFunction(string $function, array $params = []): mixed
    {
        if (!function_exists("{$this->moduleName}_{$function}")) {
            $this->markTestSkipped("Module function {$function} not found");
        }

        return call_user_func("{$this->moduleName}_{$function}", $params);
    }

    protected function setModuleConfig(array $config): void
    {
        // Set module configuration
        foreach ($config as $key => $value) {
            define("MODULE_{$key}", $value);
        }
    }
}
```

### 2. Module Test Examples
```php
<?php
// tests/Unit/Module/ProvisioningTest.php

namespace WHMCS\Module\YourModule\Tests\Unit\Module;

class ProvisioningTest extends ModuleTestCase
{
    public function testCreateAccount(): void
    {
        $params = [
            'accountid' => 1,
            'domain' => 'example.com',
            'username' => 'testuser',
            'password' => 'testpass123',
            'configoption1' => 'value1'
        ];

        $result = $this->callModuleFunction('create_account', $params);

        $this->assertTrue($result['success'] ?? false);
    }

    public function testSuspendAccount(): void
    {
        $params = [
            'accountid' => 1,
            'domain' => 'example.com'
        ];

        $result = $this->callModuleFunction('suspend_account', $params);

        $this->assertTrue($result['success'] ?? false);
    }

    public function testTerminateAccount(): void
    {
        $params = [
            'accountid' => 1,
            'domain' => 'example.com'
        ];

        $result = $this->callModuleFunction('terminate_account', $params);

        $this->assertTrue($result['success'] ?? false);
    }

    public function testChangePassword(): void
    {
        $params = [
            'accountid' => 1,
            'domain' => 'example.com',
            'password' => 'newpassword123'
        ];

        $result = $this->callModuleFunction('change_password', $params);

        $this->assertTrue($result['success'] ?? false);
    }
}
```

## Common Pitfalls and Solutions

| Pitfall | Solution |
|---------|----------|
| Module function not found | Check module is loaded |
| Missing WHMCS functions | Mock WHMCS dependencies |
| Configuration errors | Set up module config |

## Testing Checklist

- [ ] Test provisioning functions
- [ ] Test configuration options
- [ ] Test error handling
- [ ] Test edge cases

## Reference Links

- [WHMCS Module Development](https://developers.whmcs.com/module-development/)
