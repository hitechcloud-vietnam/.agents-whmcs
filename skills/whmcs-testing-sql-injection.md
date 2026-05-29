# WHMCS Testing - SQL Injection Tests

## Skill Description
Implement SQL injection prevention testing for WHMCS modules to ensure all database queries are protected.

## Prerequisites
- WHMCS 7.0+ installation
- PHP 7.4+ with PHPUnit
- SQL injection knowledge
- Security testing knowledge

## Step-by-Step Implementation

### 1. SQL Injection Test Base
```php
<?php
// tests/Security/SqlInjectionTestCase.php

namespace WHMCS\Module\YourModule\Tests\Security;

use PHPUnit\Framework\TestCase;

abstract class SqlInjectionTestCase extends TestCase
{
    protected array $injectionPatterns = [
        "' OR '1'='1",
        "'; DROP TABLE users; --",
        "1' UNION SELECT * FROM users--",
        "1' OR '1'='1' /*",
        "admin'--",
        "1' AND '1'='1",
        "1; SELECT * FROM users"
    ];

    protected function isQuerySafe(string $sql, array $params): bool
    {
        // Check if using prepared statements
        $placeholderCount = substr_count($sql, '?');

        return $placeholderCount === count($params);
    }
}
```

### 2. SQL Injection Test Examples
```php
<?php
// tests/Security/SqlInjectionVulnerabilityTest.php

namespace WHMCS\Module\YourModule\Tests\Security;

class SqlInjectionVulnerabilityTest extends SqlInjectionTestCase
{
    public function testPreparedStatementWithOr(): void
    {
        $sql = "SELECT * FROM users WHERE email = ?";
        $params = ["' OR '1'='1"];

        $this->assertTrue($this->isQuerySafe($sql, $params));
    }

    public function testPreparedStatementWithDrop(): void
    {
        $sql = "SELECT * FROM users WHERE id = ?";
        $params = ["1; DROP TABLE users"];

        $this->assertTrue($this->isQuerySafe($sql, $params));
    }

    public function testPreparedStatementWithUnion(): void
    {
        $sql = "SELECT * FROM users WHERE email = ?";
        $params = ["test@example.com' UNION SELECT * FROM users--"];

        $this->assertTrue($this->isQuerySafe($sql, $params));
    }

    public function testAllPatternsArePrevented(): void
    {
        foreach ($this->injectionPatterns as $pattern) {
            $sql = "SELECT * FROM users WHERE email = ?";
            $params = [$pattern];

            $this->assertTrue(
                $this->isQuerySafe($sql, $params),
                "Query should be safe with pattern: {$pattern}"
            );
        }
    }
}
```

## Common Pitfalls and Solutions

| Pitfall | Solution |
|---------|----------|
| String concatenation | Always use prepared statements |
| Direct interpolation | Use parameterized queries |
| Dynamic column names | Whitelist column names |

## Testing Checklist

- [ ] Test with OR patterns
- [ ] Test with UNION patterns
- [ ] Test with DROP patterns
- [ ] Test with comments

## Reference Links

- [OWASP SQL Injection](https://owasp.org/www-community/attacks/SQL_Injection)
- [SQL Injection Prevention](https://cheatsheetseries.owasp.org/cheatsheets/SQL_Injection_Prevention_Cheat_Sheet.html)
