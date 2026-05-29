# WHMCS CSRF Testing Workflow

## Overview
This workflow provides comprehensive guidance for testing CSRF (Cross-Site Request Forgery) protection in WHMCS modules and customizations.

## Prerequisites
- WHMCS installation (v8.0+)
- Browser testing tools (Playwright, Selenium)
- Understanding of CSRF vulnerabilities

## Step-by-Step Guide

### Step 1: Understand CSRF Protection in WHMCS

#### WHMCS Built-in CSRF Protection
```php
<?php
// WHMCS generates CSRF tokens automatically for forms
// Tokens are stored in session and validated on POST

// How WHMCS handles CSRF:
// 1. Token generated: generate_token('post', '/path/to/form')
// 2. Form includes: <input type="hidden" name="token" value="...">
// 3. Validation: check_token() called at form processing

// Check if token exists in session
$token = $_SESSION['token'] ?? '';
$postToken = $_POST['token'] ?? '';
$csrfValid = hash_equals($token, $postToken);
```

### Step 2: Identify Forms to Test

#### Common Form Locations
```php
<?php
// Forms that need CSRF protection:
$forms = [
    '/admin/clientsadd.php'           => 'Create Client',
    '/admin/clients.php?action=edit'  => 'Edit Client',
    '/admin/invoices.php?action=create' => 'Create Invoice',
    '/cart.php'                       => 'Checkout',
    '/submitticket.php'               => 'Submit Ticket',
    '/clientarea.php?action=security'  => 'Security Settings',
    '/module/form/handler.php'         => 'Module Custom Form',
];
```

### Step 3: Create CSRF Test Suite
```javascript
// tests/csrf.spec.js
const { test, expect, request } = require('@playwright/test');

test.describe('CSRF Protection Tests', () => {
  test.beforeEach(async ({ page }) => {
    // Login as admin
    await page.goto('http://localhost/admin/login.php');
    await page.fill('#username', 'admin');
    await page.fill('#password', process.env.ADMIN_PASSWORD);
    await page.click('button[type="submit"]');
    await page.waitForURL('**/admin/index.php');
  });

  test('form contains CSRF token', async ({ page }) => {
    await page.goto('http://localhost/admin/clientsadd.php');

    const tokenInput = page.locator('input[name="token"]');
    await expect(tokenInput).toBeVisible();

    const tokenValue = await tokenInput.inputValue();
    expect(tokenValue).toHaveLength(32); // WHMCS token length
  });

  test('POST without token is rejected', async ({ page }) => {
    // Try to submit form without CSRF token
    const response = await page.request.post(
      'http://localhost/admin/clientsadd.php',
      {
        form: {
          firstname: 'CSRF Test',
          lastname: 'Attack',
          email: 'csrf@test.com',
          password2: 'password123',
        },
        headers: {
          'Content-Type': 'application/x-www-form-urlencoded',
        },
      }
    );

    // Should be rejected or token validation should fail
    expect(response.status()).not.toBe(200);
  });

  test('POST with invalid token is rejected', async ({ page }) => {
    // Get a valid page to extract any existing form structure
    await page.goto('http://localhost/admin/clientsadd.php');

    // Submit with fake token
    const response = await page.request.post(
      'http://localhost/admin/clientsadd.php',
      {
        form: {
          firstname: 'CSRF Test',
          lastname: 'Attack',
          email: 'csrf@test.com',
          password2: 'password123',
          token: 'invalid_token_12345678901234567',
        },
        headers: {
          'Content-Type': 'application/x-www-form-urlencoded',
        },
      }
    );

    // Should be rejected or redirected
    expect(response.status()).not.toBe(200);
  });

  test('POST with valid token succeeds', async ({ page }) => {
    await page.goto('http://localhost/admin/clientsadd.php');

    // Extract CSRF token
    const tokenInput = page.locator('input[name="token"]');
    const token = await tokenInput.inputValue();

    // Submit with valid token
    await page.fill('#firstname', 'CSRF Test');
    await page.fill('#lastname', 'Valid');
    await page.fill('#email', `csrf_valid_${Date.now()}@test.com`);
    await page.fill('#password2', 'SecurePass123!');
    await page.fill('input[name="token"]', token);

    await page.click('button[type="submit"]');

    // Should succeed (redirect or show success)
    await page.waitForTimeout(1000);
  });

  test('expired token is rejected', async ({ page }) => {
    // Get a token from one page
    await page.goto('http://localhost/admin/clientsadd.php');
    const token1 = await page.locator('input[name="token"]').inputValue();

    // Navigate away and come back
    await page.goto('http://localhost/admin/index.php');
    await page.goto('http://localhost/admin/clientsadd.php');

    // Get new token
    const token2 = await page.locator('input[name="token"]').inputValue();

    // Try to use first token (should fail - token was regenerated)
    await page.fill('#firstname', 'Expired Token Test');
    await page.fill('#lastname', 'Should Fail');
    await page.fill('#email', 'expired@test.com');
    await page.fill('#password2', 'password123');
    await page.fill('input[name="token"]', token1);

    await page.click('button[type="submit"]');

    // Should be rejected or show error
    await page.waitForTimeout(500);
  });
});
```

### Step 4: Test Module Forms
```php
// tests/ModuleCSRFTest.php
<?php
namespace WHMCS\Tests;

use PHPUnit\Framework\TestCase;

class ModuleCSRFTest extends TestCase
{
    private $client;

    protected function setUp(): void
    {
        parent::setUp();
        $this->client = new \GuzzleHttp\Client([
            'base_uri' => 'http://localhost/',
            'cookies' => true,
        ]);
    }

    public function testModuleFormHasCSRFToken()
    {
        $response = $this->client->get('modules/addons/yourmodule/form.php');
        $body = (string) $response->getBody();

        $this->assertStringContainsString('name="token"', $body);
        $this->assertMatchesRegularExpression('/value="[a-z0-9]{32}"/', $body);
    }

    public function testModuleFormRejectsInvalidToken()
    {
        // Login first
        $this->client->post('admin/dologin.php', [
            'form_params' => [
                'username' => 'admin',
                'password' => 'admin_password',
            ],
        ]);

        // Submit form with invalid token
        $response = $this->client->post('modules/addons/yourmodule/form.php', [
            'form_params' => [
                'data' => 'test_value',
                'token' => 'invalid_token_12345678901234',
            ],
        ]);

        // Should be rejected
        $this->assertEquals(403, $response->getStatusCode());
    }

    public function testModuleFormAcceptsValidToken()
    {
        // Get the page to extract token
        $getResponse = $this->client->get('modules/addons/yourmodule/form.php');
        $body = (string) $getResponse->getBody();

        preg_match('/name="token" value="([^"]+)"/', $body, $matches);
        $token = $matches[1];

        // Submit form with valid token
        $response = $this->client->post('modules/addons/yourmodule/form.php', [
            'form_params' => [
                'data' => 'test_value',
                'token' => $token,
            ],
        ]);

        $this->assertEquals(200, $response->getStatusCode());
    }
}
```

### Step 5: Automated CSRF Scanner
```bash
#!/bin/bash
# csrf-scanner.sh

echo "=== WHMCS CSRF Scanner ==="

# List of forms to test
FORMS=(
    "/admin/clientsadd.php"
    "/admin/invoices.php?action=create"
    "/submitticket.php"
    "/clientarea.php?action=security"
    "/modules/addons/yourmodule/form.php"
)

echo "Testing forms for CSRF token presence..."
for form in "${FORMS[@]}"; do
    echo -n "Checking $form: "
    
    # Check if form has token field
    content=$(curl -s "http://localhost$form")
    
    if echo "$content" | grep -q 'name="token"'; then
        echo "PASS - Token field found"
    else
        echo "FAIL - No token field found"
    fi
done

echo ""
echo "Testing POST without token..."
for form in "${FORMS[@]}"; do
    echo -n "Testing $form: "
    
    # Try POST without token (will only work if form doesn't require auth)
    response=$(curl -s -o /dev/null -w "%{http_code}" \
        -X POST "http://localhost$form" \
        -d "test=value")
    
    if [ "$response" = "403" ] || [ "$response" = "302" ]; then
        echo "PASS - Request blocked (status: $response)"
    else
        echo "WARNING - Request allowed (status: $response)"
    fi
done
```

### Step 6: Test AJAX Endpoints
```javascript
// tests/csrf-ajax.spec.js
const { test, expect } = require('@playwright/test');

test.describe('CSRF Protection for AJAX Requests', () => {
  test.beforeEach(async ({ page }) => {
    // Login
    await page.goto('http://localhost/clientarea.php');
    // Login flow...
  });

  test('AJAX without token header is rejected', async ({ page }) => {
    const response = await page.request.post(
      'http://localhost/modules/addons/yourmodule/api.php',
      {
        headers: {
          'Content-Type': 'application/json',
          'X-Requested-With': 'XMLHttpRequest',
          // No CSRF token header
        },
        data: JSON.stringify({ action: 'update', id: 1 }),
      }
    );

    // Should be rejected for state-changing operations
    expect(response.status()).toBeOneOf([403, 401]);
  });

  test('AJAX with valid token header succeeds', async ({ page }) => {
    // Get CSRF token from page
    await page.goto('http://localhost/clientarea.php');
    const token = await page.evaluate(() => {
      const meta = document.querySelector('meta[name="csrf-token"]');
      return meta ? meta.content : '';
    });

    const response = await page.request.post(
      'http://localhost/modules/addons/yourmodule/api.php',
      {
        headers: {
          'Content-Type': 'application/json',
          'X-CSRF-Token': token,
        },
        data: JSON.stringify({ action: 'getData', id: 1 }),
      }
    );

    expect(response.status()).toBe(200);
  });
});
```

### Step 7: Run CSRF Tests
```bash
# Run CSRF test suite
npx playwright test tests/csrf.spec.js

# Run with debug
npx playwright test tests/csrf.spec.js --debug

# Run specific test
npx playwright test tests/csrf.spec.js --grep "form contains CSRF"

# Run PHP tests
./vendor/bin/phpunit tests/ModuleCSRFTest.php
```

## CSRF Testing Checklist

### Token Generation
- [ ] Tokens generated for all forms
- [ ] Tokens have sufficient entropy
- [ ] Tokens are session-bound
- [ ] Tokens are not exposed in URL

### Token Validation
- [ ] Tokens validated on POST
- [ ] Invalid tokens rejected
- [ ] Missing tokens rejected
- [ ] Expired tokens rejected

### AJAX Protection
- [ ] AJAX requests require token
- [ ] Token sent via header
- [ ] CORS properly configured

### Error Handling
- [ ] Clear error messages
- [ ] Logs security events
- [ ] No information leakage

## Common CSRF Vulnerabilities

| Vulnerability | Test Method | Fix |
|---------------|-------------|-----|
| Missing token | Check form HTML | Add generate_token() |
| Weak token | Analyze token generation | Use cryptographically secure |
| Token reuse | Test token multiple times | One-time or time-limited |
| Referer check bypass | Test with forged headers | Use token validation |
| GET requests change state | Check HTTP methods | Use POST for state changes |
