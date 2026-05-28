# WHMCS Integration Testing Workflow

## Purpose

Comprehensive guide to testing integrations between WHMCS and external services, including payment gateways, provisioning servers, domain registrars, and third-party APIs.

## Prerequisites

- WHMCS development environment
- Testing accounts for external services
- API credentials for sandboxes
- Testing framework (PHPUnit)
- Mocking capabilities

## Workflow Steps

### Step 1: Test Environment Setup

Configure integration testing environment:

```php
// tests/Integration/bootstrap.php

require_once __DIR__ . '/../../whmcs/includes/init.php';

// Configure test database
define('WHLMCS_TEST_MODE', true);

// Set up test API credentials
class TestCredentials
{
    public static function getPaymentGatewayCredentials(): array
    {
        return [
            'merchant_id' => 'test_merchant_' . getenv('TEST_ENV_SUFFIX'),
            'api_key' => 'test_api_key',
            'environment' => 'sandbox',
        ];
    }
    
    public static function getProvisioningCredentials(): array
    {
        return [
            'api_url' => 'https://sandbox.provisioning-api.example.com',
            'api_key' => 'test_provisioning_key',
            'api_secret' => 'test_provisioning_secret',
        ];
    }
    
    public static function getDomainRegistrarCredentials(): array
    {
        return [
            'api_username' => 'test_registrar_user',
            'api_password' => 'test_registrar_pass',
            'sandbox' => true,
        ];
    }
}
```

### Step 2: Payment Gateway Integration Tests

Test payment gateway integration:

```php
// tests/Integration/PaymentGatewayTest.php

class PaymentGatewayIntegrationTest extends TestCase
{
    private $gateway;
    private $testInvoice;
    
    protected function setUp(): void
    {
        parent::setUp();
        
        $credentials = TestCredentials::getPaymentGatewayCredentials();
        $this->gateway = new StripeGateway($credentials);
        
        // Create test invoice
        $this->testInvoice = $this->createTestInvoice();
    }
    
    /**
     * Test payment creation
     */
    public function testCreatePayment(): void
    {
        $paymentData = [
            'invoice_id' => $this->testInvoice->id,
            'amount' => $this->testInvoice->total,
            'currency' => 'USD',
            'payment_method' => 'card',
            'card_token' => 'tok_test_visa',
            'customer_email' => 'test@example.com',
        ];
        
        $result = $this->gateway->createPayment($paymentData);
        
        $this->assertTrue($result['success']);
        $this->assertArrayHasKey('transaction_id', $result);
        $this->assertNotEmpty($result['transaction_id']);
    }
    
    /**
     * Test payment refund
     */
    public function testRefundPayment(): void
    {
        // First create a payment
        $payment = $this->gateway->createPayment([
            'invoice_id' => $this->testInvoice->id,
            'amount' => $this->testInvoice->total,
            'currency' => 'USD',
            'card_token' => 'tok_test_visa',
        ]);
        
        // Then refund it
        $refundResult = $this->gateway->refundPayment([
            'transaction_id' => $payment['transaction_id'],
            'amount' => $this->testInvoice->total,
            'reason' => 'Test refund',
        ]);
        
        $this->assertTrue($refundResult['success']);
        $this->assertEquals('refunded', $refundResult['status']);
    }
    
    /**
     * Test webhook handling
     */
    public function testWebhookHandling(): void
    {
        $webhookPayload = $this->generateTestWebhookPayload('payment.succeeded');
        
        // Verify webhook signature
        $isValid = $this->gateway->verifyWebhookSignature(
            $webhookPayload['payload'],
            $webhookPayload['headers']
        );
        
        $this->assertTrue($isValid);
        
        // Process webhook
        $result = $this->gateway->handleWebhook($webhookPayload['payload']);
        
        $this->assertTrue($result['processed']);
        $this->assertEquals('payment.succeeded', $result['event_type']);
    }
    
    /**
     * Test failed payment handling
     */
    public function testFailedPaymentHandling(): void
    {
        $result = $this->gateway->createPayment([
            'invoice_id' => $this->testInvoice->id,
            'amount' => $this->testInvoice->total,
            'card_token' => 'tok_test_declined',
        ]);
        
        $this->assertFalse($result['success']);
        $this->assertArrayHasKey('error', $result);
    }
    
    private function createTestInvoice(): object
    {
        return Capsule::table('tblinvoices')->insertGetId([
            'userid' => $this->testClient->id,
            'invoicenum' => 'TEST-' . time(),
            'total' => 99.99,
            'status' => 'Unpaid',
            'date' => date('Y-m-d'),
            'duedate' => date('Y-m-d', strtotime('+7 days')),
        ]);
    }
}
```

### Step 3: Provisioning Module Integration Tests

Test server provisioning integration:

```php
// tests/Integration/ProvisioningModuleTest.php

class ProvisioningModuleIntegrationTest extends TestCase
{
    private $module;
    private $testServer;
    private $testService;
    
    protected function setUp(): void
    {
        parent::setUp();
        
        $credentials = TestCredentials::getProvisioningCredentials();
        $this->module = new CloudProviderModule($credentials);
        
        $this->testServer = $this->createTestServer();
        $this->testService = $this->createTestService();
    }
    
    /**
     * Test service creation
     */
    public function testCreateService(): void
    {
        $params = $this->buildModuleParams($this->testService);
        
        $result = $this->module->CreateAccount($params);
        
        $this->assertEquals('success', $result);
        
        // Verify service was created
        $serviceDetails = $this->module->GetServiceDetails($params);
        $this->assertEquals('active', $serviceDetails['status']);
    }
    
    /**
     * Test service suspension
     */
    public function testSuspendService(): void
    {
        $params = $this->buildModuleParams($this->testService);
        
        $result = $this->module->SuspendAccount($params);
        
        $this->assertEquals('success', $result);
        
        // Verify service is suspended
        $serviceDetails = $this->module->GetServiceDetails($params);
        $this->assertEquals('suspended', $serviceDetails['status']);
    }
    
    /**
     * Test service termination
     */
    public function testTerminateService(): void
    {
        $params = $this->buildModuleParams($this->testService);
        
        $result = $this->module->TerminateAccount($params);
        
        $this->assertEquals('success', $result);
        
        // Verify service no longer exists
        $serviceDetails = $this->module->GetServiceDetails($params);
        $this->assertFalse($serviceDetails['exists']);
    }
    
    /**
     * Test remote management
     */
    public function testRemoteManagement(): void
    {
        $params = $this->buildModuleParams($this->testService);
        
        // Test reboot
        $rebootResult = $this->module->RebootServer($params);
        $this->assertTrue($rebootResult['success']);
        
        // Test console access
        $consoleResult = $this->module->GetConsoleURL($params);
        $this->assertArrayHasKey('console_url', $consoleResult);
        
        // Test metrics
        $metricsResult = $this->module->GetServiceMetrics($params);
        $this->assertArrayHasKey('cpu_usage', $metricsResult);
        $this->assertArrayHasKey('memory_usage', $metricsResult);
    }
    
    /**
     * Test automated sync
     */
    public function testAutomatedSync(): void
    {
        $result = $this->module->SyncAccount($this->buildModuleParams($this->testService));
        
        $this->assertEquals('success', $result['description']);
        $this->assertArrayHasKey('status', $result);
        $this->assertArrayHasKey('bandwidthused', $result);
    }
    
    private function buildModuleParams($service): array
    {
        return [
            'serviceid' => $service->id,
            'userid' => $service->userid,
            'domain' => $service->domain,
            'username' => $service->username,
            'password' => decrypt($service->password),
            'server' => [
                'id' => $this->testServer->id,
                'hostname' => $this->testServer->hostname,
                'ip' => $this->testServer->ipaddress,
                'username' => $this->testServer->username,
                'password' => decrypt($this->testServer->password),
            ],
            'configoptions' => $this->getServiceConfigOptions($service->id),
        ];
    }
}
```

### Step 4: Registrar Module Integration Tests

Test domain registrar integration:

```php
// tests/Integration/RegistrarModuleTest.php

class RegistrarModuleIntegrationTest extends TestCase
{
    private $registrar;
    private $testDomain;
    
    protected function setUp(): void
    {
        parent::setUp();
        
        $credentials = TestCredentials::getDomainRegistrarCredentials();
        $this->registrar = new DomainRegistrarModule($credentials);
        
        $this->testDomain = $this->createTestDomain();
    }
    
    /**
     * Test domain registration
     */
    public function testRegisterDomain(): void
    {
        $params = $this->buildRegistrarParams($this->testDomain);
        
        $result = $this->registrar->RegisterDomain($params);
        
        $this->assertArrayHasKey('success', $result);
        $this->assertTrue($result['success']);
    }
    
    /**
     * Test domain transfer
     */
    public function testTransferDomain(): void
    {
        $params = $this->buildRegistrarParams($this->testDomain);
        $params['eppcode'] = 'transfer_secret_code';
        
        $result = $this->registrar->TransferDomain($params);
        
        $this->assertArrayHasKey('success', $result);
    }
    
    /**
     * Test domain renewal
     */
    public function testRenewDomain(): void
    {
        $params = $this->buildRegistrarParams($this->testDomain);
        $params['regperiod'] = 1;
        
        $result = $this->registrar->RenewDomain($params);
        
        $this->assertArrayHasKey('success', $result);
    }
    
    /**
     * Test nameserver management
     */
    public function testNameserverManagement(): void
    {
        $params = $this->buildRegistrarParams($this->testDomain);
        
        // Get current nameservers
        $current = $this->registrar->GetNameservers($params);
        $this->assertArrayHasKey('ns1', $current);
        
        // Update nameservers
        $params['ns1'] = 'ns1.new-provider.com';
        $params['ns2'] = 'ns2.new-provider.com';
        
        $updateResult = $this->registrar->SaveNameservers($params);
        $this->assertTrue($updateResult['success']);
    }
    
    /**
     * Test DNS management
     */
    public function testDNSManagement(): void
    {
        $params = $this->buildRegistrarParams($this->testDomain);
        
        // Get existing DNS records
        $records = $this->registrar->GetDNS($params);
        $this->assertIsArray($records);
        
        // Add new record
        $params['records'] = [
            ['hostname' => 'www', 'type' => 'A', 'address' => '192.0.2.1'],
        ];
        
        $result = $this->registrar->SaveDNS($params);
        $this->assertTrue($result['success']);
    }
    
    /**
     * Test domain sync
     */
    public function testDomainSync(): void
    {
        $params = $this->buildRegistrarParams($this->testDomain);
        
        $syncResult = $this->registrar->Sync($params);
        
        $this->assertEquals('Active', $syncResult['status']);
        $this->assertArrayHasKey('expiry', $syncResult);
    }
}
```

### Step 5: End-to-End Integration Tests

Test complete business workflows:

```php
// tests/Integration/EndToEndTest.php

class EndToEndIntegrationTest extends TestCase
{
    /**
     * Test complete order fulfillment flow
     */
    public function testCompleteOrderFulfillment(): void
    {
        // 1. Create client
        $client = $this->createTestClient();
        $this->assertNotNull($client->id);
        
        // 2. Create order
        $order = $this->createTestOrder($client->id);
        $this->assertEquals('Pending', $order->status);
        
        // 3. Process payment
        $paymentResult = $this->processTestPayment($order);
        $this->assertTrue($paymentResult['success']);
        
        // 4. Verify order status updated
        $order = Capsule::table('tblorders')
            ->where('id', $order->id)
            ->first();
        $this->assertEquals('Active', $order->status);
        
        // 5. Verify service created
        $service = Capsule::table('tblhosting')
            ->where('orderid', $order->id)
            ->first();
        $this->assertNotNull($service);
        $this->assertEquals('Active', $service->domainstatus);
        
        // 6. Verify provisioning
        $this->assertTrue($this->isServiceProvisioned($service));
        
        // 7. Verify welcome email sent
        $this->assertEmailSent($client->email, 'Welcome');
    }
    
    /**
     * Test domain registration flow
     */
    public function testDomainRegistrationFlow(): void
    {
        // 1. Create client
        $client = $this->createTestClient();
        
        // 2. Create domain order
        $domain = $this->createTestDomainOrder($client->id, 'example-new-domain.com');
        $this->assertEquals('Pending', $domain->status);
        
        // 3. Process payment
        $this->processTestPayment($domain);
        
        // 4. Verify domain registered
        $domain = Capsule::table('tbldomains')
            ->where('id', $domain->id)
            ->first();
        $this->assertEquals('Active', $domain->status);
        
        // 5. Verify nameservers set
        $this->assertNotEmpty($domain->nameservers);
    }
    
    /**
     * Test upgrade flow
     */
    public function testUpgradeFlow(): void
    {
        // 1. Create initial service
        $service = $this->createBasicService();
        
        // 2. Create upgrade order
        $order = $this->createUpgradeOrder($service, 'premium_plan');
        
        // 3. Process payment
        $this->processTestPayment($order);
        
        // 4. Verify service upgraded
        $service = Capsule::table('tblhosting')
            ->where('id', $service->id)
            ->first();
        $this->assertEquals('premium_plan', $service->packageid);
    }
    
    /**
     * Test suspension and reactivation flow
     */
    public function testSuspensionFlow(): void
    {
        // 1. Create active service
        $service = $this->createActiveService();
        
        // 2. Suspend for non-payment
        $this->suspendService($service->id, 'Overdue');
        
        // 3. Verify suspended
        $service = Capsule::table('tblhosting')
            ->where('id', $service->id)
            ->first();
        $this->assertEquals('Suspended', $service->domainstatus);
        
        // 4. Process payment
        $this->processPayment($service->userid, $service->amount);
        
        // 5. Verify reactivated
        $service = Capsule::table('tblhosting')
            ->where('id', $service->id)
            ->first();
        $this->assertEquals('Active', $service->domainstatus);
    }
}
```

### Step 6: API Integration Tests

Test WHMCS API integration:

```php
// tests/Integration/APIIntegrationTest.php

class WHMCSAPIIntegrationTest extends TestCase
{
    private $apiCredentials;
    
    protected function setUp(): void
    {
        parent::setUp();
        
        $this->apiCredentials = [
            'identifier' => getenv('WHMCS_API_IDENTIFIER'),
            'secret' => getenv('WHMCS_API_SECRET'),
            'url' => 'https://test-whmcs.example.com/includes/api.php',
        ];
    }
    
    /**
     * Test client creation via API
     */
    public function testCreateClientAPI(): void
    {
        $clientData = [
            'firstname' => 'Test',
            'lastname' => 'User',
            'email' => 'apitest_' . time() . '@example.com',
            'password2' => 'SecurePass123!',
            'country' => 'US',
            'phonenumber' => '+1-555-1234',
        ];
        
        $result = $this->callAPI('AddClient', $clientData);
        
        $this->assertArrayHasKey('clientid', $result);
        $this->assertGreaterThan(0, $result['clientid']);
    }
    
    /**
     * Test invoice creation via API
     */
    public function testCreateInvoiceAPI(): void
    {
        $clientId = $this->createTestClient()->id;
        
        $invoiceData = [
            'userid' => $clientId,
            'sendinvoice' => true,
            'items' => [
                [
                    'description' => 'Test Service',
                    'amount' => 9.99,
                ],
            ],
        ];
        
        $result = $this->callAPI('CreateInvoice', $invoiceData);
        
        $this->assertArrayHasKey('invoiceid', $result);
    }
    
    /**
     * Test module command execution
     */
    public function testModuleCommandAPI(): void
    {
        $serviceId = $this->createTestService()->id;
        
        $commandData = [
            'serviceid' => $serviceId,
            'action' => 'reboot',
        ];
        
        $result = $this->callAPI('ModuleCommand', $commandData);
        
        $this->assertArrayHasKey('result', $result);
    }
    
    private function callAPI(string $action, array $params): array
    {
        $params['action'] = $action;
        $params['api_key'] = $this->apiCredentials['secret'];
        $params['responsetype'] = 'json';
        
        $ch = curl_init();
        curl_setopt_array($ch, [
            CURLOPT_URL => $this->apiCredentials['url'],
            CURLOPT_POST => true,
            CURLOPT_POSTFIELDS => http_build_query($params),
            CURLOPT_RETURNTRANSFER => true,
        ]);
        
        $response = curl_exec($ch);
        curl_close($ch);
        
        return json_decode($response, true);
    }
}
```

## Best Practices

1. **Isolate test environments** - Use sandbox APIs
2. **Mock external services** - Don't rely on third parties
3. **Test edge cases** - Failure scenarios, timeouts
4. **Verify idempotency** - Same request, same result
5. **Check webhook reliability** - Verify delivery and processing
6. **Test rate limits** - Handle throttling gracefully
7. **Document test scenarios** - Clear test documentation
8. **Automate all tests** - CI/CD integration

## Common Pitfalls to Avoid

1. **Testing in production** - Always use sandbox/test environments
2. **Hardcoded credentials** - Use environment variables
3. **Skipping failure tests** - Test error scenarios
4. **Not cleaning state** - Tests affecting each other
5. **Ignoring timeouts** - Network issues happen
6. **No rollback plan** - What if tests fail mid-way
7. **Missing assertions** - Verify outcomes, not just success
8. **Slow tests** - Integration tests can timeout
