# WHMCS API Client DevKit
# Version: 1.0 | Updated: 2026-05-28

## DevKit Structure

```
devkits/whmcs-api-client/
├── api-client.php         # Main API client class
├── lib/
│   ├── LocalApi.php       # WHMCS Local API client
│   ├── RestApi.php        # REST API client
│   ├── ApiCredentials.php  # Credential management
│   └── ApiResponse.php    # Response handler
├── templates/
│   └── client_example.tpl # Usage examples
└── api-admin.php          # Admin configuration
```

## Main API Client Class

```php
<?php
/**
 * WHMCS API Client
 * DevKit Template
 * 
 * Supports both WHMCS Local API and REST API
 */

if (!defined("WHMCS")) {
    die("Direct access denied");
}

use WHMCS\Database\Capsule;

/**
 * WHMCS API Client Class
 */
class WhmcsApiClient {
    
    private string $apiUrl;
    private string $apiKey;
    private string $accessKey;
    private string $apiType; // 'local' or 'rest'
    private int $timeout = 30;
    private bool $debug = false;
    
    public function __construct(array $config = []) {
        $this->apiUrl = $config['api_url'] ?? '';
        $this->apiKey = $config['api_key'] ?? '';
        $this->accessKey = $config['access_key'] ?? '';
        $this->apiType = $config['api_type'] ?? 'local';
        $this->timeout = $config['timeout'] ?? 30;
        $this->debug = $config['debug'] ?? false;
    }
    
    /**
     * Execute Local API call
     */
    public function localApi(string $action, array $params = []): array {
        if ($this->apiType !== 'local') {
            throw new \Exception('API client not configured for Local API');
        }
        
        $postData = array_merge([
            'action' => $action,
            'username' => $this->apiKey,
            'password' => $this->accessKey,
            'responsetype' => 'json',
        ], $params);
        
        return $this->makeRequest($postData);
    }
    
    /**
     * Execute REST API call
     */
    public function restApi(string $method, string $endpoint, array $data = []): array {
        if ($this->apiType !== 'rest') {
            throw new \Exception('API client not configured for REST API');
        }
        
        $url = rtrim($this->apiUrl, '/') . '/' . ltrim($endpoint, '/');
        
        return $this->makeHttpRequest($method, $url, $data);
    }
    
    /**
     * Make Local API request
     */
    private function makeRequest(array $postData): array {
        $ch = curl_init();
        curl_setopt_array($ch, [
            CURLOPT_URL => $this->apiUrl ?: 'https://www.whmcs.com/includes/api.php',
            CURLOPT_POST => true,
            CURLOPT_POSTFIELDS => http_build_query($postData),
            CURLOPT_RETURNTRANSFER => true,
            CURLOPT_TIMEOUT => $this->timeout,
            CURLOPT_SSL_VERIFYPEER => true,
            CURLOPT_SSL_VERIFYHOST => 2,
        ]);
        
        $response = curl_exec($ch);
        $error = curl_error($ch);
        $httpCode = curl_getinfo($ch, CURLINFO_HTTP_CODE);
        curl_close($ch);
        
        if ($this->debug) {
            $this->log('Local API Request', $postData, $response, $httpCode);
        }
        
        if ($response === false) {
            throw new \Exception('cURL Error: ' . $error);
        }
        
        $result = json_decode($response, true);
        
        if (json_last_error() !== JSON_ERROR_NONE) {
            throw new \Exception('Invalid JSON response: ' . $response);
        }
        
        return $result;
    }
    
    /**
     * Make HTTP REST request
     */
    private function makeHttpRequest(string $method, string $url, array $data): array {
        $ch = curl_init();
        
        $headers = [
            'Authorization: Bearer ' . $this->apiKey,
            'Content-Type: application/json',
            'Accept: application/json',
        ];
        
        $curlOptions = [
            CURLOPT_URL => $url,
            CURLOPT_RETURNTRANSFER => true,
            CURLOPT_TIMEOUT => $this->timeout,
            CURLOPT_HTTPHEADER => $headers,
        ];
        
        if (in_array($method, ['POST', 'PUT', 'PATCH'])) {
            $curlOptions[CURLOPT_POST] = true;
            $curlOptions[CURLOPT_POSTFIELDS] = json_encode($data);
        }
        
        curl_setopt_array($ch, $curlOptions);
        
        $response = curl_exec($ch);
        $error = curl_error($ch);
        $httpCode = curl_getinfo($ch, CURLINFO_HTTP_CODE);
        curl_close($ch);
        
        if ($this->debug) {
            $this->log('REST API Request', ['method' => $method, 'url' => $url], $response, $httpCode);
        }
        
        if ($response === false) {
            throw new \Exception('cURL Error: ' . $error);
        }
        
        $result = json_decode($response, true);
        
        if (json_last_error() !== JSON_ERROR_NONE) {
            throw new \Exception('Invalid JSON response: ' . $response);
        }
        
        return $result;
    }
    
    private function log(string $type, $data, $response, int $httpCode): void {
        Capsule::table('mod_whmcs_api_client_logs')->insert([
            'type' => $type,
            'request_data' => is_array($data) ? json_encode($data) : $data,
            'response_data' => is_string($response) ? $response : json_encode($response),
            'http_code' => $httpCode,
            'created_at' => date('Y-m-d H:i:s'),
        ]);
    }
    
    // Client operations
    public function getClients(array $params = []): array {
        return $this->localApi('GetClients', $params);
    }
    
    public function getClient(int $clientId): array {
        return $this->localApi('GetClient', ['clientid' => $clientId]);
    }
    
    public function addClient(array $params): array {
        return $this->localApi('AddClient', $params);
    }
    
    public function updateClient(int $clientId, array $params): array {
        return $this->localApi('UpdateClient', array_merge(['clientid' => $clientId], $params));
    }
    
    // Service operations
    public function getClientsProducts(int $clientId): array {
        return $this->localApi('GetClientsProducts', ['clientid' => $clientId]);
    }
    
    public function getProducts(array $params = []): array {
        return $this->localApi('GetProducts', $params);
    }
    
    public function getOrders(array $params = []): array {
        return $this->localApi('GetOrders', $params);
    }
    
    // Invoice operations
    public function getInvoices(array $params = []): array {
        return $this->localApi('GetInvoices', $params);
    }
    
    public function getInvoice(int $invoiceId): array {
        return $this->localApi('GetInvoice', ['invoiceid' => $invoiceId]);
    }
    
    public function addInvoicePayment(int $invoiceId, string $transactionId, float $amount): array {
        return $this->localApi('AddInvoicePayment', [
            'invoiceid' => $invoiceId,
            'transid' => $transactionId,
            'amount' => $amount,
        ]);
    }
    
    // Domain operations
    public function getDomains(array $params = []): array {
        return $this->localApi('GetDomains', $params);
    }
    
    // Ticket operations
    public function getTickets(array $params = []): array {
        return $this->localApi('GetTickets', $params);
    }
    
    public function openTicket(array $params): array {
        return $this->localApi('OpenTicket', $params);
    }
    
    public function addTicketReply(int $ticketId, string $message): array {
        return $this->localApi('AddTicketReply', [
            'ticketid' => $ticketId,
            'message' => $message,
        ]);
    }
}
```

## Local API Wrapper

```php
<?php
/**
 * WHMCS Local API Client
 */

namespace WhmcsApi;

class LocalApiClient {
    
    private string $apiUrl;
    private string $username;
    private string $password;
    private int $timeout = 30;
    
    public function __construct(string $apiUrl, string $username, string $password, int $timeout = 30) {
        $this->apiUrl = rtrim($apiUrl, '/') . '/includes/api.php';
        $this->username = $username;
        $this->password = $password;
        $this->timeout = $timeout;
    }
    
    /**
     * Call WHMCS Local API action
     */
    public function call(string $action, array $params = []): array {
        $postData = array_merge([
            'action' => $action,
            'username' => $this->username,
            'password' => $this->password,
            'responsetype' => 'json',
        ], $params);
        
        $ch = curl_init();
        curl_setopt_array($ch, [
            CURLOPT_URL => $this->apiUrl,
            CURLOPT_POST => true,
            CURLOPT_POSTFIELDS => http_build_query($postData),
            CURLOPT_RETURNTRANSFER => true,
            CURLOPT_TIMEOUT => $this->timeout,
        ]);
        
        $response = curl_exec($ch);
        curl_close($ch);
        
        return json_decode($response, true) ?? [];
    }
    
    // Convenience methods
    public function getClient(int $id): array {
        return $this->call('GetClient', ['clientid' => $id]);
    }
    
    public function getClients(array $filters = []): array {
        return $this->call('GetClients', $filters);
    }
    
    public function addClient(array $data): array {
        return $this->call('AddClient', $data);
    }
    
    public function getProducts(array $filters = []): array {
        return $this->call('GetProducts', $filters);
    }
    
    public function getInvoices(array $filters = []): array {
        return $this->call('GetInvoices', $filters);
    }
    
    public function getDomains(array $filters = []): array {
        return $this->call('GetDomains', $filters);
    }
    
    public function getOrders(array $filters = []): array {
        return $this->call('GetOrders', $filters);
    }
    
    public function getTickets(array $filters = []): array {
        return $this->call('GetTickets', $filters);
    }
    
    public function openTicket(array $data): array {
        return $this->call('OpenTicket', $data);
    }
    
    public function addInvoicePayment(int $invoiceId, string $transId, float $amount): array {
        return $this->call('AddInvoicePayment', [
            'invoiceid' => $invoiceId,
            'transid' => $transId,
            'amount' => $amount,
        ]);
    }
    
    public function acceptOrder(int $orderId): array {
        return $this->call('AcceptOrder', ['orderid' => $orderId]);
    }
    
    public function createInvoice(array $params): array {
        return $this->call('CreateInvoice', $params);
    }
}
```

## REST API Client

```php
<?php
/**
 * WHMCS REST API Client
 */

namespace WhmcsApi;

class RestApiClient {
    
    private string $baseUrl;
    private string $apiKey;
    private string $secretKey;
    private int $timeout = 30;
    
    public function __construct(string $baseUrl, string $apiKey, string $secretKey = '', int $timeout = 30) {
        $this->baseUrl = rtrim($baseUrl, '/');
        $this->apiKey = $apiKey;
        $this->secretKey = $secretKey;
        $this->timeout = $timeout;
    }
    
    /**
     * Make GET request
     */
    public function get(string $endpoint, array $params = []): array {
        $url = $this->buildUrl($endpoint, $params);
        return $this->request('GET', $url);
    }
    
    /**
     * Make POST request
     */
    public function post(string $endpoint, array $data = []): array {
        $url = $this->baseUrl . '/' . ltrim($endpoint, '/');
        return $this->request('POST', $url, $data);
    }
    
    /**
     * Make PUT request
     */
    public function put(string $endpoint, array $data = []): array {
        $url = $this->baseUrl . '/' . ltrim($endpoint, '/');
        return $this->request('PUT', $url, $data);
    }
    
    /**
     * Make DELETE request
     */
    public function delete(string $endpoint): array {
        $url = $this->baseUrl . '/' . ltrim($endpoint, '/');
        return $this->request('DELETE', $url);
    }
    
    /**
     * Build URL with query parameters
     */
    private function buildUrl(string $endpoint, array $params): string {
        $url = $this->baseUrl . '/' . ltrim($endpoint, '/');
        if (!empty($params)) {
            $url .= '?' . http_build_query($params);
        }
        return $url;
    }
    
    /**
     * Make HTTP request
     */
    private function request(string $method, string $url, array $data = []): array {
        $ch = curl_init();
        
        $headers = [
            'Authorization: Bearer ' . $this->apiKey,
            'Content-Type: application/json',
            'Accept: application/json',
        ];
        
        $options = [
            CURLOPT_URL => $url,
            CURLOPT_RETURNTRANSFER => true,
            CURLOPT_TIMEOUT => $this->timeout,
            CURLOPT_HTTPHEADER => $headers,
        ];
        
        if ($method !== 'GET' && !empty($data)) {
            $options[CURLOPT_POST] = true;
            $options[CURLOPT_POSTFIELDS] = json_encode($data);
        }
        
        curl_setopt_array($ch, $options);
        
        $response = curl_exec($ch);
        $error = curl_error($ch);
        $httpCode = curl_getinfo($ch, CURLINFO_HTTP_CODE);
        curl_close($ch);
        
        if ($response === false) {
            throw new \Exception('cURL Error: ' . $error);
        }
        
        $result = json_decode($response, true);
        
        if (json_last_error() !== JSON_ERROR_NONE) {
            throw new \Exception('Invalid JSON response');
        }
        
        return [
            'success' => $httpCode >= 200 && $httpCode < 300,
            'data' => $result,
            'status_code' => $httpCode,
        ];
    }
    
    // REST API endpoints
    public function getClients(array $params = []): array {
        return $this->get('/api/v1/clients', $params);
    }
    
    public function getClient(int $id): array {
        return $this->get("/api/v1/clients/{$id}");
    }
    
    public function createClient(array $data): array {
        return $this->post('/api/v1/clients', $data);
    }
    
    public function updateClient(int $id, array $data): array {
        return $this->put("/api/v1/clients/{$id}", $data);
    }
    
    public function deleteClient(int $id): array {
        return $this->delete("/api/v1/clients/{$id}");
    }
    
    public function getProducts(array $params = []): array {
        return $this->get('/api/v1/products', $params);
    }
    
    public function getInvoices(array $params = []): array {
        return $this->get('/api/v1/invoices', $params);
    }
    
    public function getDomains(array $params = []): array {
        return $this->get('/api/v1/domains', $params);
    }
    
    public function getOrders(array $params = []): array {
        return $this->get('/api/v1/orders', $params);
    }
    
    public function getTickets(array $params = []): array {
        return $this->get('/api/v1/tickets', $params);
    }
}
```

## API Credentials Manager

```php
<?php
/**
 * API Credentials Manager
 */

namespace WhmcsApi;

use WHMCS\Database\Capsule;

class ApiCredentials {
    
    /**
     * Store API credentials
     */
    public static function store(string $name, string $apiUrl, string $apiKey, string $apiSecret = '', string $type = 'local'): int {
        $existing = Capsule::table('mod_whmcs_api_credentials')
            ->where('name', $name)
            ->first();
        
        if ($existing) {
            Capsule::table('mod_whmcs_api_credentials')
                ->where('id', $existing->id)
                ->update([
                    'api_url' => $apiUrl,
                    'api_key' => encrypt($apiKey),
                    'api_secret' => $apiSecret ? encrypt($apiSecret) : '',
                    'api_type' => $type,
                    'updated_at' => date('Y-m-d H:i:s'),
                ]);
            return $existing->id;
        }
        
        return Capsule::table('mod_whmcs_api_credentials')->insertGetId([
            'name' => $name,
            'api_url' => $apiUrl,
            'api_key' => encrypt($apiKey),
            'api_secret' => $apiSecret ? encrypt($apiSecret) : '',
            'api_type' => $type,
            'is_active' => 1,
            'created_at' => date('Y-m-d H:i:s'),
        ]);
    }
    
    /**
     * Get credentials by name
     */
    public static function get(string $name): ?array {
        $row = Capsule::table('mod_whmcs_api_credentials')
            ->where('name', $name)
            ->where('is_active', 1)
            ->first();
        
        if (!$row) {
            return null;
        }
        
        return [
            'id' => $row->id,
            'name' => $row->name,
            'api_url' => $row->api_url,
            'api_key' => decrypt($row->api_key),
            'api_secret' => $row->api_secret ? decrypt($row->api_secret) : '',
            'api_type' => $row->api_type,
        ];
    }
    
    /**
     * Get all credentials
     */
    public static function all(): array {
        return Capsule::table('mod_whmcs_api_credentials')
            ->where('is_active', 1)
            ->get()
            ->map(function($row) {
                return [
                    'id' => $row->id,
                    'name' => $row->name,
                    'api_url' => $row->api_url,
                    'api_type' => $row->api_type,
                ];
            })
            ->toArray();
    }
    
    /**
     * Delete credentials
     */
    public static function delete(string $name): bool {
        return Capsule::table('mod_whmcs_api_credentials')
            ->where('name', $name)
            ->delete() > 0;
    }
    
    /**
     * Test connection
     */
    public static function test(string $name): bool {
        $creds = self::get($name);
        if (!$creds) {
            return false;
        }
        
        $client = $creds['api_type'] === 'local'
            ? new LocalApiClient($creds['api_url'], $creds['api_key'], $creds['api_secret'])
            : new RestApiClient($creds['api_url'], $creds['api_key'], $creds['api_secret']);
        
        try {
            if ($creds['api_type'] === 'local') {
                $result = $client->call('GetClients', ['limitnum' => 1]);
                return isset($result['client']);
            } else {
                $result = $client->getClients(['limit' => 1]);
                return isset($result['data']);
            }
        } catch (\Exception $e) {
            return false;
        }
    }
}
```

## Usage Example

```php
<?php
// Initialize Local API client
$localClient = new WhmcsApiClient([
    'api_url' => 'https://your-whmcs.com/includes/api.php',
    'api_key' => 'admin_username',
    'access_key' => 'admin_password_hash',
    'api_type' => 'local',
]);

// Get clients
$clients = $localClient->getClients([
    'limitnum' => 10,
    'sorting' => 'ASC',
]);

// Get single client
$client = $localClient->getClient(12345);

// Add new client
$newClient = $localClient->addClient([
    'firstname' => 'John',
    'lastname' => 'Doe',
    'email' => 'john@example.com',
    'address1' => '123 Main St',
    'city' => 'New York',
    'country' => 'US',
    'phonenumber' => '+1-555-1234',
]);

// Initialize REST API client
$restClient = new WhmcsApiClient([
    'api_url' => 'https://your-whmcs.com/api/v1',
    'api_key' => 'your-api-key',
    'api_type' => 'rest',
]);

// Get products via REST
$products = $restClient->restApi('GET', '/products');

// Create client via REST
$client = $restClient->restApi('POST', '/clients', [
    'first_name' => 'Jane',
    'last_name' => 'Smith',
    'email' => 'jane@example.com',
]);
```

## Checklist

```
Pre-Dev:
□ Define API type (local vs REST)
□ Plan authentication method
□ Identify required API actions
□ Determine rate limits
□ Design error handling

Development:
□ Create main API client class
□ Implement LocalApi wrapper
□ Implement RestApi wrapper
□ Create ApiCredentials manager
□ Add credential storage/retrieval
□ Implement API call methods
□ Add debug logging
□ Create connection test method
□ Build admin configuration UI
□ Add timeout handling
□ Implement error handling

Testing:
□ Test Local API authentication
□ Test REST API authentication
□ Test all API actions
□ Verify error handling
□ Test connection timeout
□ Test credential storage
□ Test debug logging
□ Verify data formats
```