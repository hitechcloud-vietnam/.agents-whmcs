# WHMCS API Testing Workflow

## Purpose
Guide developers through testing WHMCS API integrations.

## Prerequisites
- WHMCS installation
- API credentials
- Testing framework
- PHP skills

## Steps

### Phase 1: Testing Setup

1. Test environment
   ```
   Testing Requirements:
   - Staging WHMCS instance
   - Test API credentials
   - Isolated database
   - Test data set
   ```

2. PHPUnit configuration
   ```xml
   <!-- phpunit.xml -->
   <phpunit bootstrap="tests/bootstrap.php">
       <testsuites>
           <testsuite name="WHMCS API Tests">
               <directory>tests/Api</directory>
           </testsuite>
       </testsuites>
   </phpunit>
   ```

### Phase 2: Test Cases

1. API client tests
   ```php
   class WHMCSApiClientTest extends PHPUnit_Framework_TestCase {
       private $api;
       
       protected function setUp() {
           $this->api = new WHMCSApiClient([
               'url' => TEST_API_URL,
               'username' => TEST_USERNAME,
               'password' => TEST_PASSWORD,
           ]);
       }
       
       public function testGetClients() {
           $result = $this->api->call('GetClients', ['limitnum' => 10]);
           $this->assertArrayHasKey('clients', $result);
       }
       
       public function testCreateClient() {
           $result = $this->api->call('AddClient', [
               'firstname' => 'Test',
               'lastname' => 'User',
               'email' => 'test@example.com',
           ]);
           
           $this->assertEquals('success', $result['result']);
           $this->assertArrayHasKey('clientid', $result);
       }
       
       public function testAuthenticationFailure() {
           $this->expectException(ApiAuthException::class);
           
           $api = new WHMCSApiClient([
               'username' => 'invalid',
               'password' => 'wrong',
           ]);
           
           $api->call('GetClients');
       }
   }
   ```

2. Integration tests
   ```php
   class IntegrationTest extends PHPUnit_Framework_TestCase {
       public function testFullOrderFlow() {
           // Create client
           $client = $this->createTestClient();
           
           // Create order
           $order = $this->createTestOrder($client['clientid']);
           
           // Verify service created
           $this->assertNotEmpty($order['orderid']);
           
           // Cleanup
           $this->deleteTestClient($client['clientid']);
       }
   }
   ```

### Phase 3: Mock Testing

1. Mock API responses
   ```php
   class MockApiClient {
       public function mockCall($action, $params) {
           $mocks = [
               'GetClients' => ['result' => 'success', 'clients' => []],
               'GetInvoices' => ['result' => 'success', 'invoices' => []],
           ];
           
           return $mocks[$action] ?? ['result' => 'error'];
       }
   }
   ```

### Phase 4: Test Coverage

1. Coverage report
   ```
   Test Coverage Areas:
   - Authentication
   - CRUD operations
   - Error handling
   - Rate limiting
   - Webhooks
   ```

## Related Workflows
- whmcs-api-authentication
- whmcs-api-error-handling
- whmcs-api-integration
