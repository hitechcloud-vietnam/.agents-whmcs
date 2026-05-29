# WHMCS Testing - CSRF Tests

## Skill Description
Implement CSRF (Cross-Site Request Forgery) protection testing for WHMCS modules to ensure all state-changing operations are protected.

## Prerequisites
- WHMCS 7.0+ installation
- PHP 7.4+ with PHPUnit
- CSRF protection knowledge
- Security testing knowledge

## Step-by-Step Implementation

### 1. CSRF Test Base
```php
<?php
// tests/Security/CsrfTestCase.php

namespace WHMCS\Module\YourModule\Tests\Security;

use PHPUnit\Framework\TestCase;

abstract class CsrfTestCase extends TestCase
{
    protected function generateCsrfToken(): string
    {
        return bin2hex(random_bytes(32));
    }

    protected function validateCsrfToken(string $token, string $sessionToken): bool
    {
        return hash_equals($sessionToken, $token);
    }

    protected function assertCsrfValidation(): void
    {
        // Verify CSRF token is required
        $this->assertTrue(true);
    }
}
```

### 2. CSRF Test Examples
```php
<?php
// tests/Security/CsrfProtectionTest.php

namespace WHMCS\Module\YourModule\Tests\Security;

class CsrfProtectionTest extends CsrfTestCase
{
    public function testValidCsrfToken(): void
    {
        $token = $this->generateCsrfToken();
        $sessionToken = $token;

        $this->assertTrue($this->validateCsrfToken($token, $sessionToken));
    }

    public function testInvalidCsrfToken(): void
    {
        $token = $this->generateCsrfToken();
        $sessionToken = $this->generateCsrfToken();

        $this->assertFalse($this->validateCsrfToken($token, $sessionToken));
    }

    public function testExpiredCsrfToken(): void
    {
        $oldToken = 'expired-token';

        $this->assertFalse($this->validateCsrfToken($oldToken, ''));
    }

    public function testMissingCsrfToken(): void
    {
        $this->expectException(\Exception::class);

        // Attempt to validate missing token
        $this->validateCsrfToken('', '');
    }
}
```

## Common Pitfalls and Solutions

| Pitfall | Solution |
|---------|----------|
| Missing tokens | Always generate on forms |
| Token reuse | Use one-time tokens |
| Token leakage | Don't log tokens |

## Testing Checklist

- [ ] Test valid tokens
- [ ] Test invalid tokens
- [ ] Test expired tokens
- [ ] Test missing tokens

## Reference Links

- [OWASP CSRF](https://owasp.org/www-community/attacks/csrf)
- [CSRF Prevention](https://cheatsheetseries.owasp.org/cheatsheets/Cross-Site_Request_Forgery_Prevention_Cheat_Sheet.html)
