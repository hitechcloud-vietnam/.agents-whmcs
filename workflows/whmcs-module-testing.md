# WHMCS Module Testing Workflow
# Version: 1.0 | Updated: 2026-05-28

## Purpose

Comprehensive testing guide for WHMCS modules.

## Test Types

### 1. Unit Testing

```php
<?php
class ApiClientTest {
    public function testCreateInstance(): void {
        $api = new ApiClient(['mock' => true]);

        $result = $api->createInstance([
            'name' => 'test-instance',
            'plan' => 'small',
        ]);

        $this->assertTrue($result['success']);
        $this->assertArrayHasKey('instance_id', $result);
    }

    public function testValidateInput(): void {
        $validator = new Validator();

        $result = $validator->validate([
            'name' => 'test',
            'email' => 'invalid-email',
        ]);

        $this->assertFalse($result['valid']);
        $this->assertArrayHasKey('email', $result['errors']);
    }
}
```

### 2. Integration Testing

```php
function testCreateAccountFlow(): void {
    $params = [
        'domain' => 'test.com',
        'username' => 'testuser',
        'password' => 'password123',
    ];

    $result = createAccount($params);

    $this->assertEquals('success', $result);

    // Verify in database
    $service = Capsule::table('tblhosting')
        ->where('domain', 'test.com')
        ->first();

    $this->assertNotNull($service);
}
```

### 3. API Mock Testing

```php
function testWithMockedApi(): void {
    $mock = $this->createMock(ApiClient::class);
    $mock->method('createInstance')
        ->willReturn(['success' => true, 'id' => '123']);

    $service = new ServiceManager($mock);

    $result = $service->createAccount([
        'domain' => 'test.com',
    ]);

    $this->assertTrue($result);
}
```

### 4. Frontend Testing

```javascript
// Admin interface tests
async function testAdminFormSubmit() {
    await page.fill('#name', 'Test Module');
    await page.fill('#api_key', 'secret_key');
    await page.click('button[type="submit"]');

    expect(await page.locator('.success-message')).toBeVisible();
}
```

## Test Coverage Targets

| Component | Target |
|------------|--------|
| Core functions | 90% |
| API client | 85% |
| Templates | 70% |
| Hooks | 80% |

## Continuous Integration

```yaml
# .github/workflows/test.yml
name: Test
on: [push, pull_request]
jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v2
      - name: Run Tests
        run: |
          composer install
          ./vendor/bin/phpunit
```
