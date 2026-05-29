# WHMCS Registrar Master

## Overview
Master skill for WHMCS domain registrar module development. Covers domain registration, transfer, renewal, WHOIS management, and DNS configuration.

## Registrar Module Structure

```php
<?php
// /modules/registrars/YourRegistrar/YourRegistrar.php

if (!defined("WHMCS")) {
    die("This file cannot be accessed directly");
}

use WHMCS\Domains\Domain;
use WHMCS\Exception\Registrar\GeneralException;

/**
 * Registrar Module Functions
 */

function YourRegistrar_getConfigArray()
{
    return [
        'FriendlyName' => [
            'Type' => 'System',
            'Value' => 'Your Registrar Name',
        ],
        'APIUsername' => [
            'FriendlyName' => 'API Username',
            'Type' => 'text',
            'Size' => '40',
            'Description' => 'Your registrar API username',
        ],
        'APIPassword' => [
            'FriendlyName' => 'API Password',
            'Type' => 'password',
            'Size' => '40',
            'Description' => 'Your registrar API password',
        ],
        'APIKey' => [
            'FriendlyName' => 'API Key',
            'Type' => 'password',
            'Size' => '50',
            'Description' => 'API key for authentication',
        ],
        'AccountNumber' => [
            'FriendlyName' => 'Account Number',
            'Type' => 'text',
            'Size' => '20',
        ],
        'ResellerID' => [
            'FriendlyName' => 'Reseller ID',
            'Type' => 'text',
            'Size' => '20',
        ],
        'TestMode' => [
            'FriendlyName' => 'Test Mode',
            'Type' => 'yesno',
            'Description' => 'Enable sandbox environment',
        ],
        'DebugMode' => [
            'FriendlyName' => 'Debug Mode',
            'Type' => 'yesno',
            'Description' => 'Log all API requests',
        ],
        'DefaultNameservers' => [
            'FriendlyName' => 'Default Nameservers',
            'Type' => 'yesno',
            'Description' => 'Use default registrar nameservers if none specified',
        ],
        'DNSSECEnabled' => [
            'FriendlyName' => 'DNSSEC Support',
            'Type' => 'yesno',
            'Description' => 'Enable DNSSEC management',
        ],
    ];
}

function YourRegistrar_registerDomain(array $params)
{
    try {
        $api = new RegistrarAPI($params);

        $result = $api->registerDomain([
            'domain' => $params['domain'],
            'period' => $params['regperiod'],
            'registrant' => [
                'first_name' => $params['firstname'],
                'last_name' => $params['lastname'],
                'organization' => $params['companyname'] ?: null,
                'email' => $params['email'],
                'address1' => $params['address1'],
                'address2' => $params['address2'] ?: null,
                'city' => $params['city'],
                'state' => $params['state'],
                'postcode' => $params['postcode'],
                'country' => $params['country'],
                'phone' => $params['phonenumber'],
                'phone_cc' => $params['phonecc'] ?? null,
            ],
            'ns1' => $params['ns1'],
            'ns2' => $params['ns2'],
            'ns3' => $params['ns3'] ?? null,
            'ns4' => $params['ns4'] ?? null,
            'protect' => $params['dnssec'] ?? false,
        ]);

        if ($result['success']) {
            return ['success' => true];
        }

        return [
            'success' => false,
            'error' => $result['error'] ?? 'Registration failed',
        ];
    } catch (\Exception $e) {
        throw new GeneralException('Registration error: ' . $e->getMessage());
    }
}

function YourRegistrar_transferDomain(array $params)
{
    try {
        $api = new RegistrarAPI($params);

        $result = $api->transferDomain([
            'domain' => $params['domain'],
            'auth_code' => $params['transfersecret'],
            'period' => $params['regperiod'],
            'registrant' => [
                'first_name' => $params['firstname'],
                'last_name' => $params['lastname'],
                'organization' => $params['companyname'] ?: null,
                'email' => $params['email'],
                'address1' => $params['address1'],
                'city' => $params['city'],
                'state' => $params['state'],
                'postcode' => $params['postcode'],
                'country' => $params['country'],
                'phone' => $params['phonenumber'],
            ],
        ]);

        if ($result['success']) {
            return ['success' => true];
        }

        return [
            'success' => false,
            'error' => $result['error'] ?? 'Transfer failed',
        ];
    } catch (\Exception $e) {
        throw new GeneralException('Transfer error: ' . $e->getMessage());
    }
}

function YourRegistrar_renewDomain(array $params)
{
    try {
        $api = new RegistrarAPI($params);

        $result = $api->renewDomain([
            'domain' => $params['domain'],
            'period' => $params['regperiod'],
            'current_expiry' => $params['expirydate'],
        ]);

        if ($result['success']) {
            return ['success' => true];
        }

        return [
            'success' => false,
            'error' => $result['error'] ?? 'Renewal failed',
        ];
    } catch (\Exception $e) {
        throw new GeneralException('Renewal error: ' . $e->getMessage());
    }
}

function YourRegistrar_Sync(array $params)
{
    try {
        $api = new RegistrarAPI($params);

        $result = $api->getDomainInfo([
            'domain' => $params['domain'],
        ]);

        if ($result['success']) {
            return [
                'success' => true,
                'expirydate' => $result['expiry_date'],
                'active' => $result['status'] === 'active',
                'expired' => $result['status'] === 'expired',
                'pending' => $result['status'] === 'pending',
                'transferredAway' => $result['transferred'],
            ];
        }

        return [
            'success' => false,
            'error' => $result['error'],
        ];
    } catch (\Exception $e) {
        return [
            'success' => false,
            'error' => $e->getMessage(),
        ];
    }
}

function YourRegistrar_TransferSync(array $params)
{
    try {
        $api = new RegistrarAPI($params);

        $result = $api->getTransferStatus([
            'domain' => $params['domain'],
        ]);

        return [
            'success' => true,
            'status' => $result['status'], // completed, pending, rejected, cancelled
            'message' => $result['message'] ?? null,
        ];
    } catch (\Exception $e) {
        return [
            'success' => false,
            'error' => $e->getMessage(),
        ];
    }
}

function YourRegistrar_GetNameservers(array $params)
{
    try {
        $api = new RegistrarAPI($params);

        $result = $api->getNameservers([
            'domain' => $params['domain'],
        ]);

        if ($result['success']) {
            return [
                'success' => true,
                'ns1' => $result['ns1'],
                'ns2' => $result['ns2'],
                'ns3' => $result['ns3'],
                'ns4' => $result['ns4'],
            ];
        }

        return [
            'success' => false,
            'error' => $result['error'],
        ];
    } catch (\Exception $e) {
        throw new GeneralException('Get nameservers error: ' . $e->getMessage());
    }
}

function YourRegistrar_SaveNameservers(array $params)
{
    try {
        $api = new RegistrarAPI($params);

        $result = $api->setNameservers([
            'domain' => $params['domain'],
            'ns1' => $params['ns1'],
            'ns2' => $params['ns2'],
            'ns3' => $params['ns3'] ?? null,
            'ns4' => $params['ns4'] ?? null,
        ]);

        if ($result['success']) {
            return ['success' => true];
        }

        return [
            'success' => false,
            'error' => $result['error'],
        ];
    } catch (\Exception $e) {
        throw new GeneralException('Save nameservers error: ' . $e->getMessage());
    }
}

function YourRegistrar_GetDNS(array $params)
{
    try {
        $api = new RegistrarAPI($params);

        $result = $api->getDnsRecords([
            'domain' => $params['domain'],
        ]);

        if ($result['success']) {
            $records = [];
            foreach ($result['records'] as $record) {
                $records[] = [
                    'hostname' => $record['name'],
                    'type' => $record['type'],
                    'priority' => $record['priority'] ?? 0,
                    'value' => $record['value'],
                    'ttl' => $record['ttl'] ?? 3600,
                ];
            }

            return [
                'success' => true,
                'records' => $records,
            ];
        }

        return [
            'success' => false,
            'error' => $result['error'],
        ];
    } catch (\Exception $e) {
        throw new GeneralException('Get DNS error: ' . $e->getMessage());
    }
}

function YourRegistrar_SaveDNS(array $params)
{
    try {
        $api = new RegistrarAPI($params);

        $records = [];
        foreach ($params['dnsrecords'] as $record) {
            $records[] = [
                'name' => $record['hostname'],
                'type' => $record['type'],
                'priority' => $record['priority'] ?? 0,
                'value' => $record['value'],
                'ttl' => $record['ttl'] ?? 3600,
            ];
        }

        $result = $api->setDnsRecords([
            'domain' => $params['domain'],
            'records' => $records,
        ]);

        if ($result['success']) {
            return ['success' => true];
        }

        return [
            'success' => false,
            'error' => $result['error'],
        ];
    } catch (\Exception $e) {
        throw new GeneralException('Save DNS error: ' . $e->getMessage());
    }
}

function YourRegistrar_GetContactDetails(array $params)
{
    try {
        $api = new RegistrarAPI($params);

        $result = $api->getContacts([
            'domain' => $params['domain'],
        ]);

        if ($result['success']) {
            return [
                'success' => true,
                'Registrant' => $result['registrant'],
                'Admin' => $result['admin'],
                'Tech' => $result['tech'],
                'Billing' => $result['billing'],
            ];
        }

        return [
            'success' => false,
            'error' => $result['error'],
        ];
    } catch (\Exception $e) {
        throw new GeneralException('Get contacts error: ' . $e->getMessage());
    }
}

function YourRegistrar_SaveContactDetails(array $params)
{
    try {
        $api = new RegistrarAPI($params);

        $result = $api->setContacts([
            'domain' => $params['domain'],
            'Registrant' => $params['contactdetails']['Registrant'],
            'Admin' => $params['contactdetails']['Admin'],
            'Tech' => $params['contactdetails']['Tech'],
            'Billing' => $params['contactdetails']['Billing'],
        ]);

        if ($result['success']) {
            return ['success' => true];
        }

        return [
            'success' => false,
            'error' => $result['error'],
        ];
    } catch (\Exception $e) {
        throw new GeneralException('Save contacts error: ' . $e->getMessage());
    }
}

function YourRegistrar_RegistrarLock(array $params)
{
    try {
        $api = new RegistrarAPI($params);

        $result = $api->setDomainLock([
            'domain' => $params['domain'],
            'lock' => true,
        ]);

        if ($result['success']) {
            return ['success' => true];
        }

        return [
            'success' => false,
            'error' => $result['error'],
        ];
    } catch (\Exception $e) {
        throw new GeneralException('Registrar lock error: ' . $e->getMessage());
    }
}

function YourRegistrar_RegistrarUnlock(array $params)
{
    try {
        $api = new RegistrarAPI($params);

        $result = $api->setDomainLock([
            'domain' => $params['domain'],
            'lock' => false,
        ]);

        if ($result['success']) {
            return ['success' => true];
        }

        return [
            'success' => false,
            'error' => $result['error'],
        ];
    } catch (\Exception $e) {
        throw new GeneralException('Registrar unlock error: ' . $e->getMessage());
    }
}

function YourRegistrar_IsOverRide(array $params)
{
    // Return true to bypass transfer lock for this registrar
    return false;
}

function YourRegistrar_GetEPPCode(array $params)
{
    try {
        $api = new RegistrarAPI($params);

        $result = $api->getAuthCode([
            'domain' => $params['domain'],
        ]);

        if ($result['success']) {
            return [
                'success' => true,
                'eppcode' => $result['auth_code'],
            ];
        }

        return [
            'success' => false,
            'error' => $result['error'],
        ];
    } catch (\Exception $e) {
        throw new GeneralException('Get EPP code error: ' . $e->getMessage());
    }
}

function YourRegistrar_RequestDelete(array $params)
{
    try {
        $api = new RegistrarAPI($params);

        $result = $api->requestDelete([
            'domain' => $params['domain'],
        ]);

        if ($result['success']) {
            return ['success' => true];
        }

        return [
            'success' => false,
            'error' => $result['error'],
        ];
    } catch (\Exception $e) {
        throw new GeneralException('Request delete error: ' . $e->getMessage());
    }
}

function YourRegistrar_GetDNSZoneRecords(array $params)
{
    // For DNS zone management
    return YourRegistrar_GetDNS($params);
}

function YourRegistrar_SaveDNSZoneRecords(array $params)
{
    // For DNS zone management
    return YourRegistrar_SaveDNS($params);
}

function YourRegistrar_EnableIDProtection(array $params)
{
    try {
        $api = new RegistrarAPI($params);

        $result = $api->setPrivacy([
            'domain' => $params['domain'],
            'enabled' => true,
        ]);

        if ($result['success']) {
            return ['success' => true];
        }

        return [
            'success' => false,
            'error' => $result['error'],
        ];
    } catch (\Exception $e) {
        throw new GeneralException('Enable ID protection error: ' . $e->getMessage());
    }
}

function YourRegistrar_DisableIDProtection(array $params)
{
    try {
        $api = new RegistrarAPI($params);

        $result = $api->setPrivacy([
            'domain' => $params['domain'],
            'enabled' => false,
        ]);

        if ($result['success']) {
            return ['success' => true];
        }

        return [
            'success' => false,
            'error' => $result['error'],
        ];
    } catch (\Exception $e) {
        throw new GeneralException('Disable ID protection error: ' . $e->getMessage());
    }
}

function YourRegistrar_GetDomainSuggestions(array $params)
{
    try {
        $api = new RegistrarAPI($params);

        $result = $api->getSuggestions([
            'domain' => $params['searchTerm'],
            'tlds' => $params['tlds'] ?? [],
        ]);

        if ($result['success']) {
            $suggestions = [];
            foreach ($result['suggestions'] as $suggestion) {
                $suggestions[] = [
                    'domain' => $suggestion['domain'],
                    'price' => $suggestion['price'],
                    'period' => $suggestion['period'] ?? 1,
                    'available' => $suggestion['available'],
                ];
            }

            return [
                'success' => true,
                'suggestions' => $suggestions,
            ];
        }

        return [
            'success' => false,
            'error' => $result['error'],
        ];
    } catch (\Exception $e) {
        return [
            'success' => false,
            'error' => $e->getMessage(),
        ];
    }
}

function YourRegistrar_DomainTransferCheck(array $params)
{
    try {
        $api = new RegistrarAPI($params);

        $result = $api->checkTransferEligibility([
            'domain' => $params['domain'],
            'auth_code' => $params['transfersecret'],
        ]);

        return [
            'success' => true,
            'transferable' => $result['eligible'],
            'reason' => $result['reason'] ?? null,
        ];
    } catch (\Exception $e) {
        return [
            'success' => false,
            'error' => $e->getMessage(),
        ];
    }
}
```

## API Client Class

```php
<?php
// /modules/registrars/YourRegistrar/lib/ApiClient.php

namespace WHMCS\Registrar\YourRegistrar;

class ApiClient
{
    private $username;
    private $password;
    private $apiKey;
    private $testMode;
    private $debugMode;

    const API_URL = 'https://api.registrar.com/v1';
    const SANDBOX_URL = 'https://sandbox.registrar.com/v1';

    public function __construct(array $params)
    {
        $this->username = $params['APIUsername'] ?? '';
        $this->password = $params['APIPassword'] ?? '';
        $this->apiKey = $params['APIKey'] ?? '';
        $this->testMode = $params['TestMode'] ?? false;
        $this->debugMode = $params['DebugMode'] ?? false;
    }

    private function getBaseUrl(): string
    {
        return $this->testMode ? self::SANDBOX_URL : self::API_URL;
    }

    public function request(string $method, string $endpoint, array $data = []): array
    {
        $url = $this->getBaseUrl() . $endpoint;
        $timestamp = gmdate('Y-m-d\TH:i:s\Z');

        $headers = [
            'Content-Type: application/json',
            'Accept: application/json',
            'X-Api-Key: ' . $this->apiKey,
            'X-Timestamp: ' . $timestamp,
        ];

        $payload = $method === 'GET' ? '' : json_encode($data);
        $signature = $this->generateSignature($method, $endpoint, $payload, $timestamp);
        $headers[] = 'X-Signature: ' . $signature;

        $ch = curl_init();
        curl_setopt_array($ch, [
            CURLOPT_URL => $url,
            CURLOPT_RETURNTRANSFER => true,
            CURLOPT_TIMEOUT => 60,
            CURLOPT_HTTPHEADER => $headers,
            CURLOPT_SSL_VERIFYPEER => true,
            CURLOPT_SSL_VERIFYHOST => 2,
        ]);

        if ($method === 'POST') {
            curl_setopt($ch, CURLOPT_POST, true);
            curl_setopt($ch, CURLOPT_POSTFIELDS, $payload);
        }

        if ($this->debugMode) {
            logActivity("[YourRegistrar] API Request: " . $method . " " . $url);
            logActivity("[YourRegistrar] Payload: " . $payload);
        }

        $response = curl_exec($ch);
        $httpCode = curl_getinfo($ch, CURLINFO_HTTP_CODE);
        $error = curl_error($ch);
        curl_close($ch);

        if ($error) {
            throw new \Exception('cURL Error: ' . $error);
        }

        $result = json_decode($response, true);

        if ($this->debugMode) {
            logActivity("[YourRegistrar] API Response: " . json_encode($result));
        }

        if ($httpCode >= 400) {
            throw new \Exception(
                $result['message'] ?? 'API Error: HTTP ' . $httpCode
            );
        }

        return $result;
    }

    private function generateSignature(string $method, string $endpoint, string $payload, string $timestamp): string
    {
        $string = $method . "\n" . $endpoint . "\n" . $payload . "\n" . $timestamp;
        return base64_encode(hash_hmac('sha256', $string, $this->apiKey, true));
    }

    public function registerDomain(array $params): array
    {
        return $this->request('POST', '/domains/register', [
            'domain' => $params['domain'],
            'registration_period' => $params['period'],
            ' registrant' => $params['registrant'],
            'nameservers' => [
                ['host' => $params['ns1']],
                ['host' => $params['ns2']],
                ['host' => $params['ns3'] ?? null],
                ['host' => $params['ns4'] ?? null],
            ],
        ]);
    }

    public function transferDomain(array $params): array
    {
        return $this->request('POST', '/domains/transfer', [
            'domain' => $params['domain'],
            'auth_code' => $params['auth_code'],
            'registration_period' => $params['period'],
            'registrant' => $params['registrant'],
        ]);
    }

    public function renewDomain(array $params): array
    {
        return $this->request('POST', '/domains/' . $params['domain'] . '/renew', [
            'period' => $params['period'],
            'current_expiry' => $params['current_expiry'],
        ]);
    }

    public function getDomainInfo(array $params): array
    {
        return $this->request('GET', '/domains/' . $params['domain']);
    }

    public function getTransferStatus(array $params): array
    {
        return $this->request('GET', '/domains/' . $params['domain'] . '/transfer');
    }

    public function getNameservers(array $params): array
    {
        return $this->request('GET', '/domains/' . $params['domain'] . '/nameservers');
    }

    public function setNameservers(array $params): array
    {
        return $this->request('PUT', '/domains/' . $params['domain'] . '/nameservers', [
            'nameservers' => [
                ['host' => $params['ns1']],
                ['host' => $params['ns2']],
                ['host' => $params['ns3'] ?? null],
                ['host' => $params['ns4'] ?? null],
            ],
        ]);
    }

    public function getContacts(array $params): array
    {
        return $this->request('GET', '/domains/' . $params['domain'] . '/contacts');
    }

    public function setContacts(array $params): array
    {
        return $this->request('PUT', '/domains/' . $params['domain'] . '/contacts', [
            'registrant' => $params['Registrant'],
            'admin' => $params['Admin'],
            'tech' => $params['Tech'],
            'billing' => $params['Billing'],
        ]);
    }

    public function setDomainLock(array $params): array
    {
        return $this->request('PUT', '/domains/' . $params['domain'] . '/lock', [
            'locked' => $params['lock'],
        ]);
    }

    public function getAuthCode(array $params): array
    {
        return $this->request('POST', '/domains/' . $params['domain'] . '/auth-code');
    }

    public function setPrivacy(array $params): array
    {
        return $this->request('PUT', '/domains/' . $params['domain'] . '/privacy', [
            'enabled' => $params['enabled'],
        ]);
    }

    public function getSuggestions(array $params): array
    {
        return $this->request('GET', '/domains/suggest', [
            'name' => $params['domain'],
            'tlds' => $params['tlds'],
        ]);
    }
}
```

## Best Practices

1. **API Security**: Use API keys and signature verification for all requests
2. **Error Handling**: Wrap all API calls in try-catch blocks
3. **Logging**: Enable debug mode for troubleshooting but disable in production
4. **Timeouts**: Set appropriate timeouts (60 seconds recommended)
5. **Idempotency**: Handle duplicate requests gracefully
6. **WHOIS Compliance**: Respect privacy/ID protection requirements
7. **Rate Limiting**: Implement rate limiting for API calls
8. **Sync Operations**: Regularly sync domain status with registrar
9. **Auth Codes**: Securely handle and transmit EPP auth codes
10. **TLD Support**: Check and document supported TLDs and their requirements
