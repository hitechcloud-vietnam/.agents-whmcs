# WHMCS Integration Testing Workflow

## Purpose

Complete guide to testing WHMCS integrations including modules, plugins, APIs, and webhooks. Covers unit testing, integration testing, end-to-end testing, automated testing frameworks, and CI/CD integration.

## Prerequisites

- WHMCS 7.0+ installation
- PHPUnit 9.0+ installed
- Testing environment (staging)
- Access to test data/accounts
- Composer installed

## Workflow Steps

### Step 1: Setting Up Testing Environment

```bash
# ===========================================
# Test Environment Setup
# ===========================================

# Install PHPUnit via Composer
composer require --dev phpunit/phpunit:^9.0 mockery/mockery:^1.3

# Create test directory structure
mkdir -p tests/
mkdir -p tests/Unit/
mkdir -p tests/Integration/
mkdir -p tests/Feature/
mkdir -p tests/fixtures/
mkdir -p tests/mocks/
mkdir -p tests/bootstrap.php
mkdir -p tests/phpunit.xml
```

```xml
<!-- tests/phpunit.xml -->
<?xml version="1.0" encoding="UTF-8"?>
<phpunit xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:noNamespaceSchemaLocation="https://schema.phpunit.de/9.0/phpunit.xsd"
         bootstrap="tests/bootstrap.php"
         colors="true"
         stopOnFailure="false"
         cacheResult="false">
    <testsuites>
        <testsuite name="Unit">
            <directory>tests/Unit</directory>
        </testsuite>
        <testsuite name="Integration">
            <directory>tests/Integration</directory>
        </testsuite>
        <testsuite name="Feature">
            <directory>tests/Feature</directory>
        </testsuite>
    </testsuites>
    <php>
        <env name="WHMCS_TEST_MODE" value="true"/>
        <env name="WHMCS_LICENSE_KEY" value="test_license"/>
        <server name="DOCUMENT_ROOT" value="/var/www/html/whmcs"/>
    </php>
</phpunit>
```

### Step 2: Creating Test Infrastructure

```php
<?php
// tests/TestCase.php

namespace WHMCS\Tests;

use PHPUnit\Framework\TestCase as BaseTestCase;
use Mockery;

abstract class TestCase extends BaseTestCase
{
    protected $mockWhmcsApi;
    protected $testDatabase;

    protected function setUp(): void
    {
        parent::setUp();

        // Set up test configuration
        $_SESSION = [];
        $_POST = [];
        $_GET = [];
        $_COOKIE = [];
    }

    protected function tearDown(): void
    {
        Mockery::close();
        parent::tearDown();
    }

    protected function createMockService(array $overrides = []): array
    {
        return array_merge([
            'serviceid' => 1,
            'userid' => 1,
            'domain' => 'test.example.com',
            'packageid' => 1,
            'server' => 1,
            'domainstatus' => 'Active',
            'subscription_id' => 'sub_123456',
            'username' => 'testuser',
            'password' => 'encrypted_password',
            'notes' => '',
        ], $overrides);
    }

    protected function createMockClient(array $overrides = []): array
    {
        return array_merge([
            'id' => 1,
            'email' => 'test@example.com',
            'firstname' => 'Test',
            'lastname' => 'User',
            'companyname' => 'Test Company',
            'datecreated' => date('Y-m-d'),
            'status' => 'Active',
        ], $overrides);
    }

    protected function mockApiResponse(array $data, int $status = 200): array
    {
        return [
            'status' => $status,
            'data' => $data,
            'headers' => ['Content-Type' => 'application/json'],
        ];
    }

    protected function simulateRequest(
        string $method = 'GET',
        string $path = '/',
        array $data = [],
        array $headers = []
    ): void {
        $_SERVER['REQUEST_METHOD'] = $method;
        $_SERVER['REQUEST_URI'] = $path;
        $_SERVER['CONTENT_TYPE'] = 'application/json';
        $_SERVER['REMOTE_ADDR'] = '127.0.0.1';

        foreach ($headers as $key => $value) {
            $_SERVER['HTTP_' . str_replace('-', '_', strtoupper($key))] = $value;
        }

        if ($method === 'POST' || $method === 'PUT') {
            $_POST = $data;
        } else {
            $_GET = $data;
        }
    }
}
```

### Step 3: Unit Testing Modules

```php
<?php
// tests/Unit/ProvisioningModuleTest.php

namespace WHMCS\Tests\Unit;

use WHMCS\Tests\TestCase;

class ProvisioningModuleTest extends TestCase
{
    private $moduleName = 'yourprovider';
    private $params;

    protected function setUp(): void
    {
        parent::setUp();

        $this->params = [
            'serverusername] => 'test_server_user',
            'serverpassword' => 'test_server_pass',
            'serverhostname' => 'api.example.com',
            'serverhttpprefix' => 'https',
            'serviceid' => 1,
            'userid' => 1,
            'domain' => 'test.example.com',
            'configoption1' => 'starter',
            'configoption2' => 'ubuntu-22.04',
        ];
    }

    /**
     * Test module metadata returns required structure
     */
    public function testMetaDataReturnsRequiredFields(): void
    {
        $metaData = yourprovider_MetaData();

        $this->assertIsArray($metaData);
        $this->assertArrayHasKey('DisplayName', $metaData);
        $this->assertArrayHasKey('APIVersion', $metaData);
        $this->assertEquals('1.0', $metaData['APIVersion']);
    }

    /**
     * Test config options structure
     */
    public function testConfigOptionsReturnsValidStructure(): void
    {
        $configOptions = yourprovider_ConfigOptions($this->params);

        $this->assertIsArray($configOptions);
        $this->assertArrayHasKey('Plan', $configOptions);
        $this->assertEquals('dropdown', $configOptions['Plan']['Type']);
    }

    /**
     * Test successful account creation
     */
    public function testCreateAccountSuccess(): void
    {
        $result = yourprovider_CreateAccount($this->params);

        $this->assertEquals('success', $result);
    }

    /**
     * Test account creation failure handling
     */
    public function testCreateAccountFailure(): void
    {
        $params = $this->params;
        $params['domain'] = ''; // Missing domain

        $result = yourprovider_CreateAccount($params);

        $this->assertStringStartsWith('Error:', $result);
    }

    /**
     * Test client area returns correct structure
     */
    public function testClientAreaReturnsValidArray(): void
    {
        $result = yourprovider_ClientArea($this->params);

        $this->assertIsArray($result);
        $this->assertArrayHasKey('pagetitle', $result);
        $this->assertArrayHasKey('templatefile', $result);
        $this->assertArrayHasKey('vars', $result);
    }
}
```

### Step 4: Integration Testing

```php
<?php
// tests/Integration/APIIntegrationTest.php

namespace WHMCS\Tests\Integration;

use WHMCS\Tests\TestCase;

class APIIntegrationTest extends TestCase
{
    private $testWhmcsUrl;
    private $testApiKey;
    private $testApiSecret;

    protected function setUp(): void
    {
        parent::setUp();

        $configFile = __DIR__ . '/../config/test_config.php';
        if (file_exists($configFile)) {
            $config = include $configFile;
            $this->testWhmcsUrl = $config['whmcs_url'];
            $this->testApiKey = $config['api_identifier'];
            $this->testApiSecret = $config['api_secret'];
        } else {
            $this->markTestSkipped('Test configuration not found');
        }
    }

    /**
     * Test client creation via API
     */
    public function testCreateClient(): void
    {
        $clientData = [
            'firstname' => 'Integration',
            'lastname' => 'Test',
            'email' => 'integration_' . time() . '@test.com',
            'country' => 'US',
            'password' => 'TestPass123!',
        ];

        $response = $this->apiCall('AddClient', $clientData);

        $this->assertEquals('success', $response['result']);
        $this->assertArrayHasKey('clientid', $response);

        return $response['clientid'];
    }

    /**
     * Test getting client details
     */
    public function testGetClient(): void
    {
        $clientId = $this->testCreateClient();

        $response = $this->apiCall('GetClient', [
            'clientid' => $clientId,
        ]);

        $this->assertEquals('success', $response['result']);
        $this->assertEquals($clientId, $response['id']);
    }

    /**
     * Test invoice creation and retrieval
     */
    public function testInvoiceOperations(): void
    {
        $clientId = $this->testCreateClient();

        $response = $this->apiCall('CreateInvoice', [
            'userid' => $clientId,
            'items' => json_encode([
                ['description' => 'Test Service', 'amount' => 9.99],
            ]),
        ]);

        $this->assertEquals('success', $response['result']);
        $this->assertArrayHasKey('invoiceid', $response);

        return $response['invoiceid'];
    }

    private function apiCall(string $action, array $params = []): array
    {
        $postData = array_merge($params, [
            'identifier' => $this->testApiKey,
            'secret' => $this->testApiSecret,
            'responsetype' => 'json',
        ]);

        if ($action) {
            $postData['action'] = $action;
        }

        $ch = curl_init();
        curl_setopt_array($ch, [
            CURLOPT_URL => $this->testWhmcsUrl . '/includes/api.php',
            CURLOPT_POST => true,
            CURLOPT_POSTFIELDS => http_build_query($postData),
            CURLOPT_RETURNTRANSFER => true,
            CURLOPT_TIMEOUT => 30,
        ]);

        $response = curl_exec($ch);
        curl_close($ch);

        return json_decode($response, true) ?? [];
    }
}
```

### Step 5: CI/CD Integration

```yaml
# .github/workflows/whmcs-testing.yml

name: WHMCS Integration Tests

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]

jobs:
  unit-tests:
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@v3

      - name: Setup PHP
        uses: shivammathur/setup-php@v2
        with:
          php-version: '8.1'
          extensions: json, curl, pdo, pdo_mysql

      - name: Install Dependencies
        run: composer install --no-interaction

      - name: Run Unit Tests
        run: vendor/bin/phpunit --testsuite Unit

  integration-tests:
    runs-on: ubuntu-latest
    services:
      mysql:
        image: mysql:8.0
        env:
          MYSQL_ROOT_PASSWORD: root
          MYSQL_DATABASE: whmcs_test
        ports:
          - 3306:3306

    steps:
      - uses: actions/checkout@v3
      - name: Setup PHP
        uses: shivammathur/setup-php@v2
        with:
          php-version: '8.1'
      - name: Run Integration Tests
        run: vendor/bin/phpunit --testsuite Integration
```

## Verification Checklist

```
Test Suite:
□ All module functions tested
□ Success and failure paths covered
□ Mock dependencies configured
□ Test data fixtures available

Integration:
□ API calls to test WHMCS instance
□ Database operations tested
□ Webhook handling tested

CI/CD:
□ GitHub/GitLab actions configured
□ Tests run on commit/pull request
□ Coverage reporting enabled
```

## WHMCS ClassDocs References

- [WHMCS Testing Guidelines](https://developers.whmcs.com/advanced/testing-guidelines/)
- [PHPUnit Documentation](https://phpunit.de/documentation.html)
- [API Testing Reference](https://developers.whmcs.com/api-reference/)
