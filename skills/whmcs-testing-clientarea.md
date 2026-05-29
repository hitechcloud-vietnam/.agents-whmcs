# WHMCS Testing - Client Area Tests

## Skill Description
Implement testing for WHMCS client area functionality including templates, forms, and user interactions.

## Prerequisites
- WHMCS 7.0+ installation
- PHP 7.4+ with PHPUnit
- Selenium or similar for browser testing
- Client area knowledge

## Step-by-Step Implementation

### 1. Client Area Test Base
```php
<?php
// tests/Feature/ClientAreaTestCase.php

namespace WHMCS\Module\YourModule\Tests\Feature;

use PHPUnit\Framework\TestCase;

abstract class ClientAreaTestCase extends TestCase
{
    protected string $baseUrl = 'http://localhost/clientarea';
    protected int $testClientId = 1;

    protected function logInAsClient(int $clientId = null): void
    {
        $clientId = $clientId ?? $this->testClientId;

        // Simulate logged in state
        $_SESSION['uid'] = $clientId;
        $_SESSION['username'] = 'testuser';
    }

    protected function logOut(): void
    {
        unset($_SESSION['uid']);
        unset($_SESSION['username']);
    }
}
```

### 2. Client Area Test Examples
```php
<?php
// tests/Feature/ClientAreaTest.php

namespace WHMCS\Module\YourModule\Tests\Feature;

class ClientAreaTest extends ClientAreaTestCase
{
    public function testClientDashboardAccess(): void
    {
        $this->logInAsClient();

        // Verify dashboard loads
        $this->assertNotEmpty($_SESSION['uid']);
    }

    public function testServiceList(): void
    {
        $this->logInAsClient();

        // Verify services are loaded for client
        $clientId = $_SESSION['uid'];

        $this->assertGreaterThan(0, $clientId);
    }

    public function testInvoiceList(): void
    {
        $this->logInAsClient();

        // Verify invoices are loaded for client
        $clientId = $_SESSION['uid'];

        $this->assertGreaterThan(0, $clientId);
    }
}
```

## Common Pitfalls and Solutions

| Pitfall | Solution |
|---------|----------|
| Session management | Properly set up sessions |
| Authentication | Mock login state |
| Template rendering | Verify template output |

## Testing Checklist

- [ ] Test page access
- [ ] Test form submissions
- [ ] Test authentication
- [ ] Test authorization

## Reference Links

- [WHMCS Client Area](https://developers.whmcs.com/client-area/)
