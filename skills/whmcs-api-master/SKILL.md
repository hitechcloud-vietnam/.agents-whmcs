# WHMCS API Master

## Overview
Master skill for WHMCS API integration and development. Covers API authentication, endpoints, request handling, and custom API creation.

## WHMCS API Structure

### API Configuration

```php
<?php
// /includes/api_config.php

return [
    'api_key' => 'your-api-key-here',
    'api_username' => 'admin_username',
    'api_url' => 'https://your-whmcs.com/includes/api.php',
    'access_key' => 'optional-access-key',
    'debug_mode' => false,
];
```

### API Client Class

```php
<?php
// /includes/api/WhmcsApiClient.php

namespace WHMCS\Api;

class WhmcsApiClient
{
    private $apiUrl;
    private $apiKey;
    private $apiUsername;
    private $debugMode;

    public function __construct(array $config)
    {
        $this->apiUrl = rtrim($config['api_url'], '/');
        $this->apiKey = $config['api_key'];
        $this->apiUsername = $config['api_username'] ?? '';
        $this->debugMode = $config['debug_mode'] ?? false;
    }

    public function post(string $action, array $data = []): array
    {
        $data['action'] = $action;
        $data['username'] = $this->apiUsername;
        $data['password'] = $this->apiKey;
        $data['responsetype'] = 'json';

        $ch = curl_init($this->apiUrl);

        curl_setopt_array($ch, [
            CURLOPT_POST => true,
            CURLOPT_POSTFIELDS => http_build_query($data),
            CURLOPT_RETURNTRANSFER => true,
            CURLOPT_TIMEOUT => 60,
            CURLOPT_SSL_VERIFYPEER => true,
            CURLOPT_HTTPHEADER => [
                'Content-Type: application/x-www-form-urlencoded',
            ],
        ]);

        $response = curl_exec($ch);
        $httpCode = curl_getinfo($ch, CURLINFO_HTTP_CODE);
        $error = curl_error($ch);
        curl_close($ch);

        if ($this->debugMode) {
            logActivity("[WHMCS API] Request: " . json_encode($data));
            logActivity("[WHMCS API] Response: " . $response);
        }

        if ($error) {
            throw new \Exception('cURL Error: ' . $error);
        }

        $result = json_decode($response, true);

        if (isset($result['result']) && $result['result'] === 'error') {
            throw new \Exception($result['message'] ?? 'API Error');
        }

        return $result;
    }

    public function get(string $action, array $params = []): array
    {
        $params['action'] = $action;
        $params['username'] = $this->apiUsername;
        $params['password'] = $this->apiKey;
        $params['responsetype'] = 'json';

        $url = $this->apiUrl . '?' . http_build_query($params);

        $ch = curl_init($url);

        curl_setopt_array($ch, [
            CURLOPT_RETURNTRANSFER => true,
            CURLOPT_TIMEOUT => 60,
            CURLOPT_SSL_VERIFYPEER => true,
        ]);

        $response = curl_exec($ch);
        $httpCode = curl_getinfo($ch, CURLINFO_HTTP_CODE);
        $error = curl_error($ch);
        curl_close($ch);

        if ($error) {
            throw new \Exception('cURL Error: ' . $error);
        }

        return json_decode($response, true);
    }

    // Client operations
    public function getClients(array $params = []): array
    {
        return $this->post('getclients', $params);
    }

    public function getClient(array $params): array
    {
        return $this->post('getclient', $params);
    }

    public function addClient(array $params): array
    {
        return $this->post('addclient', $params);
    }

    public function updateClient(array $params): array
    {
        return $this->post('updateclient', $params);
    }

    public function deleteClient(array $params): array
    {
        return $this->post('deleteclient', $params);
    }

    public function validateClientLogin(array $params): array
    {
        return $this->post('validatelogin', $params);
    }

    // Order operations
    public function getOrders(array $params = []): array
    {
        return $this->post('getorders', $params);
    }

    public function getOrder(array $params): array
    {
        return $this->post('getorder', $params);
    }

    public function acceptOrder(array $params): array
    {
        return $this->post('acceptorder', $params);
    }

    public function pendingOrder(array $params): array
    {
        return $this->post('pendingorder', $params);
    }

    public function cancelOrder(array $params): array
    {
        return $this->post('cancelorder', $params);
    }

    public function fraudOrder(array $params): array
    {
        return $this->post('('fraudorder', $params);
    }

    // Invoice operations
    public function getInvoices(array $params = []): array
    {
        return $this->post('getinvoices', $params);
    }

    public function getInvoice(array $params): array
    {
        return $this->post('getinvoice', $params);
    }

    public function createInvoice(array $params): array
    {
        return $this->post('createinvoice', $params);
    }

    public function addInvoicePayment(array $params): array
    {
        return $this->post('addinvoicepayment', $params);
    }

    public function applyCredit(array $params): array
    {
        return $this->post('applycredit', $params);
    }

    public function addCredit(array $params): array
    {
        return $this->post('addcredit', $params);
    }

    // Service operations
    public function getProducts(array $params = []): array
    {
        return $this->post('getproducts', $params);
    }

    public function getClientsProducts(array $params = []): array
    {
        return $this->post('getclientsproducts', $params);
    }

    public function getService(array $params): array
    {
        return $this->post('getservice', $params);
    }

    public function createService(array $params): array
    {
        return $this->post('create service', $params);
    }

    public function updateService(array $params): array
    {
        return $this->post('updateservice', $params);
    }

    public function terminateService(array $params): array
    {
        return $this->post('terminateservice', $params);
    }

    public function suspendService(array $params): array
    {
        return $this->post('suspendservice', $params);
    }

    public function unsuspendService(array $params): array
    {
        return $this->post('unsuspendservice', $params);
    }

    public function changeServicePackage(array $params): array
    {
        return $this->post('changeservicepackage', $params);
    }

    // Domain operations
    public function getDomains(array $params = []): array
    {
        return $this->post('getdomains', $params);
    }

    public function getDomain(array $params): array
    {
        return $this->post('getdomain', $params);
    }

    public function registerDomain(array $params): array
    {
        return $this->post('registerdomain', $params);
    }

    public function transferDomain(array $params): array
    {
        return $this->post('transferdomain', $params);
    }

    public function renewDomain(array $params): array
    {
        return $this->post('renewdomain', $params);
    }

    public function updateDomain(array $params): array
    {
        return $this->post('updatedomain', $params);
    }

    // Ticket operations
    public function getTickets(array $params = []): array
    {
        return $this->post('gettickets', $params);
    }

    public function getTicket(array $params): array
    {
        return $this->post('getticket', $params);
    }

    public function openTicket(array $params): array
    {
        return $this->post('openticket', $params);
    }

    public function replyTicket(array $params): array
    {
        return $this->post('replyticket', $params);
    }

    public function addTicketNote(array $params): array
    {
        return $this->post('addticketnote', $params);
    }

    public function closeTicket(array $params): array
    {
        return $this->post('closeticket', $params);
    }

    // Utility operations
    public function getActivityLog(array $params = []): array
    {
        return $this->post('getactivitylog', $params);
    }

    public function getStats(): array
    {
        return $this->post('getstats');
    }

    public function addTransaction(array $params): array
    {
        return $this->post('addtransaction', $params);
    }

    public function updateInvoice(array $params): array
    {
        return $this->post('updateinvoice', $params);
    }

    public function capturePayment(array $params): array
    {
        return $this->post('capturepayment', $params);
    }

    public function massMassUpdate(array $params): array
    {
        return $this->post('massmassupdate', $params);
    }

    public function emailTicket(array $params): array
    {
        return $this->post('emailticket', $params);
    }

    public function getEmailTemplates(array $params = []): array
    {
        return $this->post('getemailtemplates', $params);
    }

    public function sendEmail(array $params): array
    {
        return $this->post('sendemail', $params);
    }

    public function sendEmailTemplate(array $params): array
    {
        return $this->post('sendemailtemplate', $params);
    }
}
```

## Custom API Endpoints

```php
<?php
// /includes/api/custom/my_custom_endpoint.php

if (!defined("WHMCS")) {
    die("This file cannot be accessed directly");
}

require_once __DIR__ . '/../../../init.php';
require_once __DIR__ . '/WhmcsApiClient.php';

// Verify authentication
if (!verify_api_authentication()) {
    http_response_code(401);
    echo json_encode(['error' => 'Unauthorized']);
    exit;
}

// Get request method and parameters
$method = $_SERVER['REQUEST_METHOD'];
$requestBody = json_decode(file_get_contents('php://input'), true) ?? [];

// Route the request
$action = $_GET['action'] ?? '';

switch ($action) {
    case 'custom_action':
        handleCustomAction($requestBody);
        break;

    case 'bulk_operation':
        handleBulkOperation($requestBody);
        break;

    case 'stats':
        handleStats($requestBody);
        break;

    default:
        http_response_code(404);
        echo json_encode(['error' => 'Endpoint not found']);
}

function handleCustomAction(array $params): void
{
    // Validate required parameters
    $required = ['client_id', 'service_id'];
    foreach ($required as $field) {
        if (empty($params[$field])) {
            http_response_code(400);
            echo json_encode(['error' => "Missing required field: {$field}"]);
            return;
        }
    }

    // Perform the action
    $client = \WHMCS\User\Client::find($params['client_id']);
    if (!$client) {
        http_response_code(404);
        echo json_encode(['error' => 'Client not found']);
        return;
    }

    $service = \WHMCS\Service\Service::find($params['service_id']);
    if (!$service) {
        http_response_code(404);
        echo json_encode(['error' => 'Service not found']);
        return;
    }

    // Process and return result
    $result = [
        'success' => true,
        'client_id' => $client->id,
        'service_id' => $service->id,
        'data' => [
            'client_name' => $client->fullName,
            'service_domain' => $service->domain,
            'service_status' => $service->status,
        ],
    ];

    http_response_code(200);
    echo json_encode($result);
}

function handleBulkOperation(array $params): void
{
    $ids = $params['ids'] ?? [];
    $operation = $params['operation'] ?? '';

    if (empty($ids) || empty($operation)) {
        http_response_code(400);
        echo json_encode(['error' => 'Missing required parameters']);
        return;
    }

    $results = [];
    foreach ($ids as $id) {
        try {
            $result = processBulkOperation($id, $operation);
            $results[] = [
                'id' => $id,
                'success' => true,
                'result' => $result,
            ];
        } catch (\Exception $e) {
            $results[] = [
                'id' => $id,
                'success' => false,
                'error' => $e->getMessage(),
            ];
        }
    }

    http_response_code(200);
    echo json_encode([
        'success' => true,
        'results' => $results,
    ]);
}

function handleStats(array $params): void
{
    $stats = [
        'total_clients' => \Illuminate\Database\Capsule\Manager::table('tblclients')->count(),
        'active_services' => \Illuminate\Database\Capsule\Manager::table('tblhosting')
            ->where('domainstatus', 'Active')->count(),
        'pending_orders' => \Illuminate\Database\Capsule\Manager::table('tblorders')
            ->where('status', 'Pending')->count(),
        'open_tickets' => \Illuminate\Database\Capsule\Manager::table('tbltickets')
            ->where('status', 'Open')->count(),
        'monthly_revenue' => \Illuminate\Database\Capsule\Manager::table('tblinvoices')
            ->where('status', 'Paid')
            ->where('date', '>=', date('Y-m-01'))
            ->sum('total'),
    ];

    http_response_code(200);
    echo json_encode([
        'success' => true,
        'stats' => $stats,
    ]);
}

function verify_api_authentication(): bool
{
    // Check API key header
    $apiKey = $_SERVER['HTTP_X_API_KEY'] ?? '';
    $expectedKey = \WHMCS\Config\Setting::getValue('api_key');

    return hash_equals($expectedKey, $apiKey);
}
```

## REST API Wrapper

```php
<?php
// /includes/api/RestApiClient.php

namespace WHMCS\Api;

class RestApiClient
{
    private $baseUrl;
    private $apiKey;
    private $accessKey;

    public function __construct(string $baseUrl, string $apiKey, string $accessKey = '')
    {
        $this->baseUrl = rtrim($baseUrl, '/');
        $this->apiKey = $apiKey;
        $this->accessKey = $accessKey;
    }

    public function request(string $method, string $endpoint, array $data = [], array $headers = []): array
    {
        $url = $this->baseUrl . $endpoint;

        $defaultHeaders = [
            'Authorization: Bearer ' . $this->apiKey,
            'Content-Type: application/json',
            'Accept: application/json',
        ];

        if ($this->accessKey) {
            $defaultHeaders[] = 'X-Access-Key: ' . $this->accessKey;
        }

        $allHeaders = array_merge($defaultHeaders, $headers);

        $ch = curl_init();

        curl_setopt_array($ch, [
            CURLOPT_URL => $url,
            CURLOPT_RETURNTRANSFER => true,
            CURLOPT_TIMEOUT => 60,
            CURLOPT_HTTPHEADER => $allHeaders,
        ]);

        if (in_array($method, ['POST', 'PUT', 'PATCH'])) {
            curl_setopt($ch, CURLOPT_POST, true);
            curl_setopt($ch, CURLOPT_POSTFIELDS, json_encode($data));
        }

        if ($method === 'PUT' || $method === 'PATCH') {
            curl_setopt($ch, CURLOPT_CUSTOMREQUEST, $method);
        }

        if ($method === 'DELETE') {
            curl_setopt($ch, CURLOPT_CUSTOMREQUEST, 'DELETE');
        }

        $response = curl_exec($ch);
        $httpCode = curl_getinfo($ch, CURLINFO_HTTP_CODE);
        $error = curl_error($ch);
        curl_close($ch);

        if ($error) {
            throw new \Exception('cURL Error: ' . $error);
        }

        $result = json_decode($response, true);

        if ($httpCode >= 400) {
            throw new \Exception(
                $result['message'] ?? 'API Error: HTTP ' . $httpCode
            );
        }

        return $result;
    }

    public function get(string $endpoint, array $params = []): array
    {
        if (!empty($params)) {
            $endpoint .= '?' . http_build_query($params);
        }
        return $this->request('GET', $endpoint);
    }

    public function post(string $endpoint, array $data): array
    {
        return $this->request('POST', $endpoint, $data);
    }

    public function put(string $endpoint, array $data): array
    {
        return $this->request('PUT', $endpoint, $data);
    }

    public function patch(string $endpoint, array $data): array
    {
        return $this->request('PATCH', $endpoint, $data);
    }

    public function delete(string $endpoint): array
    {
        return $this->request('DELETE', $endpoint);
    }
}
```

## Best Practices

1. **Authentication**: Always use secure API keys and HTTPS
2. **Error Handling**: Implement proper error handling for all API calls
3. **Rate Limiting**: Respect API rate limits
4. **Caching**: Cache API responses when appropriate
5. **Logging**: Log all API requests and responses
6. **Validation**: Validate all input parameters
7. **Timeout**: Set appropriate timeouts (60 seconds recommended)
8. **Retry Logic**: Implement retry with exponential backoff
9. **Pagination**: Handle paginated responses properly
10. **Documentation**: Document all custom API endpoints
