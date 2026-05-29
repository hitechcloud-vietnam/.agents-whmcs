# WHMCS Registrar Module Development

## Overview

Registrar modules manage domain registration, transfer, renewal, and DNS management with domain registries.

## Module Structure

```
modules/registrars/yourregistrar/
├── yourregistrar.php       # Main module file
├── yourregistrar_api.php   # API client
└── templates/
    └── manage.tpl          # Management template
```

## Required Functions

### Config

```php
<?php
function yourregistrar_config(): array
{
    return [
        'FriendlyName' => [
            'Type' => 'System',
            'Value' => 'Your Registrar',
        ],
        'apiKey' => [
            'FriendlyName' => 'API Key',
            'Type' => 'password',
            'Size' => '40',
            'Description' => 'Your registrar API key',
        ],
        'apiSecret' => [
            'FriendlyName' => 'API Secret',
            'Type' => 'password',
            'Size' => '40',
        ],
        'environment' => [
            'FriendlyName' => 'Environment',
            'Type' => 'dropdown',
            'Options' => 'Live,Test',
            'Default' => 'Test',
        ],
        'defaultNameservers' => [
            'FriendlyName' => 'Default Nameservers',
            'Type' => 'text',
            'Size' => '100',
            'Description' => 'Comma-separated default nameservers',
        ],
    ];
}
```

### RegisterDomain

```php
<?php
function yourregistrar_RegisterDomain(array $params): array
{
    try {
        $api = new RegistrarApiClient($params);
        
        $result = $api->registerDomain([
            'domain' => $params['sld'] . '.' . $params['tld'],
            'period' => $params['regperiod'],
            ' registrant' => [
                'name' => $params['firstname'] . ' ' . $params['lastname'],
                'organization' => $params['companyname'],
                'email' => $params['email'],
                'address1' => $params['address1'],
                'address2' => $params['address2'],
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
        
        return ['error' => $result['error'] ?? 'Registration failed'];
        
    } catch (Exception $e) {
        return ['error' => $e->getMessage()];
    }
}
```

### TransferDomain

```php
<?php
function yourregistrar_TransferDomain(array $params): array
{
    try {
        $api = new RegistrarApiClient($params);
        
        $result = $api->transferDomain([
            'domain' => $params['sld'] . '.' . $params['tld'],
            'auth_code' => $params['eppcode'],
            'registrant' => [
                'name' => $params['firstname'] . ' ' . $params['lastname'],
                'email' => $params['email'],
                'country' => $params['country'],
            ],
        ]);
        
        if ($result['success']) {
            return ['success' => true];
        }
        
        return ['error' => $result['error'] ?? 'Transfer failed'];
        
    } catch (Exception $e) {
        return ['error' => $e->getMessage()];
    }
}
```

### RenewDomain

```php
<?php
function yourregistrar_RenewDomain(array $params): array
{
    try {
        $api = new RegistrarApiClient($params);
        
        $result = $api->renewDomain([
            'domain' => $params['sld'] . '.' . $params['tld'],
            'period' => $params['regperiod'],
        ]);
        
        if ($result['success']) {
            return ['success' => true];
        }
        
        return ['error' => $result['error'] ?? 'Renewal failed'];
        
    } catch (Exception $e) {
        return ['error' => $e->getMessage()];
    }
}
```

### GetNameservers

```php
<?php
function yourregistrar_GetNameservers(array $params): array
{
    try {
        $api = new RegistrarApiClient($params);
        
        $result = $api->getNameservers($params['sld'] . '.' . $params['tld']);
        
        if ($result['success']) {
            return [
                'success' => true,
                'ns1' => $result['nameservers'][0] ?? '',
                'ns2' => $result['nameservers'][1] ?? '',
                'ns3' => $result['nameservers'][2] ?? '',
                'ns4' => $result['nameservers'][3] ?? '',
            ];
        }
        
        return ['error' => $result['error']];
        
    } catch (Exception $e) {
        return ['error' => $e->getMessage()];
    }
}
```

### SaveNameservers

```php
<?php
function yourregistrar_SaveNameservers(array $params): array
{
    try {
        $api = new RegistrarApiClient($params);
        
        $nameservers = [];
        if (!empty($params['ns1'])) {
            $nameservers[] = $params['ns1'];
        }
        if (!empty($params['ns2'])) {
            $nameservers[] = $params['ns2'];
        }
        if (!empty($params['ns3'])) {
            $nameservers[] = $params['ns3'];
        }
        if (!empty($params['ns4'])) {
            $nameservers[] = $params['ns4'];
        }
        
        $result = $api->setNameservers(
            $params['sld'] . '.' . $params['tld'],
            $nameservers
        );
        
        if ($result['success']) {
            return ['success' => true];
        }
        
        return ['error' => $result['error']];
        
    } catch (Exception $e) {
        return ['error' => $e->getMessage()];
    }
}
```

### GetContactDetails

```php
<?php
function yourregistrar_GetContactDetails(array $params): array
{
    try {
        $api = new RegistrarApiClient($params);
        
        $result = $api->getContacts($params['sld'] . '.' . $params['tld']);
        
        if ($result['success']) {
            return [
                'success' => true,
                'Registrant' => $result['contacts']['registrant'] ?? [],
                'Admin' => $result['contacts']['admin'] ?? [],
                'Technical' => $result['contacts']['technical'] ?? [],
                'Billing' => $result['contacts']['billing'] ?? [],
            ];
        }
        
        return ['error' => $result['error']];
        
    } catch (Exception $e) {
        return ['error' => $e->getMessage()];
    }
}
```

### SaveContactDetails

```php
<?php
function yourregistrar_SaveContactDetails(array $params): array
{
    try {
        $api = new RegistrarApiClient($params);
        
        $contacts = [
            'Registrant' => $params['Registrant'] ?? [],
            'Admin' => $params['Admin'] ?? [],
            'Technical' => $params['Technical'] ?? [],
            'Billing' => $params['Billing'] ?? [],
        ];
        
        $result = $api->setContacts(
            $params['sld'] . '.' . $params['tld'],
            $contacts
        );
        
        if ($result['success']) {
            return ['success' => true];
        }
        
        return ['error' => $result['error']];
        
    } catch (Exception $e) {
        return ['error' => $e->getMessage()];
    }
}
```

### GetRegistrarLock

```php
<?php
function yourregistrar_GetRegistrarLock(array $params): array
{
    try {
        $api = new RegistrarApiClient($params);
        
        $result = $api->getLockStatus($params['sld'] . '.' . $params['tld']);
        
        return [
            'success' => true,
            'lockstatus' => $result['locked'] ? 'locked' : 'unlocked',
        ];
        
    } catch (Exception $e) {
        return ['error' => $e->getMessage()];
    }
}
```

### SaveRegistrarLock

```php
<?php
function yourregistrar_SaveRegistrarLock(array $params): array
{
    try {
        $api = new RegistrarApiClient($params);
        
        $lock = strtolower($params['lockenabled']) === 'true' || $params['lockenabled'] === '1';
        
        $result = $api->setLock(
            $params['sld'] . '.' . $params['tld'],
            $lock
        );
        
        if ($result['success']) {
            return ['success' => true];
        }
        
        return ['error' => $result['error']];
        
    } catch (Exception $e) {
        return ['error' => $e->getMessage()];
    }
}
```

### GetEPPCodes

```php
<?php
function yourregistrar_GetEPPCodes(array $params): array
{
    try {
        $api = new RegistrarApiClient($params);
        
        $result = $api->getEppCode($params['sld'] . '.' . $params['tld']);
        
        if ($result['success']) {
            return [
                'success' => true,
                'eppcode' => $result['eppcode'],
            ];
        }
        
        return ['error' => $result['error']];
        
    } catch (Exception $e) {
        return ['error' => $e->getMessage()];
    }
}
```

### Sync

```php
<?php
function yourregistrar_Sync(array $params): array
{
    try {
        $api = new RegistrarApiClient($params);
        
        $result = $api->getDomainInfo($params['sld'] . '.' . $params['tld']);
        
        if ($result['success']) {
            return [
                'success' => true,
                'expiry' => $result['expiry_date'],
                'status' => $result['status'],
            ];
        }
        
        return ['error' => $result['error']];
        
    } catch (Exception $e) {
        return ['error' => $e->getMessage()];
    }
}
```

## API Client Class

```php
<?php
class RegistrarApiClient
{
    private string $apiKey;
    private string $apiSecret;
    private string $baseUrl;
    private bool $testMode;
    
    public function __construct(array $params)
    {
        $this->apiKey = $params['apiKey'];
        $this->apiSecret = $params['apiSecret'];
        $this->testMode = ($params['environment'] ?? 'Test') === 'Test';
        $this->baseUrl = $this->testMode
            ? 'https://api.test.registrar.com/v1'
            : 'https://api.registrar.com/v1';
    }
    
    public function registerDomain(array $data): array
    {
        return $this->request('POST', '/domains/register', $data);
    }
    
    public function transferDomain(array $data): array
    {
        return $this->request('POST', '/domains/transfer', $data);
    }
    
    public function renewDomain(array $data): array
    {
        return $this->request('POST', '/domains/renew', $data);
    }
    
    public function getNameservers(string $domain): array
    {
        return $this->request('GET', "/domains/{$domain}/nameservers");
    }
    
    public function setNameservers(string $domain, array $nameservers): array
    {
        return $this->request('PUT', "/domains/{$domain}/nameservers", [
            'nameservers' => $nameservers,
        ]);
    }
    
    public function getContacts(string $domain): array
    {
        return $this->request('GET', "/domains/{$domain}/contacts");
    }
    
    public function setContacts(string $domain, array $contacts): array
    {
        return $this->request('PUT', "/domains/{$domain}/contacts", $contacts);
    }
    
    public function getLockStatus(string $domain): array
    {
        return $this->request('GET', "/domains/{$domain}/lock");
    }
    
    public function setLock(string $domain, bool $lock): array
    {
        return $this->request('POST', "/domains/{$domain}/lock", [
            'lock' => $lock,
        ]);
    }
    
    public function getEppCode(string $domain): array
    {
        return $this->request('POST', "/domains/{$domain}/eppcode");
    }
    
    public function getDomainInfo(string $domain): array
    {
        return $this->request('GET', "/domains/{$domain}");
    }
    
    private function request(string $method, string $endpoint, array $data = []): array
    {
        $url = $this->baseUrl . $endpoint;
        $timestamp = time();
        
        $ch = curl_init($url);
        curl_setopt_array($ch, [
            CURLOPT_RETURNTRANSFER => true,
            CURLOPT_TIMEOUT => 30,
            CURLOPT_HTTPHEADER => [
                'Authorization: Bearer ' . $this->getAuthToken($method, $endpoint, $timestamp),
                'Content-Type: application/json',
                'X-Timestamp: ' . $timestamp,
            ],
        ]);
        
        if ($method === 'POST' || $method === 'PUT') {
            curl_setopt($ch, CURLOPT_POST, true);
            curl_setopt($ch, CURLOPT_POSTFIELDS, json_encode($data));
        }
        
        if ($method === 'DELETE') {
            curl_setopt($ch, CURLOPT_CUSTOMREQUEST, 'DELETE');
        }
        
        $response = curl_exec($ch);
        $httpCode = curl_getinfo($ch, CURLINFO_HTTP_CODE);
        curl_close($ch);
        
        $result = json_decode($response, true);
        
        if ($httpCode >= 200 && $httpCode < 300) {
            return array_merge(['success' => true], $result ?? []);
        }
        
        return [
            'success' => false,
            'error' => $result['message'] ?? 'API request failed',
        ];
    }
    
    private function getAuthToken(string $method, string $endpoint, int $timestamp): string
    {
        $payload = $method . $endpoint . $timestamp . json_encode([]);
        $signature = hash_hmac('sha256', $payload, $this->apiSecret);
        
        return $this->apiKey . ':' . $signature;
    }
}
```

## Best Practices

1. **Return proper arrays** - Use `['success' => true]` or `['error' => 'message']`
2. **Handle all errors** - Catch exceptions and return errors
3. **Log API calls** - Track all registrar API interactions
4. **Implement sync** - Keep domain expiry dates accurate
5. **Test thoroughly** - Test all registry operations

## Related Documentation

- [WHMCS Domain Hooks](/docs/whmcs-domain-hooks.md)