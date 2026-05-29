# WHMCS Registrar Module Builder

## Concept

A WHMCS registrar module handles domain registration, transfer, renewal, and management through domain registrars. It communicates with registry APIs to register domains, manage DNS, and handle contact information.

## File Structure

```
/modules/registrars/
├── yourregistrar/
│   ├── yourregistrar.php    # Main registrar file
│   ├── includes/
│   │   └── API.php          # API client
│   └── README.md
```

## Core Registrar Functions

```php
<?php
/**
 * Registrar Module: Your Registrar
 * Version: 1.0.0
 * Description: Domain registrar module for...
 */

if (!defined("WHMCS")) {
    die("This file cannot be accessed directly");
}

function yourregistrar_MetaData()
{
    return [
        'DisplayName' => 'Your Registrar',
        'APIVersion' => '1.0',
    ];
}

function yourregistrar_config(array $params)
{
    return [
        'FriendlyName' => ['Type' => 'System', 'Value' => 'Your Registrar'],
        'api_username' => [
            'FriendlyName' => 'API Username',
            'Type' => 'text',
            'Size' => '40',
        ],
        'api_password' => [
            'FriendlyName' => 'API Password',
            'Type' => 'password',
            'Size' => '40',
        ],
        'api_key' => [
            'FriendlyName' => 'API Key',
            'Type' => 'password',
            'Size' => '40',
        ],
        'sandbox' => [
            'FriendlyName' => 'Sandbox Mode',
            'Type' => 'yesno',
        ],
    ];
}

// Register domain
function yourregistrar_RegisterDomain(array $params)
{
    try {
        $api = new YourRegistrarAPI($params);
        
        $result = $api->registerDomain([
            'domain' => $params['sld'] . '.' . $params['tld'],
            'years' => $params['regperiod'],
            'nameservers' => [
                ['nameserver' => $params['ns1']],
                ['nameserver' => $params['ns2']],
            ],
            'registrant' => [
                'firstname' => $params['firstname'],
                'lastname' => $params['lastname'],
                'organization' => $params['companyname'],
                'email' => $params['email'],
                'address1' => $params['address1'],
                'city' => $params['city'],
                'state' => $params['state'],
                'postcode' => $params['postcode'],
                'country' => $params['country'],
                'phone' => $params['phonenumber'],
            ],
        ]);
        
        return ['success' => true, 'domainid' => $result['id']];
    } catch (\Exception $e) {
        return ['error' => $e->getMessage()];
    }
}

// Transfer domain
function yourregistrar_TransferDomain(array $params)
{
    try {
        $api = new YourRegistrarAPI($params);
        
        $result = $api->transferDomain([
            'domain' => $params['sld'] . '.' . $params['tld'],
            'authcode' => $params['transfersecret'],
            'registrant' => [
                'email' => $params['email'],
            ],
        ]);
        
        return ['success' => true];
    } catch (\Exception $e) {
        return ['error' => $e->getMessage()];
    }
}

// Renew domain
function yourregistrar_RenewDomain(array $params)
{
    try {
        $api = new YourRegistrarAPI($params);
        
        $result = $api->renewDomain([
            'domain' => $params['sld'] . '.' . $params['tld'],
            'years' => $params['regperiod'],
        ]);
        
        return ['success' => true];
    } catch (\Exception $e) {
        return ['error' => $e->getMessage()];
    }
}

// Release domain (transfer away)
function yourregistrar_ReleaseDomain(array $params)
{
    try {
        $api = new YourRegistrarAPI($params);
        
        $result = $api->transferAway([
            'domain' => $params['sld'] . '.' . $params['tld'],
            'tag' => $params['newRegistrarTag'],
        ]);
        
        return ['success' => true];
    } catch (\Exception $e) {
        return ['error' => $e->getMessage()];
    }
}

// Get nameservers
function yourregistrar_GetNameservers(array $params)
{
    try {
        $api = new YourRegistrarAPI($params);
        
        $result = $api->getNameServers($params['sld'] . '.' . $params['tld']);
        
        return [
            'success' => true,
            'ns1' => $result['nameservers'][0],
            'ns2' => $result['nameservers'][1],
            'ns3' => $result['nameservers'][2] ?? '',
            'ns4' => $result['nameservers'][3] ?? '',
            'ns5' => $result['nameservers'][4] ?? '',
        ];
    } catch (\Exception $e) {
        return ['error' => $e->getMessage()];
    }
}

// Save nameservers
function yourregistrar_SaveNameservers(array $params)
{
    try {
        $api = new YourRegistrarAPI($params);
        
        $nameservers = array_filter([
            $params['ns1'],
            $params['ns2'],
            $params['ns3'],
            $params['ns4'],
            $params['ns5'],
        ]);
        
        $api->setNameServers($params['sld'] . '.' . $params['tld'], $nameservers);
        
        return ['success' => true];
    } catch (\Exception $e) {
        return ['error' => $e->getMessage()];
    }
}

// Get contact details
function yourregistrar_GetContactDetails(array $params)
{
    try {
        $api = new YourRegistrarAPI($params);
        
        $result = $api->getContacts($params['sld'] . '.' . $params['tld']);
        
        return [
            'success' => true,
            'Registrant' => $result['registrant'],
            'Admin' => $result['admin'],
            'Tech' => $result['tech'],
            'Billing' => $result['billing'],
        ];
    } catch (\Exception $e) {
        return ['error' => $e->getMessage()];
    }
}

// Save contact details
function yourregistrar_SaveContactDetails(array $params)
{
    try {
        $api = new YourRegistrarAPI($params);
        
        $api->updateContacts($params['sld'] . '.' . $params['tld'], [
            'Registrant' => $params['Registrant'],
            'Admin' => $params['Admin'],
            'Tech' => $params['Tech'],
            'Billing' => $params['Billing'],
        ]);
        
        return ['success' => true];
    } catch (\Exception $e) {
        return ['error' => $e->getMessage()];
    }
}

// Get domain info
function yourregistrar_GetDomainInformation(array $params)
{
    try {
        $api = new YourRegistrarAPI($params);
        
        $result = $api->getDomainInfo($params['sld'] . '.' . $params['tld']);
        
        return [
            'success' => true,
            'registrationstatus' => $result['status'] === 'active' ? 'Active' : 'Expired',
            'registrationdate' => $result['created_date'],
            'expirydate' => $result['expiry_date'],
            'nextduedate' => $result['expiry_date'],
            'dnsmanagement' => $result['dns_enabled'],
            'emailforwarding' => $result['email_forwarding_enabled'],
            'idprotection' => $result['privacy_enabled'],
        ];
    } catch (\Exception $e) {
        return ['error' => $e->getMessage()];
    }
}

// DNS management
function yourregistrar_RegisterNameserver(array $params)
{
    try {
        $api = new YourRegistrarAPI($params);
        
        $api->registerChildHost($params['nameserver'], $params['ip']);
        
        return ['success' => true];
    } catch (\Exception $e) {
        return ['error' => $e->getMessage()];
    }
}

function yourregistrar_DeleteNameserver(array $params)
{
    try {
        $api = new YourRegistrarAPI($params);
        
        $api->deleteChildHost($params['nameserver']);
        
        return ['success' => true];
    } catch (\Exception $e) {
        return ['error' => $e->getMessage()];
    }
}

// Sync domain status
function yourregistrar_Sync(array $params)
{
    try {
        $api = new YourRegistrarAPI($params);
        
        $result = $api->getDomainInfo($params['sld'] . '.' . $params['tld']);
        
        return [
            'success' => true,
            'active' => $result['status'] === 'active',
            'expired' => $result['status'] === 'expired',
            'expirydate' => $result['expiry_date'],
            'daysuntilexpiry' => $result['days_remaining'],
        ];
    } catch (\Exception $e) {
        return ['error' => $e->getMessage()];
    }
}

// Transfer sync
function yourregistrar_TransferSync(array $params)
{
    try {
        $api = new YourRegistrarAPI($params);
        
        $result = $api->getTransferStatus($params['sld'] . '.' . $params['tld']);
        
        return [
            'success' => true,
            'status' => $result['status'],
            'message' => $result['message'],
        ];
    } catch (\Exception $e) {
        return ['error' => $e->getMessage()];
    }
}
```

## API Client Implementation

```php
<?php
namespace WHMCS\Module\Registrar\YourRegistrar;

class YourRegistrarAPI
{
    private $username;
    private $password;
    private $apiKey;
    private $sandbox;
    private $baseUrl;
    
    public function __construct(array $params)
    {
        $this->username = $params['api_username'];
        $this->password = decrypt($params['api_password']);
        $this->apiKey = decrypt($params['api_key']);
        $this->sandbox = !empty($params['sandbox']);
        
        $this->baseUrl = $this->sandbox
            ? 'https://sandbox.yourregistrar.com/api'
            : 'https://api.yourregistrar.com/v1';
    }
    
    private function request(string $method, string $endpoint, array $data = [])
    {
        $url = $this->baseUrl . '/' . ltrim($endpoint, '/');
        
        $ch = curl_init();
        curl_setopt_array($ch, [
            CURLOPT_URL => $url,
            CURLOPT_RETURNTRANSFER => true,
            CURLOPT_TIMEOUT => 30,
            CURLOPT_HTTPHEADER => [
                'Authorization: Bearer ' . $this->apiKey,
                'Content-Type: application/json',
            ],
        ]);
        
        if ($method === 'POST') {
            curl_setopt($ch, CURLOPT_POST, true);
            curl_setopt($ch, CURLOPT_POSTFIELDS, json_encode($data));
        }
        
        $response = curl_exec($ch);
        $httpCode = curl_getinfo($ch, CURLINFO_HTTP_CODE);
        curl_close($ch);
        
        $result = json_decode($response, true);
        
        if ($httpCode >= 400) {
            throw new \Exception($result['message'] ?? 'API Error');
        }
        
        return $result;
    }
    
    public function registerDomain(array $data)
    {
        return $this->request('POST', '/domains/register', $data);
    }
    
    public function transferDomain(array $data)
    {
        return $this->request('POST', '/domains/transfer', $data);
    }
    
    public function renewDomain(array $data)
    {
        return $this->request('POST', '/domains/renew', $data);
    }
    
    public function getNameServers(string $domain)
    {
        $result = $this->request('GET', '/domains/' . $domain . '/nameservers');
        return $result;
    }
    
    public function setNameServers(string $domain, array $nameservers)
    {
        return $this->request('PUT', '/domains/' . $domain . '/nameservers', [
            'nameservers' => $nameservers,
        ]);
    }
    
    public function getContacts(string $domain)
    {
        return $this->request('GET', '/domains/' . $domain . '/contacts');
    }
    
    public function updateContacts(string $domain, array $contacts)
    {
        return $this->request('PUT', '/domains/' . $domain . '/contacts', $contacts);
    }
    
    public function getDomainInfo(string $domain)
    {
        return $this->request('GET', '/domains/' . $domain);
    }
    
    public function transferAway(string $domain, string $tag)
    {
        return $this->request('POST', '/domains/' . $domain . '/transfer-away', [
            'tag' => $tag,
        ]);
    }
    
    public function registerChildHost(string $hostname, string $ip)
    {
        return $this->request('POST', '/hosts', [
            'hostname' => $hostname,
            'addresses' => [$ip],
        ]);
    }
    
    public function deleteChildHost(string $hostname)
    {
        return $this->request('DELETE', '/hosts/' . $hostname);
    }
    
    public function getTransferStatus(string $domain)
    {
        return $this->request('GET', '/domains/' . $domain . '/transfer');
    }
}
```

## Step-by-Step Implementation

1. Create module directory in `/modules/registrars/yourregistrar/`
2. Implement all required registrar functions
3. Create API client class
4. Add error handling and logging
5. Implement transfer sync functionality
6. Test in sandbox environment

## Implementation Checklist

- [ ] Create module directory structure
- [ ] Implement MetaData function
- [ ] Implement config array
- [ ] Implement RegisterDomain
- [ ] Implement TransferDomain
- [ ] Implement RenewDomain
- [ ] Implement ReleaseDomain
- [ ] Implement GetNameservers
- [ ] Implement SaveNameservers
- [ ] Implement GetContactDetails
- [ ] Implement SaveContactDetails
- [ ] Implement GetDomainInformation
- [ ] Implement RegisterNameserver
- [ ] Implement DeleteNameserver
- [ ] Implement Sync function
- [ ] Implement TransferSync
- [ ] Create API client
- [ ] Test in sandbox