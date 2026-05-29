# WHMCS Regression Testing Workflow

## Overview
This workflow provides comprehensive guidance for regression testing WHMCS modules after changes.

## Prerequisites
- WHMCS installation (v8.0+)
- Test suite
- CI/CD pipeline

## Step-by-Step Guide

### Step 1: Create Regression Test Suite
```php
// tests/RegressionTest.php
<?php
namespace WHMCS\Tests;

use PHPUnit\Framework\TestCase;

class RegressionTest extends TestCase
{
    public function setUp(): void
    {
        parent::setUp();
        $this->module = new \WHMCS\Module\YourModule\Module();
    }

    public function testModuleActivation()
    {
        $result = $this->module->activate();
        $this->assertTrue($result['success']);
    }

    public function testModuleDeactivation()
    {
        $result = $this->module->deactivate();
        $this->assertTrue($result['success']);
    }

    public function testConfigurationSave()
    {
        $config = [
            'api_key' => 'test_key',
            'setting1' => 'value1',
        ];

        $result = $this->module->saveConfig($config);
        $this->assertTrue($result['success']);

        // Verify saved
        $saved = $this->module->getConfig();
        $this->assertEquals($config['api_key'], $saved['api_key']);
    }

    public function testClientSync()
    {
        $clientId = 1;
        $result = $this->module->syncClient($clientId);
        $this->assertTrue($result['success']);
    }

    public function testOrderFulfillment()
    {
        $orderId = 1;
        $result = $this->module->fulfillOrder($orderId);
        $this->assertTrue($result['success']);
    }

    public function testServiceTermination()
    {
        $serviceId = 1;
        $result = $this->module->terminateService($serviceId);
        $this->assertTrue($result['success']);
    }

    public function testInvoiceGeneration()
    {
        $invoiceId = 1;
        $result = $this->module->generateInvoiceItems($invoiceId);
        $this->assertTrue($result['success']);
    }

    public function testAdminAreaOutput()
    {
        ob_start();
        $this->module->outputAdminArea();
        $output = ob_get_clean();

        $this->assertNotEmpty($output);
    }

    public function testWidgetOutput()
    {
        ob_start();
        $this->module->outputWidget();
        $output = ob_get_clean();

        $this->assertNotEmpty($output);
    }
}
```

### Step 2: Run Regression Tests
```bash
# Run all regression tests
./vendor/bin/phpunit tests/RegressionTest.php

# Run with verbose output
./vendor/bin/phpunit tests/RegressionTest.php --testdox

# Run all tests for full regression
./vendor/bin/phpunit tests/
```

## Regression Testing Checklist

### Core Functionality
- [ ] Module activation works
- [ ] Configuration saves
- [ ] Client operations work
- [ ] Order processing works

### Admin Features
- [ ] Admin area renders
- [ ] Widgets display
- [ ] Reports generate

### API
- [ ] API calls work
- [ ] Webhooks fire
- [ ] Callbacks process

### Edge Cases
- [ ] Error handling works
- [ ] Timeouts handled
- [ ] Invalid input handled
