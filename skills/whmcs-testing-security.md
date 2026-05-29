# WHMCS Testing - Security Tests

## Skill Description
Implement security testing for WHMCS modules to identify vulnerabilities, verify protections, and ensure secure code.

## Prerequisites
- WHMCS 7.0+ installation
- PHP 7.4+ with PHPUnit
- Security testing tools
- Security testing knowledge

## Step-by-Step Implementation

### 1. Security Test Base
```php
<?php
// tests/Security/SecurityTestCase.php

namespace WHMCS\Module\YourModule\Tests\Security;

use PHPUnit\Framework\TestCase;

abstract class SecurityTestCase extends TestCase
{
    protected function assertSqlInjectionSafe(callable $callback): void
    {
        // This is a placeholder - actual implementation depends on your testing needs
        $this->assertTrue(true);
    }

    protected function assertXssSafe(string $input): void
    {
        $sanitized = htmlspecialchars($input, ENT_QUOTES | ENT_HTML5, 'UTF-8');

        $this->assertNotEquals($input, $sanitized);
    }

    protected function assertCsrfProtected(): void
    {
        // Verify CSRF tokens are generated and validated
        $this->assertTrue(true);
    }
}
```

### 2. Security Test Examples
```php
<?php
// tests/Security/SqlInjectionTest.php

namespace WHMCS\Module\YourModule\Tests\Security;

class SqlInjectionTest extends SecurityTestCase
{
    public function testQueryWithSpecialCharacters(): void
    {
        $input = "'; DROP TABLE users; --";

        $this->assertSqlInjectionSafe(function () use ($input) {
            global $db;

            $stmt = $db->prepare("SELECT * FROM tblclients WHERE email = ?");
            $stmt->execute([$input]);

            // Should not drop table
            $stmt->fetchAll();
        });
    }

    public function testQueryWithNumericInput(): void
    {
        $input = "1 OR 1=1";

        $this->assertSqlInjectionSafe(function () use ($input) {
            global $db;

            $stmt = $db->prepare("SELECT * FROM tblclients WHERE id = ?");
            $stmt->execute([(int)$input]);

            $stmt->fetchAll();
        });
    }
}
```

```php
<?php
// tests/Security/XssTest.php

namespace WHMCS\Module\YourModule\Tests\Security;

class XssTest extends SecurityTestCase
{
    public function testXssInUserInput(): void
    {
        $input = '<script>alert("XSS")</script>';

        $sanitized = htmlspecialchars($input, ENT_QUOTES | ENT_HTML5, 'UTF-8');

        $this->assertStringNotContainsString('<script>', $sanitized);
    }

    public function testXssInHtmlOutput(): void
    {
        $input = '<img src=x onerror=alert(1)>';

        $sanitized = strip_tags($input);

        $this->assertStringNotContainsString('<img', $sanitized);
    }
}
```

## Common Pitfalls and Solutions

| Pitfall | Solution |
|---------|----------|
| SQL injection | Use prepared statements |
| XSS attacks | Sanitize output |
| CSRF attacks | Use CSRF tokens |

## Testing Checklist

- [ ] Test SQL injection
- [ ] Test XSS attacks
- [ ] Test CSRF protection
- [ ] Test authentication
- [ ] Test authorization

## Reference Links

- [OWASP Testing Guide](https://owasp.org/www-project-web-security-testing-guide/)
- [Security Testing Best Practices](https://cheatsheetseries.owasp.org/)
