# WHMCS Testing - XSS Tests

## Skill Description
Implement XSS (Cross-Site Scripting) vulnerability testing for WHMCS modules to ensure all user input is properly sanitized.

## Prerequisites
- WHMCS 7.0+ installation
- PHP 7.4+ with PHPUnit
- XSS vulnerability knowledge
- Security testing knowledge

## Step-by-Step Implementation

### 1. XSS Test Base
```php
<?php
// tests/Security/XssTestCase.php

namespace WHMCS\Module\YourModule\Tests\Security;

use PHPUnit\Framework\TestCase;

abstract class XssTestCase extends TestCase
{
    protected array $xssPatterns = [
        '<script>alert(1)</script>',
        '<img src=x onerror=alert(1)>',
        '<svg onload=alert(1)>',
        'javascript:alert(1)',
        '<iframe src=javascript:alert(1)>',
        "';alert(1);//"
    ];

    protected function sanitize(string $input): string
    {
        return htmlspecialchars($input, ENT_QUOTES | ENT_HTML5, 'UTF-8');
    }

    protected function stripAllHtml(string $input): string
    {
        return strip_tags($input);
    }

    protected function isXssSafe(string $input): bool
    {
        $sanitized = $this->sanitize($input);
        return strpos($sanitized, '<script>') === false
            && strpos($sanitized, 'javascript:') === false
            && strpos($sanitized, 'onerror') === false;
    }
}
```

### 2. XSS Test Examples
```php
<?php
// tests/Security/XssVulnerabilityTest.php

namespace WHMCS\Module\YourModule\Tests\Security;

class XssVulnerabilityTest extends XssTestCase
{
    public function testScriptTagBlocked(): void
    {
        $input = '<script>alert("XSS")</script>';
        $output = $this->sanitize($input);

        $this->assertStringNotContainsString('<script>', $output);
    }

    public function testImgOnerrorBlocked(): void
    {
        $input = '<img src=x onerror=alert(1)>';
        $output = $this->sanitize($input);

        $this->assertStringNotContainsString('onerror', $output);
    }

    public function testJavascriptProtocolBlocked(): void
    {
        $input = '<a href="javascript:alert(1)">Click</a>';
        $output = $this->sanitize($input);

        $this->assertStringNotContainsString('javascript:', $output);
    }

    public function testSvgOnloadBlocked(): void
    {
        $input = '<svg onload=alert(1)>';
        $output = $this->sanitize($input);

        $this->assertStringNotContainsString('onload', $output);
    }

    public function testAllPatternsAreSafe(): void
    {
        foreach ($this->xssPatterns as $pattern) {
            $this->assertTrue($this->isXssSafe($this->sanitize($pattern)));
        }
    }
}
```

## Common Pitfalls and Solutions

| Pitfall | Solution |
|---------|----------|
| Incomplete sanitization | Use comprehensive escaping |
| Stored XSS | Sanitize on output |
| Reflected XSS | Sanitize on input and output |

## Testing Checklist

- [ ] Test script tags
- [ ] Test event handlers
- [ ] Test javascript protocol
- [ ] Test SVG elements

## Reference Links

- [OWASP XSS](https://owasp.org/www-community/attacks/xss/)
- [XSS Prevention](https://cheatsheetseries.owasp.org/cheatsheets/Cross_Site_Scripting_Prevention_Cheat_Sheet.html)
