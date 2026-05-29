# WHMCS Integration API

Complete guide for integrating with WHMCS via its API.

## Overview

WHMCS provides REST APIs for external integration with billing, support, and provisioning systems.

## WHMCS API Authentication

### API Authentication

```php
<?php
/**
 * WHMCS API authentication
 */
class WHMCSApiClient
{
    private string $baseUrl;
    private string $apiKey;
    
    public function __construct(string $baseUrl, string $apiKey)
    {
        $this->baseUrl = rtrim($baseUrl, '/');
        $this->apiKey = $apiKey;
    }
    
    /**
     * Make API call
     */
    public function call(string $action, array $params = []): array
    {
        $params['action'] = $action;
        $params['username'] = $this->apiKey;
        $params['password'] = ''; // API key auth
        $params['responsetype'] = 'json';
        
        $ch = curl_init();
        curl_setopt_array($ch, [
            CURLOPT_URL => $this->baseUrl . '/includes/api.php',
            CURLOPT_POST => true,
            CURLOPT_POSTFIELDS => http_build_query($params),
            CURLOPT_RETURNTRANSFER => true,
            CURLOPT_TIMEOUT => 30,
        ]);
        
        $response = curl_exec($ch);
        $httpCode = curl_getinfo($ch, CURLINFO_HTTP_CODE);
        curl_close($ch);
        
        return json_decode($response, true) ?? [];
    }
}
```

### API Key Authentication

```php
<?php
/**
 * Generate API credentials
 */
function createApiCredentials(): array
{
    // API credentials are generated in WHMCS Admin > Setup > Staff Management > API Credentials
    
    return [
        'api_url' => 'https://whmcs.example.com/includes/api.php',
        'api_key' => 'your_api_key_here',
        'secret' => 'your_api_secret_here',
    ];
}
```

## Client Management

### Create Client

```php
<?php
/**
 * Create new client via API
 */
function createClient(array $clientData): array
{
    $api = new WHMCSApiClient(WHMCS_URL, API_KEY);
    
    $params = [
        'firstname' => $clientData['firstname'],
        'lastname' => $clientData['lastname'],
        'email' => $clientData['email'],
        'companyname' => $clientData['companyname'] ?? '',
        'address1' => $clientData['address1'],
        'city' => $clientData['city'],
        'state' => $clientData['state'],
        'postcode' => $clientData['postcode'],
        'country' => $clientData['country'],
        'phonenumber' => $clientData['phonenumber'],
        'password2' => $clientData['password'],
        'clientip' => $_SERVER['REMOTE_ADDR'] ?? '',
    ];
    
    return $api->call('AddClient', $params);
}

/**
 * Example response
 */
$response = [
    'result' => 'success',
    'clientid' => 123,
    'password' => 'encrypted_password',
];
```

### Get Client Details

```php
<?php
/**
 * Get client details
 */
function getClientDetails(int $clientId): array
{
    $api = new WHMCSApiClient(WHMCS_URL, API_KEY);
    
    return $api->call('GetClientsDetails', [
        'clientid' => $clientId,
        'stats' => true,
    ]);
}

/**
 * Get clients by email
 */
function getClientByEmail(string $email): array
{
    $api = new WHMCSApiClient(WHMCS_URL, API_KEY);
    
    return $api->call('GetClientsDetails', [
        'email' => $email,
    ]);
}
```

### Update Client

```php
<?php
/**
 * Update client information
 */
function updateClient(int $clientId, array $data): array
{
    $api = new WHMCSApiClient(WHMCS_URL, API_KEY);
    
    $params = ['clientid' => $clientId];
    
    $allowedFields = [
        'firstname', 'lastname', 'email', 'companyname',
        'address1', 'address2', 'city', 'state',
        'postcode', 'country', 'phonenumber',
    ];
    
    foreach ($allowedFields as $field) {
        if (isset($data[$field])) {
            $params[$field] = $data[$field];
        }
    }
    
    return $api->call('UpdateClient', $params);
}
```

## Service Management

### Create Service/Product

```php
<?php
/**
 * Create new service
 */
function createService(int $clientId, int $productId, array $config = []): array
{
    $api = new WHMCSApiClient(WHMCS_URL, API_KEY);
    
    $params = [
        'clientid' => $clientId,
        'pid' => $productId,
        'domain' => $config['domain'],
        'billingcycle' => $config['billingcycle'] ?? 'monthly',
        'regperiod' => $config['regperiod'] ?? 1,
        'paymentmethod' => $config['paymentmethod'] ?? 'paypal',
        'customfields' => base64_encode(serialize($config['customfields'] ?? [])),
        'configoptions' => base64_encode(serialize($config['configoptions'] ?? [])),
    ];
    
    return $api->call('AddOrder', $params);
}
```

### Get Service Details

```php
<?php
/**
 * Get service/hosting details
 */
function getServiceDetails(int $serviceId): array
{
    $api = new WHMCSApiClient(WHMCS_URL, API_KEY);
    
    return $api->call('GetClientsProducts', [
        'serviceid' => $serviceId,
    ]);
}

/**
 * Get all client services
 */
function getClientServices(int $clientId): array
{
    $api = new WHMCSApiClient(WHMCS_URL, API_KEY);
    
    return $api->call('GetClientsProducts', [
        'clientid' => $clientId,
        'stats' => true,
    ]);
}
```

### Suspend Service

```php
<?php
/**
 * Suspend a service
 */
function suspendService(int $serviceId, string $reason = ''): array
{
    $api = new WHMCSApiClient(WHMCS_URL, API_KEY);
    
    return $api->call('ModuleSuspend', [
        'serviceid' => $serviceId,
        'suspendreason' => $reason,
    ]);
}

/**
 * Unsuspend service
 */
function unsuspendService(int $serviceId): array
{
    $api = new WHMCSApiClient(WHMCS_URL, API_KEY);
    
    return $api->call('ModuleUnsuspend', [
        'serviceid' => $serviceId,
    ]);
}

/**
 * Terminate service
 */
function terminateService(int $serviceId): array
{
    $api = new WHMCSApiClient(WHMCS_URL, API_KEY);
    
    return $api->call('ModuleTerminate', [
        'serviceid' => $serviceId,
    ]);
}
```

## Invoice Management

### Create Invoice

```php
<?php
/**
 * Create invoice
 */
function createInvoice(int $clientId, array $items): array
{
    $api = new WHMCSApiClient(WHMCS_URL, API_KEY);
    
    $params = [
        'userid' => $clientId,
        'sendinvoice' => true,
        'itemdescription1' => $items[0]['description'],
        'itemamount1' => $items[0]['amount'],
        'itemtaxed1' => $items[0]['taxed'] ?? false,
    ];
    
    // Add additional items
    for ($i = 1; $i < count($items); $i++) {
        $params["itemdescription" . ($i + 1)] = $items[$i]['description'];
        $params["itemamount" . ($i + 1)] = $items[$i]['amount'];
        $params["itemtaxed" . ($i + 1)] = $items[$i]['taxed'] ?? false;
    }
    
    return $api->call('CreateInvoice', $params);
}

/**
 * Get invoice details
 */
function getInvoice(int $invoiceId): array
{
    $api = new WHMCSApiClient(WHMCS_URL, API_KEY);
    
    return $api->call('GetInvoice', [
        'invoiceid' => $invoiceId,
    ]);
}
```

### Add Invoice Payment

```php
<?php
/**
 * Add payment to invoice
 */
function addInvoicePayment(int $invoiceId, string $transactionId, float $amount): array
{
    $api = new WHMCSApiClient(WHMCS_URL, API_KEY);
    
    return $api->call('AddInvoicePayment', [
        'invoiceid' => $invoiceId,
        'transid' => $transactionId,
        'amount' => $amount,
        'gateway' => 'custom',
    ]);
}
```

## Domain Management

### Register Domain

```php
<?php
/**
 * Register domain
 */
function registerDomain(array $domainData): array
{
    $api = new WHMCSApiClient(WHMCS_URL, API_KEY);
    
    $params = [
        'clientid' => $domainData['clientid'],
        'domain' => $domainData['domain'],
        'regperiod' => $domainData['regperiod'] ?? 1,
        'registrar' => $domainData['registrar'] ?? '',
        'paymentmethod' => $domainData['paymentmethod'] ?? 'paypal',
        'nameservers' => [
            'ns1' => $domainData['ns1'],
            'ns2' => $domainData['ns2'],
        ],
        ' registrant' => [
            'firstname' => $domainData['firstname'],
            'lastname' => $domainData['lastname'],
            'email' => $domainData['email'],
            'companyname' => $domainData['companyname'] ?? '',
            'address1' => $domainData['address1'],
            'city' => $domainData['city'],
            'state' => $domainData['state'],
            'postcode' => $domainData['postcode'],
            'country' => $domainData['country'],
            'phonenumber' => $domainData['phonenumber'],
        ],
    ];
    
    return $api->call('RegisterDomain', $params);
}
```

### Transfer Domain

```php
<?php
/**
 * Transfer domain
 */
function transferDomain(array $transferData): array
{
    $api = new WHMCSApiClient(WHMCS_URL, API_KEY);
    
    $params = [
        'clientid' => $transferData['clientid'],
        'domain' => $transferData['domain'],
        'transfersecret' => $transferData['epp_code'],
        'regperiod' => $transferData['regperiod'] ?? 1,
        'registrar' => $transferData['registrar'] ?? '',
        'paymentmethod' => $transferData['paymentmethod'] ?? 'paypal',
    ];
    
    return $api->call('TransferDomain', $params);
}
```

## Ticket Management

```php
<?php
/**
 * Open support ticket
 */
function openTicket(array $ticketData): array
{
    $api = new WHMCSApiClient(WHMCS_URL, API_KEY);
    
    $params = [
        'clientid' => $ticketData['clientid'],
        'subject' => $ticketData['subject'],
        'message' => $ticketData['message'],
        'priority' => $ticketData['priority'] ?? 'Medium', // Low, Medium, High
        'deptid' => $ticketData['department_id'],
    ];
    
    return $api->call('OpenTicket', $params);
}

/**
 * Reply to ticket
 */
function replyToTicket(int $ticketId, string $message): array
{
    $api = new WHMCSApiClient(WHMCS_URL, API_KEY);
    
    return $api->call('AddTicketReply', [
        'ticketid' => $ticketId,
        'message' => $message,
    ]);
}
```

## Webhooks

### Configuring Webhooks

```php
<?php
/**
 * WHMCS Webhook handler endpoint
 */
function handleWebhook(array $payload, string $secret): array
{
    // Verify webhook signature
    $signature = $_SERVER['HTTP_X_WHMCS_SIGNATURE'] ?? '';
    
    if (!verifyWebhookSignature($payload, $signature, $secret)) {
        return ['error' => 'Invalid signature'];
    }
    
    $eventType = $payload['event_type'] ?? '';
    
    switch ($eventType) {
        case 'InvoiceCreated':
            return handleInvoiceCreated($payload);
        case 'InvoicePaid':
            return handleInvoicePaid($payload);
        case 'ServiceCreated':
            return handleServiceCreated($payload);
        case 'DomainRenewed':
            return handleDomainRenewed($payload);
        default:
            return ['status' => 'ignored'];
    }
}

/**
 * Verify webhook signature
 */
function verifyWebhookSignature(array $payload, string $signature, string $secret): bool
{
    $expectedSignature = hash_hmac('sha256', json_encode($payload), $secret);
    return hash_equals($expectedSignature, $signature);
}
```

## Rate Limiting

```php
<?php
/**
 * Implement rate limiting for API calls
 */
class RateLimitedApiClient extends WHMCSApiClient
{
    private int $requestsPerMinute;
    private array $requestLog = [];
    
    public function __construct(string $baseUrl, string $apiKey, int $rpm = 60)
    {
        parent::__construct($baseUrl, $apiKey);
        $this->requestsPerMinute = $rpm;
    }
    
    public function call(string $action, array $params = []): array
    {
        $this->checkRateLimit();
        
        $result = parent::call($action, $params);
        
        $this->requestLog[] = time();
        
        return $result;
    }
    
    private function checkRateLimit(): void
    {
        $now = time();
        $this->requestLog = array_filter(
            $this->requestLog,
            fn($timestamp) => $timestamp > ($now - 60)
        );
        
        if (count($this->requestLog) >= $this->requestsPerMinute) {
            $oldestRequest = min($this->requestLog);
            $sleepTime = 60 - ($now - $oldestRequest);
            
            if ($sleepTime > 0) {
                sleep($sleepTime);
            }
        }
    }
}
```

## Best Practices

1. **Use HTTPS** - Always use SSL for API calls
2. **Store credentials securely** - Never commit API keys to version control
3. **Implement retry logic** - Handle transient failures
4. **Log all calls** - Keep audit trail of API operations
5. **Handle rate limits** - Respect API rate limits
6. **Validate responses** - Check API response structure

## Related Documentation

- [whmcs-integration-webhooks.md](whmcs-integration-webhooks.md)
- [whmcs-integration-oauth.md](whmcs-integration-oauth.md)
