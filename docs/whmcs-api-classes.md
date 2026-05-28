# WHMCS API Classes Reference

**Version:** 8.x | **Updated:** 2026-05-29
**Related Skills:** `whmcs-api-integration`, `webhook-events-reference`, `api-gateway`

---

## Overview

WHMCS provides two primary API interfaces for external integration:
1. **localAPI** - Internal PHP API for hooks and modules
2. **REST API** - External HTTP API for third-party integrations

---

## localAPI Class

The localAPI allows modules and hooks to call WHMCS functionality internally using the `localApi()` method.

### Basic Usage

```php
<?php
use WHMCS\Authentication\Users\User;

/**
 * Execute local API call
 *
 * @param string $action API action name
 * @param array $params  Parameters for the action
 * @param string $user   Admin username (optional, uses session if omitted)
 * @return array
 */
$results = localApi($action, $params, $user);
```

### Common Actions

#### Client Management

```php
// Get client details
$results = localApi('GetClientDetails', [
    'client_id' => 123,
    'stats' => true,
    'customfields' => true,
]);

// Add new client
$results = localApi('AddClient', [
    'firstname' => 'John',
    'lastname' => 'Doe',
    'email' => 'john@example.com',
    'password2' => 'SecurePass123',
    'country' => 'US',
    'currency' => 1,
]);

// Update client
$results = localApi('UpdateClient', [
    'client_id' => 123,
    'firstname' => 'Jane',
    'notes' => 'VIP customer',
]);

// Close client
$results = localApi('CloseClient', [
    'client_id' => 123,
    'close_related_services' => true,
]);
```

#### Invoice Operations

```php
// Create invoice
$results = localApi('CreateInvoice', [
    'user_id' => 123,
    'sendinvoice' => true,
    'itemdescription0' => 'Web Hosting',
    'itemamount0' => '9.99',
    'duedate' => date('Y-m-d', strtotime('+7 days')),
]);

// Add payment
$results = localApi('AddInvoicePayment', [
    'invoice_id' => 456,
    'transaction_id' => 'TXN-' . time(),
    'amount' => 9.99,
    'gateway' => 'paypal',
]);

// Get invoices
$results = localApi('GetInvoices', [
    'user_id' => 123,
    'status' => 'Unpaid',
    'limitstart' => 0,
    'limitnum' => 25,
]);
```

#### Service Management

```php
// Create service
$results = localApi('ModuleCreate', [
    'serviceid' => 789,
    'username' => 'newuser',
]);

// Suspend service
$results = localApi('ModuleSuspend', [
    'serviceid' => 789,
    'suspend_reason' => 'Non-payment',
]);

// Terminate service
$results = localApi('ModuleTerminate', [
    'serviceid' => 789,
]);

// Change password
$results = localApi('ModuleChangePassword', [
    'serviceid' => 789,
    'newpassword' => 'NewSecurePass123!',
]);
```

#### Domain Operations

```php
// Register domain
$results = localApi('RegisterDomain', [
    'client_id' => 123,
    'domain' => 'newdomain.com',
    'regperiod' => 2,
    'dnsmanagement' => true,
]);

// Transfer domain
$results = localApi('TransferDomain', [
    'client_id' => 123,
    'domain' => 'transfer.com',
    'transfersecret' => 'auth_code_here',
]);

// Renew domain
$results = localApi('RenewDomain', [
    'client_id' => 123,
    'domain' => 'example.com',
    'regperiod' => 1,
]);
```

---

## REST API Client

### Authentication Methods

```php
<?php
/**
 * WHMCS REST API v2 Client
 */
class WHMCSRestClient {
    private string $baseUrl;
    private string $apiIdentifier;
    private string $apiSecret;
    private int $timeout = 30;

    public function __construct(string $baseUrl, string $apiIdentifier, string $apiSecret) {
        $this->baseUrl = rtrim($baseUrl, '/');
        $this->apiIdentifier = $apiIdentifier;
        $this->apiSecret = $apiSecret;
    }

    public function request(string $method, string $endpoint, array $data = []): array {
        $ch = curl_init();

        curl_setopt_array($ch, [
            CURLOPT_URL => $this->baseUrl . '/api/v2/' . ltrim($endpoint, '/'),
            CURLOPT_RETURNTRANSFER => true,
            CURLOPT_TIMEOUT => $this->timeout,
            CURLOPT_SSL_VERIFYPEER => true,
            CURLOPT_SSL_VERIFYHOST => 2,
        ]);

        $headers = [
            'Authorization: Bearer ' . base64_encode($this->apiIdentifier . ':' . $this->apiSecret),
            'Content-Type: application/json',
            'Accept: application/json',
        ];

        if ($method !== 'GET') {
            curl_setopt($ch, CURLOPT_CUSTOMREQUEST, $method);
            curl_setopt($ch, CURLOPT_POSTFIELDS, json_encode($data));
        }

        curl_setopt($ch, CURLOPT_HTTPHEADER, $headers);

        $response = curl_exec($ch);
        $httpCode = curl_getinfo($ch, CURLINFO_HTTP_CODE);
        curl_close($ch);

        $decoded = json_decode($response, true) ?? [];

        if ($httpCode >= 400) {
            throw new \Exception($decoded['message'] ?? 'HTTP ' . $httpCode);
        }

        return $decoded;
    }

    public function get(string $endpoint): array {
        return $this->request('GET', $endpoint);
    }

    public function post(string $endpoint, array $data = []): array {
        return $this->request('POST', $endpoint, $data);
    }
}
```

### Usage Example

```php
$whmcs = new WHMCSRestClient(
    'https://whmcs.example.com',
    'your_api_identifier',
    'your_api_secret'
);

// Get clients
$clients = $whmcs->post('clients', [
    'limit' => 100,
    'sorting' => 'client_id',
]);

// Add client
$newClient = $whmcs->post('clients', [
    'first_name' => 'Jane',
    'last_name' => 'Smith',
    'email' => 'jane@example.com',
    'password' => 'SecurePass123',
    'country' => 'US',
]);

// Get client details
$client = $whmcs->get('clients/' . $clientId);
```

---

## API Response Handling

### Success Response

```php
$results = localApi('GetClientDetails', ['client_id' => 123]);

if ($results['result'] === 'success') {
    $client = $results;
    // Process client data
} else {
    // Handle error
    $error = $results['message'];
}
```

### Error Handling

```php
try {
    $results = localApi('AddClient', $params);

    if ($results['result'] !== 'success') {
        throw new \Exception($results['message']);
    }

    return $results['client_id'];

} catch (\Exception $e) {
    logActivity('API Error: ' . $e->getMessage());
    return null;
}
```

---

## API Response Codes

| Code | Result | Description |
|------|--------|-------------|
| 200 | success | Request successful |
| 400 | error | Invalid parameters or validation failed |
| 401 | error | Authentication failed |
| 403 | error | Permission denied |
| 404 | error | Resource not found |
| 429 | error | Rate limit exceeded |
| 500 | error | Internal server error |

---

## Rate Limiting

### Implementation

```php
class RateLimitedApiClient {
    private int $maxRequestsPerMinute = 60;
    private array $requestLog = [];

    public function makeRequest(array $postData): array {
        $now = time();

        // Clean old entries
        $this->requestLog = array_filter(
            $this->requestLog,
            fn($timestamp) => $timestamp > $now - 60
        );

        if (count($this->requestLog) >= $this->maxRequestsPerMinute) {
            $waitTime = 60 - ($now - end($this->requestLog));
            throw new \Exception("Rate limit exceeded. Wait {$waitTime} seconds.");
        }

        $this->requestLog[] = $now;
        return $this->executeRequest($postData);
    }

    private function executeRequest(array $postData): array {
        // Make API request
        $ch = curl_init();
        curl_setopt_array($ch, [
            CURLOPT_URL => WHMCS_API_URL,
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

---

## Webhook Integration

```php
add_hook('AfterModuleCreate', 1, function($vars) {
    // Send webhook notification
    $webhookPayload = [
        'event' => 'service_created',
        'service_id' => $vars['serviceid'],
        'user_id' => $vars['userid'],
        'timestamp' => date('c'),
    ];

    // Use WHMCS webhook system
    send_hook_notification('service_events', $webhookPayload);
});
```

---

## Best Practices

1. **Use API Credentials** - Never hardcode admin credentials
2. **Implement Retry Logic** - Handle temporary failures with exponential backoff
3. **Log All Requests** - Maintain audit trail for debugging
4. **Use Pagination** - Respect limit parameters for large datasets
5. **Validate Input** - Sanitize all user-supplied data
6. **Handle Webhooks** - Subscribe to events for real-time updates
7. **SSL Verification** - Always verify SSL certificates in production

---

## Related Documentation

- [API Endpoints Reference](api-endpoints-reference.md)
- [Webhook Events Reference](webhook-events-reference.md)
- [Hooks Reference](hooks-reference.md)
- [Email Template Variables](email-template-variables.md)
