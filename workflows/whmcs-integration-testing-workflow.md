# WHMCS Integration Testing Workflow

## Purpose

Systematic approach to testing WHMCS integrations with third-party services, including payment gateways, registrars, provisioning modules, and external APIs.

## Prerequisites

- WHMCS development/testing environment
- Integration sandbox accounts
- Testing frameworks (PHPUnit, Codeception)
- API testing tools (Postman, Insomnia)

## Workflow Steps

### Step 1: Set Up Testing Environment

Create isolated testing infrastructure:

```bash
#!/bin/bash
# setup_integration_test_env.sh

set -euo pipefail

# Create test database
mysql -u root -p -e "CREATE DATABASE IF NOT EXISTS whmcs_integration_test"
mysql -u root -p -e "GRANT ALL ON whmcs_integration_test.* TO 'whmcs_test'@'localhost'"

# Clone WHMCS for testing
TEST_WHMCS="/var/www/whmcs-integration-test"
if [ ! -d "$TEST_WHMCS" ]; then
    cp -r /var/www/whmcs "$TEST_WHMCS"
fi

# Create test configuration
cat > "$TEST_WHMCS/configuration.php" << 'EOF'
<?php
$db_host = "localhost";
$db_username = "whmcs_test";
$db_password = "test_password";
$db_name = "whmcs_integration_test";
$cc_encryption_hash = 'test_encryption_hash';
$systems_url = 'http://localhost:8080';
$domain = 'localhost';
$display_errors = true;
$debug = true;
EOF

# Install test data
php << 'PHP'
<?php
require_once '/var/www/whmcs-integration-test/init.php';

// Create test client
$clientId = Capsule::table('tblclients')->insertGetId([
    'firstname' => 'Test',
    'lastname' => 'Integration',
    'email' => 'test@example.com',
    'company' => 'Test Company',
    'password' => md5('TestPassword123!'),
    'datecreated' => date('Y-m-d H:i:s'),
    'status' => 'Active',
]);

// Create test product
$productId = Capsule::table('tblproducts')->insertGetId([
    'type' => 'hosting',
    'name' => 'Test Hosting',
    'description' => 'Test product for integration testing',
    '的自由' => 'on',
    'monthly' => 9.99,
    'quarterly' => 26.97,
    'semiannually' => 51.96,
    'annually' => 95.88,
    'biennially' => 179.76,
    'configoption1' => 1,
    'configoption2' => 10,
    'configoption3' => 'linux',
    'configoption4' => 'cpanel',
]);

// Create test server
$serverId = Capsule::table('tblservers')->insertGetId([
    'name' => 'Test Server',
    'hostname' => 'test-server.local',
    'ipaddress' => '192.168.1.100',
    'type' => 'cpanel',
    'active' => 1,
    'maxaccounts' => 100,
]);

echo "Test environment setup complete\n";
echo "Client ID: $clientId\n";
echo "Product ID: $productId\n";
echo "Server ID: $serverId\n";
PHP
```

### Step 2: Create Integration Test Suite

Build comprehensive integration tests:

```php
<?php
// resources/testing/Integration/PaymentGatewayTest.php

namespace WHMCS\Testing\Integration;

use PHPUnit\Framework\TestCase;
use WHMCS\Module\Payment\PaymentProcessor;

class PaymentGatewayIntegrationTest extends TestCase
{
    private $gateway;
    private $testInvoice;
    
    protected function setUp(): void
    {
        parent::setUp();
        
        // Initialize payment gateway
        $this->gateway = new PaymentProcessor('stripe');
        
        // Create test invoice
        $this->testInvoice = $this->createTestInvoice();
    }
    
    public function testPaymentGatewayConnection()
    {
        // Test API connectivity
        $result = $this->gateway->testConnection([
            'apiKey' => $_ENV['STRIPE_TEST_API_KEY'],
            'environment' => 'test',
        ]);
        
        $this->assertTrue($result['success'], 'Gateway connection should succeed');
    }
    
    public function testCreatePaymentIntent()
    {
        $result = $this->gateway->createPaymentIntent([
            'amount' => $this->testInvoice->total * 100, // Stripe uses cents
            'currency' => 'usd',
            'invoice_id' => $this->testInvoice->id,
            'customer_email' => 'test@example.com',
        ]);
        
        $this->assertTrue($result['success']);
        $this->assertArrayHasKey('client_secret', $result);
        $this->assertArrayHasKey('payment_intent_id', $result);
        
        // Store for webhook testing
        $this->gatewayPaymentIntent = $result['payment_intent_id'];
    }
    
    public function testWebhookProcessing()
    {
        // Simulate webhook payload
        $webhookPayload = [
            'type' => 'payment_intent.succeeded',
            'data' => [
                'object' => [
                    'id' => $this->gatewayPaymentIntent,
                    'amount' => $this->testInvoice->total * 100,
                    'currency' => 'usd',
                    'metadata' => [
                        'invoice_id' => $this->testInvoice->id,
                    ],
                ],
            ],
        ];
        
        $webhookSignature = $this->gateway->generateWebhookSignature($webhookPayload);
        
        // Process webhook
        $result = $this->gateway->processWebhook(
            json_encode($webhookPayload),
            $webhookSignature
        );
        
        $this->assertTrue($result['success']);
        
        // Verify invoice was updated
        $invoice = Capsule::table('tblinvoices')
            ->where('id', $this->testInvoice->id)
            ->first();
        
        $this->assertEquals('Paid', $invoice->status);
    }
    
    public function testRefundProcessing()
    {
        // First process payment
        $this->gateway->processPayment([
            'invoice_id' => $this->testInvoice->id,
            'amount' => $this->testInvoice->total,
            'transaction_id' => 'test_txn_' . time(),
        ]);
        
        // Then refund
        $result = $this->gateway->refund([
            'invoice_id' => $this->testInvoice->id,
            'amount' => $this->testInvoice->total,
            'reason' => 'Test refund',
        ]);
        
        $this->assertTrue($result['success']);
    }
    
    public function testFailedPaymentHandling()
    {
        $result = $this->gateway->createPaymentIntent([
            'amount' => $this->testInvoice->total * 100,
            'currency' => 'usd',
            'invoice_id' => $this->testInvoice->id,
        ]);
        
        // Simulate failed payment
        $webhookPayload = [
            'type' => 'payment_intent.payment_failed',
            'data' => [
                'object' => [
                    'id' => $result['payment_intent_id'],
                    'last_payment_error' => [
                        'message' => 'Card declined',
                    ],
                ],
            ],
        ];
        
        $result = $this->gateway->processWebhook(
            json_encode($webhookPayload),
            $this->gateway->generateWebhookSignature($webhookPayload)
        );
        
        $this->assertTrue($result['processed']);
    }
    
    private function createTestInvoice()
    {
        $clientId = Capsule::table('tblclients')
            ->where('email', 'test@example.com')
            ->value('id');
        
        $invoiceId = Capsule::table('tblinvoices')->insertGetId([
            'userid' => $clientId,
            'invoicenum' => 'INV-TEST-' . time(),
            'date' => date('Y-m-d'),
            'duedate' => date('Y-m-d', strtotime('+7 days')),
            'total' => 99.99,
            'status' => 'Unpaid',
        ]);
        
        Capsule::table('tblinvoiceitems')->insert([
            'invoiceid' => $invoiceId,
            'userid' => $clientId,
            'description' => 'Test Service',
            'amount' => 99.99,
            'taxed' => 1,
        ]);
        
        return Capsule::table('tblinvoices')->where('id', $invoiceId)->first();
    }
}
```

### Step 3: Create Registrar Integration Tests

Test domain registrar integrations:

```php
<?php
// resources/testing/Integration/RegistrarIntegrationTest.php

namespace WHMCS\Testing\Integration;

use PHPUnit\Framework\TestCase;

class RegistrarIntegrationTest extends TestCase
{
    private $registrar;
    private $testDomain;
    
    protected function setUp(): void
    {
        parent::setUp();
        
        $this->registrar = new \WHMCS\Module\Registrar('enom');
        $this->testDomain = 'test-integration-' . time() . '.com';
    }
    
    public function testRegistrarConnection()
    {
        $result = $this->registrar->testConnection([
            'RegistrarAPIUsername' => $_ENV['ENOM_API_USER'],
            'RegistrarAPISecret' => $_ENV['ENOM_API_KEY'],
            'TestMode' => true,
        ]);
        
        $this->assertTrue($result['success'], 'Registrar connection should succeed');
    }
    
    public function testDomainAvailabilityCheck()
    {
        $domains = ['test-domain-available.com', 'google.com'];
        
        $result = $this->registrar->checkAvailability([
            'tlds' => ['com', 'net'],
            'sld' => 'test-domain-available',
        ]);
        
        $this->assertTrue($result['success']);
        $this->assertArrayHasKey('available', $result);
    }
    
    public function testDomainRegistration()
    {
        $result = $this->registrar->registerDomain([
            'sld' => substr($this->testDomain, 0, strpos($this->testDomain, '.')),
            'tld' => 'com',
            'registrationperiod' => 1,
            'registrant' => [
                'firstname' => 'Test',
                'lastname' => 'User',
                'email' => 'test@example.com',
                'address1' => '123 Test St',
                'city' => 'Test City',
                'state' => 'TS',
                'country' => 'US',
                'postcode' => '12345',
                'phonenumber' => '+1.5555555555',
            ],
            'dnsmanagement' => true,
            'emailforwarding' => true,
            'idprotection' => false,
        ]);
        
        $this->assertTrue($result['success']);
        
        // Clean up - delete test domain
        $this->deleteTestDomain();
    }
    
    public function testDomainTransfer()
    {
        $result = $this->registrar->transferDomain([
            'sld' => 'example-domain',
            'tld' => 'com',
            'transfersecret' => 'AUTH_CODE_123',
        ]);
        
        $this->assertTrue($result['success']);
        $this->assertArrayHasKey('transferid', $result);
    }
    
    public function testNameserverUpdate()
    {
        // Register domain first
        $this->registrar->registerDomain([
            'sld' => substr($this->testDomain, 0, strpos($this->testDomain, '.')),
            'tld' => 'com',
            'registrationperiod' => 1,
            'registrant' => $this->getRegistrantData(),
        ]);
        
        // Then update nameservers
        $result = $this->registrar->saveNameservers([
            'sld' => substr($this->testDomain, 0, strpos($this->testDomain, '.')),
            'tld' => 'com',
            'ns1' => 'ns1.testdns.com',
            'ns2' => 'ns2.testdns.com',
            'ns3' => 'ns3.testdns.com',
        ]);
        
        $this->assertTrue($result['success']);
        
        // Clean up
        $this->deleteTestDomain();
    }
    
    public function testDomainLocking()
    {
        $this->registrar->registerDomain([
            'sld' => substr($this->testDomain, 0, strpos($this->testDomain, '.')),
            'tld' => 'com',
            'registrationperiod' => 1,
            'registrant' => $this->getRegistrantData(),
        ]);
        
        // Lock domain
        $result = $this->registrar->lock([
            'sld' => substr($this->testDomain, 0, strpos($this->testDomain, '.')),
            'tld' => 'com',
        ]);
        
        $this->assertTrue($result['success']);
        
        // Verify lock status
        $status = $this->registrar->getRegistrarLock([
            'sld' => substr($this->testDomain, 0, strpos($this->testDomain, '.')),
            'tld' => 'com',
        ]);
        
        $this->assertEquals('locked', $status['lockstatus']);
        
        // Unlock
        $this->registrar->unlock([
            'sld' => substr($this->testDomain, 0, strpos($this->testDomain, '.')),
            'tld' => 'com',
        ]);
        
        // Clean up
        $this->deleteTestDomain();
    }
    
    public function testDNSSECManagement()
    {
        // Add DNSSEC
        $result = $this->registrar->saveDnssec([
            'sld' => substr($this->testDomain, 0, strpos($this->testDomain, '.')),
            'tld' => 'com',
            'dnssec' => [
                ['flags' => 257, 'protocol' => 3, 'alg' => 8, 'pubKey' => 'PUBLIC_KEY_DATA'],
            ],
        ]);
        
        $this->assertTrue($result['success']);
        
        // Get DNSSEC
        $dnssec = $this->registrar->getDnssec([
            'sld' => substr($this->testDomain, 0, strpos($this->testDomain, '.')),
            'tld' => 'com',
        ]);
        
        $this->assertIsArray($dnssec['dnssec']);
        $this->assertCount(1, $dnssec['dnssec']);
    }
    
    private function getRegistrantData()
    {
        return [
            'firstname' => 'Test',
            'lastname' => 'User',
            'email' => 'test@example.com',
            'address1' => '123 Test St',
            'city' => 'Test City',
            'state' => 'TS',
            'country' => 'US',
            'postcode' => '12345',
            'phonenumber' => '+1.5555555555',
        ];
    }
    
    private function deleteTestDomain()
    {
        try {
            $this->registrar->requestDelete([
                'sld' => substr($this->testDomain, 0, strpos($this->testDomain, '.')),
                'tld' => 'com',
            ]);
        } catch (\Exception $e) {
            // Ignore cleanup errors
        }
    }
}
```

### Step 4: Create Provisioning Module Tests

Test server provisioning integration:

```php
<?php
// resources/testing/Integration/ProvisioningIntegrationTest.php

namespace WHMCS\Testing\Integration;

use PHPUnit\Framework\TestCase;

class ProvisioningIntegrationTest extends TestCase
{
    private $module;
    private $testService;
    
    protected function setUp(): void
    {
        parent::setUp();
        
        $this->module = new \WHMCS\Module\Server('cpanel');
        $this->testService = $this->createTestService();
    }
    
    public function testProvisioningConnection()
    {
        $result = $this->module->testConnection([
            'hostname' => 'test.cpanel.server',
            'ipaddress' => '192.168.1.100',
            'username' => $_ENV['CPANEL_TEST_USER'],
            'password' => $_ENV['CPANEL_TEST_PASS'],
            'accesshash' => '',
            'secure' => true,
        ]);
        
        $this->assertTrue($result['success']);
    }
    
    public function testAccountCreation()
    {
        $result = $this->module->createAccount([
            'serviceid' => $this->testService->id,
            'model' => $this->testService,
        ]);
        
        $this->assertEquals('success', $result);
        
        // Verify in WHMCS
        $service = Capsule::table('tblhosting')
            ->where('id', $this->testService->id)
            ->first();
        
        $this->assertEquals('Active', $service->domainstatus);
        $this->assertNotEmpty($service->username);
        
        // Clean up
        $this->module->terminateAccount([
            'serviceid' => $this->testService->id,
            'model' => $service,
        ]);
    }
    
    public function testAccountTermination()
    {
        // Create account first
        $this->module->createAccount([
            'serviceid' => $this->testService->id,
            'model' => $this->testService,
        ]);
        
        // Then terminate
        $result = $this->module->terminateAccount([
            'serviceid' => $this->testService->id,
            'model' => $this->testService,
        ]);
        
        $this->assertEquals('success', $result);
        
        // Verify status
        $service = Capsule::table('tblhosting')
            ->where('id', $this->testService->id)
            ->first();
        
        $this->assertEquals('Terminated', $service->domainstatus);
    }
    
    public function testAccountSuspension()
    {
        $this->module->createAccount([
            'serviceid' => $this->testService->id,
            'model' => $this->testService,
        ]);
        
        $result = $this->module->suspendAccount([
            'serviceid' => $this->testService->id,
            'model' => $this->testService,
            'suspendreason' => 'Test suspension',
        ]);
        
        $this->assertEquals('success', $result);
        
        // Unsuspend
        $result = $this->module->unsuspendAccount([
            'serviceid' => $this->testService->id,
            'model' => $this->testService,
        ]);
        
        $this->assertEquals('success', $result);
    }
    
    public function testChangePassword()
    {
        $this->module->createAccount([
            'serviceid' => $this->testService->id,
            'model' => $this->testService,
        ]);
        
        $newPassword = 'NewTestPass123!';
        $result = $this->module->changePassword([
            'serviceid' => $this->testService->id,
            'model' => $this->testService,
            'password' => $newPassword,
        ]);
        
        $this->assertEquals('success', $result);
    }
    
    public function testChangePackage()
    {
        $this->module->createAccount([
            'serviceid' => $this->testService->id,
            'model' => $this->testService,
        ]);
        
        $newPackage = Capsule::table('tblproducts')
            ->where('type', 'hosting')
            ->where('id', '!=', $this->testService->packageid)
            ->first();
        
        if ($newPackage) {
            $result = $this->module->changePackage([
                'serviceid' => $this->testService->id,
                'model' => $this->testService,
            ]);
            
            $this->assertEquals('success', $result);
        }
    }
    
    private function createTestService()
    {
        $clientId = Capsule::table('tblclients')
            ->where('email', 'test@example.com')
            ->value('id');
        
        $productId = Capsule::table('tblproducts')
            ->where('type', 'hosting')
            ->value('id');
        
        $serverId = Capsule::table('tblservers')
            ->where('type', 'cpanel')
            ->value('id');
        
        $serviceId = Capsule::table('tblhosting')->insertGetId([
            'server' => $serverId,
            'userid' => $clientId,
            'packageid' => $productId,
            'domain' => 'test-integration-' . time() . '.example.com',
            'regdate' => date('Y-m-d'),
            'domainstatus' => 'Pending',
            'username' => '',
            'password' => '',
        ]);
        
        return Capsule::table('tblhosting')->where('id', $serviceId)->first();
    }
}
```

### Step 5: Create API Integration Tests

Test WHMCS API integrations:

```php
<?php
// resources/testing/Integration/APIIntegrationTest.php

namespace WHMCS\Testing\Integration;

use PHPUnit\Framework\TestCase;

class WHMCSAPIIntegrationTest extends TestCase
{
    private $apiUrl;
    private $apiIdentifier;
    private $apiSecret;
    
    protected function setUp(): void
    {
        parent::setUp();
        
        $this->apiUrl = 'http://localhost:8080/includes/api.php';
        $this->apiIdentifier = $_ENV['WHMCS_API_ID'] ?? 'test_api_key';
        $this->apiSecret = $_ENV['WHMCS_API_SECRET'] ?? 'test_api_secret';
    }
    
    public function testValidateLogin()
    {
        $result = $this->apiCall('ValidateLogin', [
            'email' => 'test@example.com',
            'password2' => 'TestPassword123!',
        ]);
        
        $this->assertTrue($result['result'] === 'success');
    }
    
    public function testGetClients()
    {
        $result = $this->apiCall('GetClients', [
            'limitstart' => 0,
            'limitnum' => 10,
        ]);
        
        $this->assertTrue($result['result'] === 'success');
        $this->assertArrayHasKey('clients', $result);
    }
    
    public function testGetClientDetails()
    {
        $clientId = Capsule::table('tblclients')
            ->where('email', 'test@example.com')
            ->value('id');
        
        $result = $this->apiCall('GetClientDetails', [
            'clientid' => $clientId,
        ]);
        
        $this->assertTrue($result['result'] === 'success');
        $this->assertEquals($clientId, $result['userid']);
    }
    
    public function testCreateInvoice()
    {
        $clientId = Capsule::table('tblclients')
            ->where('email', 'test@example.com')
            ->value('id');
        
        $result = $this->apiCall('CreateInvoice', [
            'userid' => $clientId,
            'sendinvoice' => false,
            'items' => [
                [
                    'description' => 'Test Service',
                    'amount' => 99.99,
                ],
            ],
        ]);
        
        $this->assertTrue($result['result'] === 'success');
        $this->assertArrayHasKey('invoiceid', $result);
        
        // Clean up
        Capsule::table('tblinvoices')
            ->where('id', $result['invoiceid'])
            ->delete();
    }
    
    public function testAddOrder()
    {
        $clientId = Capsule::table('tblclients')
            ->where('email', 'test@example.com')
            ->value('id');
        
        $productId = Capsule::table('tblproducts')
            ->where('type', 'hosting')
            ->value('id');
        
        $result = $this->apiCall('AddOrder', [
            'clientid' => $clientId,
            'pid' => [$productId],
            'billingcycle' => 'Monthly',
            'paymentmethod' => 'paypal',
            'domain' => 'test-api-' . time() . '.com',
        ]);
        
        $this->assertTrue($result['result'] === 'success');
        $this->assertArrayHasKey('orderid', $result);
        
        // Clean up
        Capsule::table('tblorders')
            ->where('id', $result['orderid'])
            ->delete();
    }
    
    public function testAcceptOrder()
    {
        $clientId = Capsule::table('tblclients')
            ->where('email', 'test@example.com')
            ->value('id');
        
        $productId = Capsule::table('tblproducts')
            ->where('type', 'hosting')
            ->value('id');
        
        // Create order
        $createResult = $this->apiCall('AddOrder', [
            'clientid' => $clientId,
            'pid' => [$productId],
            'billingcycle' => 'Monthly',
            'paymentmethod' => 'paypal',
            'domain' => 'test-api-' . time() . '.com',
            'noprovision' => true,
        ]);
        
        // Accept order
        $acceptResult = $this->apiCall('AcceptOrder', [
            'orderid' => $createResult['orderid'],
        ]);
        
        $this->assertTrue($acceptResult['result'] === 'success');
        
        // Clean up
        Capsule::table('tblorders')
            ->where('id', $createResult['orderid'])
            ->delete();
    }
    
    private function apiCall($action, array $postData)
    {
        $postData['action'] = $action;
        $postData['username'] = $this->apiIdentifier;
        $postData['password'] = md5($this->apiSecret);
        $postData['responsetype'] = 'json';
        
        $ch = curl_init($this->apiUrl);
        curl_setopt_array($ch, [
            CURLOPT_POST => true,
            CURLOPT_POSTFIELDS => http_build_query($postData),
            CURLOPT_RETURNTRANSFER => true,
            CURLOPT_TIMEOUT => 30,
        ]);
        
        $response = curl_exec($ch);
        curl_close($ch);
        
        return json_decode($response, true);
    }
}
```

### Step 6: Create Webhook Integration Tests

Test webhook integrations:

```php
<?php
// resources/testing/Integration/WebhookIntegrationTest.php

namespace WHMCS\Testing\Integration;

use PHPUnit\Framework\TestCase;

class WebhookIntegrationTest extends TestCase
{
    private $webhookUrl;
    private $webhookSecret;
    
    protected function setUp(): void
    {
        parent::setUp();
        
        $this->webhookUrl = 'http://localhost:8080/api/webhooks/test';
        $this->webhookSecret = $_ENV['WEBHOOK_SECRET'] ?? 'test_secret';
    }
    
    public function testInvoicePaymentWebhook()
    {
        $payload = [
            'event' => 'invoice.paid',
            'timestamp' => date('c'),
            'data' => [
                'invoice_id' => $this->getTestInvoiceId(),
                'amount' => 99.99,
                'client_id' => 1,
                'payment_method' => 'stripe',
            ],
        ];
        
        $result = $this->sendWebhook($payload);
        
        $this->assertTrue($result['received']);
        $this->assertTrue($result['processed']);
    }
    
    public function testServiceCreationWebhook()
    {
        $payload = [
            'event' => 'service.created',
            'timestamp' => date('c'),
            'data' => [
                'service_id' => $this->getTestServiceId(),
                'domain' => 'test-service.example.com',
                'client_id' => 1,
                'product_name' => 'Test Hosting',
            ],
        ];
        
        $result = $this->sendWebhook($payload);
        
        $this->assertTrue($result['received']);
        $this->assertTrue($result['processed']);
    }
    
    public function testTicketCreationWebhook()
    {
        $payload = [
            'event' => 'ticket.created',
            'timestamp' => date('c'),
            'data' => [
                'ticket_id' => rand(1000, 9999),
                'subject' => 'Test Ticket',
                'client_id' => 1,
                'priority' => 'Medium',
            ],
        ];
        
        $result = $this->sendWebhook($payload);
        
        $this->assertTrue($result['received']);
    }
    
    public function testInvalidSignatureRejected()
    {
        $payload = [
            'event' => 'invoice.paid',
            'data' => ['invoice_id' => 123],
        ];
        
        $ch = curl_init($this->webhookUrl);
        curl_setopt_array($ch, [
            CURLOPT_POST => true,
            CURLOPT_POSTFIELDS => json_encode($payload),
            CURLOPT_HTTPHEADER => [
                'Content-Type: application/json',
                'X-Webhook-Signature: invalid_signature',
            ],
            CURLOPT_RETURNTRANSFER => true,
        ]);
        
        $response = curl_exec($ch);
        $httpCode = curl_getinfo($ch, CURLINFO_HTTP_CODE);
        curl_close($ch);
        
        $this->assertEquals(401, $httpCode);
    }
    
    private function sendWebhook(array $payload)
    {
        $signature = hash_hmac('sha256', json_encode($payload), $this->webhookSecret);
        
        $ch = curl_init($this->webhookUrl);
        curl_setopt_array($ch, [
            CURLOPT_POST => true,
            CURLOPT_POSTFIELDS => json_encode($payload),
            CURLOPT_HTTPHEADER => [
                'Content-Type: application/json',
                'X-Webhook-Signature: ' . $signature,
            ],
            CURLOPT_RETURNTRANSFER => true,
            CURLOPT_TIMEOUT => 30,
        ]);
        
        $response = curl_exec($ch);
        curl_close($ch);
        
        return json_decode($response, true);
    }
}
```

### Step 7: Run Integration Tests

Execute integration test suite:

```bash
#!/bin/bash
# run_integration_tests.sh

set -euo pipefail

echo "=== WHMCS Integration Test Suite ==="
echo "Started at: $(date)"
echo ""

# Environment setup
export WHMCS_API_ID="${WHMCS_API_ID:-test_api}"
export WHMCS_API_SECRET="${WHMCS_API_SECRET:-test_secret}"
export STRIPE_TEST_API_KEY="${STRIPE_TEST_API_KEY:-sk_test_xxx}"
export ENOM_API_USER="${ENOM_API_USER:-test}"
export ENOM_API_KEY="${ENOM_API_KEY:-test}"
export WEBHOOK_SECRET="${WEBHOOK_SECRET:-test_secret}"
export CPANEL_TEST_USER="${CPANEL_TEST_USER:-test}"
export CPANEL_TEST_PASS="${CPANEL_TEST_PASS:-test}"

# Start test server if needed
echo "Starting test server..."
php -S localhost:8080 -t /var/www/whmcs-integration-test > /dev/null 2>&1 &
TEST_SERVER_PID=$!
sleep 3

# Run PHPUnit
cd /var/www/whmcs
./vendor/bin/phpunit \
    --configuration phpunit-integration.xml \
    --testsuite Integration \
    --log-junit test-results/integration-junit.xml \
    --coverage-html test-results/coverage \
    || true

# Stop test server
kill $TEST_SERVER_PID 2>/dev/null || true

echo ""
echo "=== Test Suite Complete ==="
echo "Results saved to test-results/"
```

```xml
<!-- phpunit-integration.xml -->
<?xml version="1.0"?>
<phpunit xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:noNamespaceSchemaLocation="vendor/phpunit/phpunit/phpunit.xsd"
         bootstrap="resources/testing/bootstrap_integration.php"
         colors="true"
         convertErrorsToExceptions="true"
         convertWarningsToExceptions="true"
         stopOnFailure="false">
    <testsuites>
        <testsuite name="Integration">
            <directory>resources/testing/Integration</directory>
        </testsuite>
    </testsuites>
    <coverage>
        <include>
            <directory suffix=".php">modules/</directory>
            <directory suffix=".php">includes/</directory>
        </include>
        <exclude>
            <directory>vendor</directory>
        </exclude>
    </coverage>
</phpunit>
```

## Verification Checklist

- [ ] Test environment configured
- [ ] Payment gateway tests passing
- [ ] Registrar tests passing
- [ ] Provisioning tests passing
- [ ] API tests passing
- [ ] Webhook tests passing
- [ ] Test data cleanup working
- [ ] Coverage reports generated
- [ ] CI/CD pipeline configured
- [ ] Test documentation complete

## Related Skills and Documentation

- [WHMCS Module Testing](whmcs-module-testing.md)
- [WHMCS API Development](whmcs-api-development-workflow.md)
- [WHMCS Webhook Automation](whmcs-webhook-automation-workflow.md)
- PHPUnit Documentation: https://phpunit.readthedocs.io/
- WHMCS API Documentation: https://developers.whmcs.com/api-index/

## Notes

- Always use test/sandbox environments for integration testing
- Clean up test data after each test run
- Use unique identifiers to avoid test collisions
- Mock external services when not in sandbox mode
- Run integration tests in CI/CD pipeline
- Monitor test coverage for critical paths
- Update tests when external APIs change
