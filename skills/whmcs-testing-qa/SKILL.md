# WHMCS Testing & QA Skill
# Version: 1.0 | Updated: 2026-05-28

## Purpose

Guide for testing WHMCS modules with systematic test plans and quality assurance procedures.

## When to Use

- Testing new modules before deployment
- Verifying bug fixes
- Performing regression testing
- Validating payment gateways

## Test Categories

### 1. Provisioning Module Tests

```php
// Test Case: Create Account
function test_create_account() {
    $params = [
        'domain' => 'test.example.com',
        'username' => 'testuser',
        'password' => 'SecurePass123!',
        'configoption1' => 'starter',
        'serviceid' => 12345,
    ];

    $result = module_CreateAccount($params);

    assert($result === 'success', 'CreateAccount should return success');

    // Verify server details saved
    $serverId = getCustomFieldValue(12345, 'server_id');
    assert(!empty($serverId), 'Server ID should be saved');
}

// Test Case: Suspend Account
function test_suspend_account() {
    $params = [
        'serviceid' => 12345,
        'customfields' => ['server_id' => 'srv_abc123'],
    ];

    $result = module_SuspendAccount($params);
    assert($result === 'success', 'SuspendAccount should return success');
}

// Test Case: Terminate Account
function test_terminate_account() {
    $params = [
        'serviceid' => 12345,
        'customfields' => ['server_id' => 'srv_abc123'],
    ];

    $result = module_TerminateAccount($params);
    assert($result === 'success', 'TerminateAccount should return success');
}

// Test Case: Test Connection
function test_connection() {
    $params = [
        'serverhostname' => 'api.provider.com',
        'serverhttpprefix' => 'https',
        'serverusername' => 'test_api_key',
        'serverpassword' => 'test_api_secret',
    ];

    $result = module_TestConnection($params);
    assert($result['success'] === true, 'TestConnection should return success');
}
```

### 2. Payment Gateway Tests

```php
// Test Case: Generate Payment Link
function test_payment_link() {
    $params = [
        'invoiceid' => 12345,
        'amount' => 100.00,
        'currency' => 'VND',
        'returnurl' => 'https://example.com/return',
        'cancelurl' => 'https://example.com/cancel',
        'apiEndpoint' => 'https://api.provider.com',
        'merchantId' => 'MERCHANT123',
        'apiKey' => 'SECRET_KEY',
    ];

    $html = module_link($params);

    assert(strpos($html, '<form') !== false, 'Should return HTML form');
    assert(strpos($html, 'MERCHANT123') !== false, 'Should include merchant ID');
    assert(strpos($html, 'Pay Now') !== false, 'Should include submit button');
}

// Test Case: Callback Processing
function test_callback_success() {
    $_POST = [
        'merchant_id' => 'MERCHANT123',
        'order_id' => 'INV12345_' . time(),
        'amount' => 10000,
        'status' => 'completed',
        'transaction_id' => 'TXN123456',
        'signature' => 'valid_signature',
    ];

    ob_start();
    include 'modules/gateways/callback/module.php';
    $output = ob_get_clean();

    // Verify redirect
    assert(strpos($output, 'Location:') !== false, 'Should redirect after success');
}

// Test Case: Refund Processing
function test_refund() {
    $params = [
        'transid' => 'TXN123456',
        'amount' => 50.00,
        'apiKey' => 'SECRET_KEY',
        'apiEndpoint' => 'https://api.provider.com',
    ];

    $result = module_refund($params);
    assert($result['status'] === 'success', 'Refund should succeed');
}
```

### 3. Addon Module Tests

```php
// Test Case: Module Activation
function test_activate() {
    $result = module_activate();

    assert($result['status'] === 'success', 'Activation should succeed');

    // Verify tables created
    $tables = Capsule::schema()->getTables();
    assert(in_array('mod_module_logs', $tables), 'Logs table should exist');
    assert(in_array('mod_module_settings', $tables), 'Settings table should exist');
}

// Test Case: Module Deactivation
function test_deactivate() {
    $result = module_deactivate();

    assert($result['status'] === 'success', 'Deactivation should succeed');

    // Verify tables dropped
    $tables = Capsule::schema()->getTables();
    assert(!in_array('mod_module_logs', $tables), 'Logs table should be dropped');
}

// Test Case: Settings Save
function test_save_settings() {
    $_POST = [
        'api_key' => 'test_key',
        'webhook_url' => 'https://example.com/webhook',
        'debug_mode' => '1',
        'token' => generate_token(),
    ];

    $result = module_output(['action' => 'save_settings']);

    $settings = getSettings();
    assert($settings['api_key'] === 'test_key', 'API key should be saved');
}

// Test Case: CSRF Protection
function test_csrf_validation() {
    $_POST = ['action' => 'save_settings']; // No token

    $result = module_output($_POST);

    // Should redirect with error or show CSRF error
    assert(headers_sent() || strpos($result, 'CSRF') !== false, 'Should reject without token');
}
```

### 4. Registrar Module Tests

```php
// Test Case: Domain Registration
function test_register_domain() {
    $params = [
        'domain' => 'test-domain.com',
        'regperiod' => 1,
        'contactdetails' => [
            'Registrant' => [
                'fullname' => 'John Doe',
                'email' => 'john@example.com',
                'companyname' => 'Example Inc',
                'country' => 'US',
            ],
        ],
    ];

    $result = module_RegisterDomain($params);
    assert($result['success'] === true, 'Registration should succeed');
}

// Test Case: Domain Transfer
function test_transfer_domain() {
    $params = [
        'domain' => 'test-domain.com',
        'transfersecret' => 'AUTHCODE123',
    ];

    $result = module_TransferDomain($params);
    assert($result['success'] === true, 'Transfer should succeed');
}

// Test Case: Sync Domain
function test_sync_domain() {
    $params = ['domain' => 'test-domain.com'];

    $result = module_Sync($params);

    assert(in_array($result['status'], ['Active', 'Expired', 'Pending Transfer']),
           'Status should be valid');
    assert(!empty($result['expiry']), 'Expiry date should be returned');
}
```

### 5. Notification Module Tests

```php
// Test Case: Connection Test
function test_notification_connection() {
    $module = new NotificationProvider();
    $module->setSetting('api_key', 'test_key');
    $module->setSetting('api_secret', 'test_secret');

    try {
        $module->testConnection();
        assert(true, 'Connection test should pass');
    } catch (\Exception $e) {
        assert(false, 'Connection test failed: ' . $e->getMessage());
    }
}

// Test Case: Send Notification
function test_send_notification() {
    $module = new NotificationProvider();
    $module->setSetting('api_key', 'test_key');

    $notification = new MockNotification([
        'title' => 'Test Title',
        'message' => 'Test message body',
        'url' => 'https://example.com/link',
    ]);

    $settings = [
        'recipient_type' => 'static',
        'static_recipient' => '+84123456789',
        'template' => '{title}: {message}',
    ];

    try {
        $module->send($notification, $settings);
        assert(true, 'Notification should be sent');
    } catch (\Exception $e) {
        assert(false, 'Send failed: ' . $e->getMessage());
    }
}
```

## Test Checklist

### Pre-Deployment
- [ ] All function signatures correct
- [ ] Return values match WHMCS expectations
- [ ] Error messages are user-friendly
- [ ] Logging is adequate
- [ ] CSRF protection in place

### Module-Specific
- [ ] Provisioning: Create/Suspend/Unsuspend/Terminate all work
- [ ] Gateway: Payment link generated correctly
- [ ] Gateway: Callback processes all statuses
- [ ] Gateway: Refund works
- [ ] Registrar: Register/Transfer/Renew all work
- [ ] Registrar: Sync returns correct status
- [ ] Addon: Activation creates tables
- [ ] Addon: Settings save correctly
- [ ] Addon: CSRF protection works
- [ ] Notification: Test connection passes

### Security
- [ ] No hardcoded credentials
- [ ] Input validation in place
- [ ] Output escaping in templates
- [ ] Callback signature validation
- [ ] Rate limiting if applicable

---

**Related Skills:**
- whmcs-validator
- whmcs-security-hardening
- whmcs-deployment