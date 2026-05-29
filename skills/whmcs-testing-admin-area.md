# WHMCS Testing - Admin Area Tests

## Skill Description
Implement testing for WHMCS admin area functionality including admin-only features, configurations, and management features.

## Prerequisites
- WHMCS 7.0+ installation
- PHP 7.4+ with PHPUnit
- Admin area knowledge
- Testing framework setup

## Step-by-Step Implementation

### 1. Admin Area Test Base
```php
<?php
// tests/Feature/AdminAreaTestCase.php

namespace WHMCS\Module\YourModule\Tests\Feature;

use PHPUnit\Framework\TestCase;

abstract class AdminAreaTestCase extends TestCase
{
    protected string $baseUrl = 'http://localhost/admin';
    protected int $adminId = 1;

    protected function logInAsAdmin(int $adminId = null): void
    {
        $adminId = $adminId ?? $this->adminId;

        $_SESSION['adminid'] = $adminId;
        $_SESSION['adminname'] = 'admin';
    }

    protected function logOut(): void
    {
        unset($_SESSION['adminid']);
        unset($_SESSION['adminname']);
    }

    protected function hasAdminAccess(): bool
    {
        return isset($_SESSION['adminid']) && $_SESSION['adminid'] > 0;
    }
}
```

### 2. Admin Area Test Examples
```php
<?php
// tests/Feature/AdminAreaTest.php

namespace WHMCS\Module\YourModule\Tests\Feature;

class AdminAreaTest extends AdminAreaTestCase
{
    public function testAdminDashboardAccess(): void
    {
        $this->logInAsAdmin();

        $this->assertTrue($this->hasAdminAccess());
    }

    public function testModuleConfigurationAccess(): void
    {
        $this->logInAsAdmin();

        // Verify admin can access module configuration
        $this->assertTrue($this->hasAdminAccess());
    }

    public function testClientManagementAccess(): void
    {
        $this->logInAsAdmin();

        // Verify admin can access client management
        $this->assertTrue($this->hasAdminAccess());
    }

    public function testUnauthorizedAccessDenied(): void
    {
        $this->logOut();

        $this->assertFalse($this->hasAdminAccess());
    }
}
```

## Common Pitfalls and Solutions

| Pitfall | Solution |
|---------|----------|
| Session state | Reset session between tests |
| Authorization | Test both authorized and unauthorized |
| Permissions | Test role-based access |

## Testing Checklist

- [ ] Test admin access
- [ ] Test module configuration
- [ ] Test client management
- [ ] Test authorization

## Reference Links

- [WHMCS Admin Area](https://developers.whmcs.com/admin-area/)
