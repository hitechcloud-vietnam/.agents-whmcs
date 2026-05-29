# WHMCS API Integration Guide

## Overview
This workflow covers integrating external services with WHMCS using the REST API.

## Step 1: WHMCS API Client

```php
<?php
// src/Service/WhmcsApiClient.php

namespace WHMCS\Module\Addon\YourModule\Service;

class WhmcsApiClient
{
    private $baseUrl;
    private $apiUsername;
    private $apiPassword;
    private $apiIdentifier;
    private $apiSecret;

    public function __construct(string $baseUrl, string $apiUsername, string $apiPassword)
    {
        $this->baseUrl = rtrim($baseUrl, '/');
        $this->apiUsername = $apiUsername;
        $this->apiPassword = $apiPassword;
    }

    public function call(string $action, array $parameters = []): array
    {
        $parameters['username'] = $this->apiUsername;
        $parameters['password'] = $this->apiPassword;
        $parameters['action'] = $action;
        $parameters['responsetype'] = 'json';

        $ch = curl_init($this->baseUrl . '/api.php');
        curl_setopt_array($ch, [
            CURLOPT_POST => true,
            CURLOPT_POSTFIELDS => http_build_query($parameters),
            CURLOPT_RETURNTRANSFER => true,
            CURLOPT_TIMEOUT => 30,
            CURLOPT_SSL_VERIFYPEER => true,
            CURLOPT_HTTPHEADER => ['Content-Type: application/x-www-form-urlencoded']
        ]);

        $response = curl_exec($ch);
        $httpCode = curl_getinfo($ch, CURLINFO_HTTP_CODE);
        $error = curl_error($ch);
        curl_close($ch);

        if ($error) {
            throw new \Exception("API Error: " . $error);
        }

        if ($httpCode !== 200) {
            throw new \Exception("API HTTP Error: " . $httpCode);
        }

        $result = json_decode($response, true);

        if (isset($result['result']) && $result['result'] === 'error') {
            throw new \Exception("API Error: " . ($result['message'] ?? 'Unknown error'));
        }

        return $result;
    }

    // Client Operations
    public function getClients(array $filters = []): array
    {
        return $this->call('GetClients', $filters);
    }

    public function getClient(int $clientId): array
    {
        return $this->call('GetClientsDetails', ['clientid' => $clientId]);
    }

    public function createClient(array $data): array
    {
        return $this->call('AddClient', $data);
    }

    public function updateClient(int $clientId, array $data): array
    {
        $data['clientid'] = $clientId;
        return $this->call('UpdateClient', $data);
    }

    public function deleteClient(int $clientId): array
    {
        return $this->call('DeleteClient', ['clientid' => $clientId]);
    }

    // Service Operations
    public function getServices(int $clientId): array
    {
        return $this->call('GetClientsProducts', ['clientid' => $clientId]);
    }

    public function createService(array $data): array
    {
        return $this->call('AddOrder', $data);
    }

    public function suspendService(int $serviceId): array
    {
        return $this->call('SuspendOrder', ['serviceid' => $serviceId]);
    }

    public function unsuspendService(int $serviceId): array
    {
        return $this->call('UnsuspendOrder', ['serviceid' => $serviceId]);
    }

    public function terminateService(int $serviceId): array
    {
        return $this->call('TerminateOrder', ['serviceid' => $serviceId]);
    }

    // Invoice Operations
    public function getInvoices(array $filters = []): array
    {
        return $this->call('GetInvoices', $filters);
    }

    public function createInvoice(array $data): array
    {
        return $this->call('CreateInvoice', $data);
    }

    public function addInvoicePayment(int $invoiceId, float $amount, string $transactionId): array
    {
        return $this->call('AddInvoicePayment', [
            'invoiceid' => $invoiceId,
            'amount' => $amount,
            'transactionid' => $transactionId
        ]);
    }

    // Domain Operations
    public function getDomains(int $clientId): array
    {
        return $this->call('GetClientsDomains', ['clientid' => $clientId]);
    }

    public function registerDomain(array $data): array
    {
        return $this->call('RegisterDomain', $data);
    }

    public function transferDomain(array $data): array
    {
        return $this->call('TransferDomain', $data);
    }

    // Ticket Operations
    public function createTicket(array $data): array
    {
        return $this->call('OpenTicket', $data);
    }

    public function getTickets(array $filters = []): array
    {
        return $this->call('GetTickets', $filters);
    }

    public function addTicketReply(int $ticketId, string $message): array
    {
        return $this->call('AddTicketReply', [
            'ticketid' => $ticketId,
            'message' => $message
        ]);
    }
}
```

## Step 2: External API Integration

```php
<?php
// src/Service/ExternalApiService.php

namespace WHMCS\Module\Addon\YourModule\Service;

class ExternalApiService
{
    private $baseUrl;
    private $apiKey;
    private $timeout = 30;

    public function __construct(string $baseUrl, string $apiKey)
    {
        $this->baseUrl = rtrim($baseUrl, '/');
        $this->apiKey = $apiKey;
    }

    public function request(string $method, string $endpoint, array $data = []): array
    {
        $url = $this->baseUrl . '/' . ltrim($endpoint, '/');
        $headers = [
            'Authorization: Bearer ' . $this->apiKey,
            'Content-Type: application/json',
            'Accept: application/json'
        ];

        $ch = curl_init();
        curl_setopt_array($ch, [
            CURLOPT_URL => $url,
            CURLOPT_RETURNTRANSFER => true,
            CURLOPT_TIMEOUT => $this->timeout,
            CURLOPT_HTTPHEADER => $headers
        ]);

        if (in_array($method, ['POST', 'PUT', 'PATCH'])) {
            curl_setopt($ch, CURLOPT_POST, true);
            curl_setopt($ch, CURLOPT_POSTFIELDS, json_encode($data));
        }

        if ($method === 'DELETE') {
            curl_setopt($ch, CURLOPT_CUSTOMREQUEST, 'DELETE');
        }

        $response = curl_exec($ch);
        $httpCode = curl_getinfo($ch, CURLINFO_HTTP_CODE);
        $error = curl_error($ch);
        curl_close($ch);

        if ($error) {
            throw new \Exception("Request failed: " . $error);
        }

        $decoded = json_decode($response, true);

        if ($httpCode >= 400) {
            $message = $decoded['error']['message'] ?? 'Request failed';
            throw new \Exception("API Error ($httpCode): " . $message);
        }

        return $decoded;
    }

    public function get(string $endpoint, array $params = []): array
    {
        $query = http_build_query($params);
        $url = $query ? "$endpoint?$query" : $endpoint;
        return $this->request('GET', $url);
    }

    public function post(string $endpoint, array $data): array
    {
        return $this->request('POST', $endpoint, $data);
    }

    public function put(string $endpoint, array $data): array
    {
        return $this->request('PUT', $endpoint, $data);
    }

    public function delete(string $endpoint): array
    {
        return $this->request('DELETE', $endpoint);
    }
}
```

## Step 3: Sync Service

```php
<?php
// src/Service/SyncService.php

namespace WHMCS\Module\Addon\YourModule\Service;

use WHMCS\Database\Capsule;

class SyncService
{
    private $whmcsApi;
    private $externalApi;

    public function __construct(WhmcsApiClient $whmcsApi, ExternalApiService $externalApi)
    {
        $this->whmcsApi = $whmcsApi;
        $this->externalApi = $externalApi;
    }

    public function syncClients(): array
    {
        $results = ['created' => 0, 'updated' => 0, 'errors' => 0];

        try {
            // Get clients from external system
            $externalClients = $this->externalApi->get('/clients');

            foreach ($externalClients as $extClient) {
                try {
                    $this->syncClient($extClient, $results);
                } catch (\Exception $e) {
                    $results['errors']++;
                    logActivity("Client sync error: " . $e->getMessage());
                }
            }
        } catch (\Exception $e) {
            logActivity("Sync failed: " . $e->getMessage());
        }

        return $results;
    }

    private function syncClient(array $externalClient, array &$results): void
    {
        // Check if client exists in WHMCS
        $existing = Capsule::table('tblclients')
            ->where('email', $externalClient['email'])
            ->first();

        if ($existing) {
            // Update existing client
            $this->whmcsApi->updateClient($existing->id, [
                'firstname' => $externalClient['first_name'],
                'lastname' => $externalClient['last_name'],
                'companyname' => $externalClient['company'] ?? '',
                'phonenumber' => $externalClient['phone'] ?? ''
            ]);
            $results['updated']++;
        } else {
            // Create new client
            $this->whmcsApi->createClient([
                'firstname' => $externalClient['first_name'],
                'lastname' => $externalClient['last_name'],
                'email' => $externalClient['email'],
                'password2' => $this->generateRandomPassword(),
                'country' => $externalClient['country'] ?? 'US',
                'phonenumber' => $externalClient['phone'] ?? ''
            ]);
            $results['created']++;
        }
    }

    private function generateRandomPassword(int $length = 12): string
    {
        $chars = 'abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ0123456789';
        $password = '';
        for ($i = 0; $i < $length; $i++) {
            $password .= $chars[random_int(0, strlen($chars) - 1)];
        }
        return $password;
    }

    public function syncServices(): array
    {
        $results = ['synced' => 0, 'errors' => 0];

        try {
            $externalServices = $this->externalApi->get('/services');

            foreach ($externalServices as $extService) {
                try {
                    $this->syncService($extService);
                    $results['synced']++;
                } catch (\Exception $e) {
                    $results['errors']++;
                }
            }
        } catch (\Exception $e) {
            logActivity("Service sync failed: " . $e->getMessage());
        }

        return $results;
    }

    private function syncService(array $externalService): void
    {
        // Find client
        $client = Capsule::table('tblclients')
            ->where('email', $externalService['client_email'])
            ->first();

        if (!$client) {
            throw new \Exception("Client not found for service");
        }

        // Create order
        $this->whmcsApi->createService([
            'client_id' => $client->id,
            'pid' => $externalService['product_id'],
            'billingcycle' => $externalService['billing_cycle'] ?? 'monthly',
            'domaintype' => 'register',
            'domain' => $externalService['domain']
        ]);
    }
}
```

## Step 4: Sync Cron Job

```php
<?php
// includes/cron/sync_cron.php

require_once __DIR__ . '/../../init.php';

use WHMCS\Module\Addon\YourModule\Service\WhmcsApiClient;
use WHMCS\Module\Addon\YourModule\Service\ExternalApiService;
use WHMCS\Module\Addon\YourModule\Service\SyncService;

echo "=== External Sync Cron ===\n";

$whmcsApi = new WhmcsApiClient(
    Capsule::config('system_url'),
    get_config('api_username'),
    get_config('api_password')
);

$externalApi = new ExternalApiService(
    get_config('external_api_url'),
    get_config('external_api_key')
);

$syncService = new SyncService($whmcsApi, $externalApi);

echo "Syncing clients...\n";
$clientResults = $syncService->syncClients();
echo "Created: {$clientResults['created']}, Updated: {$clientResults['updated']}, Errors: {$clientResults['errors']}\n";

echo "Syncing services...\n";
$serviceResults = $syncService->syncServices();
echo "Synced: {$serviceResults['synced']}, Errors: {$serviceResults['errors']}\n";

echo "=== Sync Complete ===\n";
```

## Verification Checklist

- [ ] WHMCS API client implemented
- [ ] External API client implemented
- [ ] Client sync working
- [ ] Service sync working
- [ ] Error handling implemented
- [ ] Sync cron configured
- [ ] Test sync completed successfully
