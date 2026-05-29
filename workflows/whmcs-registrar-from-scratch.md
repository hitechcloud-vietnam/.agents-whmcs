# WHMCS Registrar From Scratch Workflow

## Description
Create a custom domain registrar module for WHMCS from scratch.

## Prerequisites
- WHMCS installation
- Registrar API documentation
- PHP 8.1+
- SSL certificate

## Module Structure
```
modules/registrars/
└── clicodes_registrar/
    ├── clicodes_registrar.php      # Main module
    ├── API/
    │   └── Client.php               # API client
    ├── functions.php                # Helper functions
    ├── LICENSE
    └── README.md
```

## Steps

### Step 1: Create Module Directory
```bash
mkdir -p /var/www/whmcs/modules/registrars/clicodes_registrar
mkdir -p /var/www/whmcs/modules/registrars/clicodes_registrar/API
```

### Step 2: Create Main Registrar File
```php
<?php
/**
 * WHMCS Registrar Module - CLICodes Registrar
 *
 * @copyright Copyright (c) 2024 Your Company
 * @license https://example.com/license
 * @version 1.0.0
 */

if (!defined("WHMCS")) {
    die("This file cannot be accessed directly");
}

/**
 * Define registrar meta data
 */
function clicodes_registrar_MetaData()
{
    return [
        'DisplayName' => 'CLICodes Domain Registrar',
        'APIVersion' => '1.1',
        'RequiresRegSync' => true,
        'Parameters' => [
            'api_key' => ['Type' => 'text'],
            'api_secret' => ['Type' => 'password'],
            'sandbox' => ['Type' => 'yesno'],
        ],
    ];
}

/**
 * Get configuration array
 */
function clicodes_registrar_getConfigArray()
{
    return [
        'FriendlyName' => [
            'Type' => 'System',
            'Value' => 'CLICodes Registrar',
        ],
        'Description' => [
            'Type' => 'System',
            'Value' => 'Connect your domain management to CLICodes Registrar API.',
        ],
        'apiKey' => [
            'Type' => 'text',
            'Label' => 'API Key',
            'Size' => '40',
            'Description' => 'Enter your CLICodes API key.',
        ],
        'apiSecret' => [
            'Type' => 'password',
            'Label' => 'API Secret',
            'Size' => '40',
            'Description' => 'Enter your CLICodes API secret.',
        ],
        'sandbox' => [
            'Type' => 'yesno',
            'Label' => 'Sandbox Mode',
            'Description' => 'Enable sandbox for testing.',
            'Default' => 'on',
        ],
        'accountId' => [
            'Type' => 'text',
            'Label' => 'Account ID',
            'Description' => 'Your registrar account ID.',
        ],
        'defaultContact' => [
            'Type' => 'text',
            'Label' => 'Default Contact ID',
            'Description' => 'Default contact for new registrations.',
        ],
    ];
}

/**
 * Register domain
 */
function clicodes_registrar_RegisterDomain($params)
{
    $api = new CLICodesRegistrarAPI($params);
    
    $domainName = $params['domainname'];
    $registrationPeriod = $params['regperiod'];
    $registrarLock = $params['lockenabled'] ? '1' : '0';
    
    // Prepare contact details
    $ registrantDetails = [
        'name' => $params['firstname'] . ' ' . $params['lastname'],
        'org' => $params['companyname'] ?: 'N/A',
        'email' => $params['email'],
        'address1' => $params['address1'],
        'address2' => $params['address2'],
        'city' => $params['city'],
        'state' => $params['state'],
        'postcode' => $params['postcode'],
        'country' => $params['country'],
        'phone' => $params['phonenumber'],
        'phone_cc' => $params['countrycode'],
    ];
    
    try {
        $response = $api->request('POST', '/domains/register', [
            'domain' => $domainName,
            'period' => $registrationPeriod,
            'registrant' => $registrantDetails,
            'dns_management' => $params['dnsmanagement'] ?? false,
            'email_forwarding' => $params['emailforwarding'] ?? false,
            'id_protection' => $params['idprotection'] ?? false,
        ]);
        
        if ($response['success']) {
            return [
                'success' => true,
                'domain' => $domainName,
            ];
        }
        
        return [
            'success' => false,
            'error' => $response['message'] ?? 'Registration failed',
        ];
        
    } catch (Exception $e) {
        return [
            'success' => false,
            'error' => $e->getMessage(),
        ];
    }
}

/**
 * Transfer domain
 */
function clicodes_registrar_TransferDomain($params)
{
    $api = new CLICodesRegistrarAPI($params);
    
    $domainName = $params['domainname'];
    $transferSecret = $params['transfersecret'];
    
    try {
        $response = $api->request('POST', '/domains/transfer', [
            'domain' => $domainName,
            'auth_code' => $transferSecret,
            'period' => $params['regperiod'] ?? 1,
        ]);
        
        if ($response['success']) {
            return [
                'success' => true,
                'transferid' => $response['transfer_id'],
            ];
        }
        
        return [
            'success' => false,
            'error' => $response['message'] ?? 'Transfer failed',
        ];
        
    } catch (Exception $e) {
        return [
            'success' => false,
            'error' => $e->getMessage(),
        ];
    }
}

/**
 * Renew domain
 */
function clicodes_registrar_RenewDomain($params)
{
    $api = new CLICodesRegistrarAPI($params);
    
    $domainName = $params['domainname'];
    $period = $params['regperiod'];
    
    try {
        $response = $api->request('POST', '/domains/renew', [
            'domain' => $domainName,
            'period' => $period,
        ]);
        
        if ($response['success']) {
            return [
                'success' => true,
                'expirydate' => $response['expiry_date'],
            ];
        }
        
        return [
            'success' => false,
            'error' => $response['message'] ?? 'Renewal failed',
        ];
        
    } catch (Exception $e) {
        return [
            'success' => false,
            'error' => $e->getMessage(),
        ];
    }
}

/**
 * Release domain
 */
function clicodes_registrar_ReleaseDomain($params)
{
    $api = new CLICodesRegistrarAPI($params);
    
    $domainName = $params['domainname'];
    $newRegistrar = $params['newRegistrar'];
    
    try {
        $response = $api->request('POST', '/domains/release', [
            'domain' => $domainName,
            'new_registrar' => $newRegistrar,
        ]);
        
        return [
            'success' => $response['success'],
            'error' => $response['message'] ?? null,
        ];
        
    } catch (Exception $e) {
        return [
            'success' => false,
            'error' => $e->getMessage(),
        ];
    }
}

/**
 * Get domain nameservers
 */
function clicodes_registrar_GetNameservers($params)
{
    $api = new CLICodesRegistrarAPI($params);
    
    try {
        $response = $api->request('GET', '/domains/' . $params['domainname'] . '/nameservers');
        
        if ($response['success']) {
            return [
                'success' => true,
                'ns1' => $response['nameservers'][0] ?? '',
                'ns2' => $response['nameservers'][1] ?? '',
                'ns3' => $response['nameservers'][2] ?? '',
                'ns4' => $response['nameservers'][3] ?? '',
                'ns5' => $response['nameservers'][4] ?? '',
            ];
        }
        
        return [
            'success' => false,
            'error' => $response['message'] ?? 'Failed to get nameservers',
        ];
        
    } catch (Exception $e) {
        return [
            'success' => false,
            'error' => $e->getMessage(),
        ];
    }
}

/**
 * Save domain nameservers
 */
function clicodes_registrar_SaveNameservers($params)
{
    $api = new CLICodesRegistrarAPI($params);
    
    $nameservers = array_filter([
        $params['ns1'],
        $params['ns2'],
        $params['ns3'],
        $params['ns4'],
        $params['ns5'],
    ]);
    
    try {
        $response = $api->request('PUT', '/domains/' . $params['domainname'] . '/nameservers', [
            'nameservers' => $nameservers,
        ]);
        
        return [
            'success' => $response['success'],
            'error' => $response['message'] ?? null,
        ];
        
    } catch (Exception $e) {
        return [
            'success' => false,
            'error' => $e->getMessage(),
        ];
    }
}

/**
 * Get registrar lock status
 */
function clicodes_registrar_GetRegistrarLock($params)
{
    $api = new CLICodesRegistrarAPI($params);
    
    try {
        $response = $api->request('GET', '/domains/' . $params['domainname'] . '/lock');
        
        return [
            'success' => true,
            'lockenabled' => ($response['locked'] ?? false) ? 'locked' : 'unlocked',
        ];
        
    } catch (Exception $e) {
        return [
            'success' => false,
            'error' => $e->getMessage(),
        ];
    }
}

/**
 * Save registrar lock
 */
function clicodes_registrar_SaveRegistrarLock($params)
{
    $api = new CLICodesRegistrarAPI($params);
    
    $lock = ($params['lockenabled'] == 'locked') ? true : false;
    
    try {
        $response = $api->request('PUT', '/domains/' . $params['domainname'] . '/lock', [
            'lock' => $lock,
        ]);
        
        return [
            'success' => $response['success'],
            'error' => $response['message'] ?? null,
        ];
        
    } catch (Exception $e) {
        return [
            'success' => false,
            'error' => $e->getMessage(),
        ];
    }
}

/**
 * Get WHOIS information
 */
function clicodes_registrar_GetWHOIS($params)
{
    $api = new CLICodesRegistrarAPI($params);
    
    try {
        $response = $api->request('GET', '/domains/' . $params['domainname'] . '/whois');
        
        if ($response['success']) {
            return [
                'success' => true,
                'Registrant' => $response['registrant'],
                'Admin' => $response['admin'],
                'Tech' => $response['tech'],
                'Billing' => $response['billing'],
            ];
        }
        
        return [
            'success' => false,
            'error' => $response['message'] ?? 'Failed to get WHOIS',
        ];
        
    } catch (Exception $e) {
        return [
            'success' => false,
            'error' => $e->getMessage(),
        ];
    }
}

/**
 * Save WHOIS information
 */
function clicodes_registrar_SaveWHOIS($params)
{
    $api = new CLICodesRegistrarAPI($params);
    
    try {
        $response = $api->request('PUT', '/domains/' . $params['domainname'] . '/whois', [
            'registrant' => $params['Registrant'],
            'admin' => $params['Admin'],
            'tech' => $params['Tech'],
            'billing' => $params['Billing'],
        ]);
        
        return [
            'success' => $response['success'],
            'error' => $response['message'] ?? null,
        ];
        
    } catch (Exception $e) {
        return [
            'success' => false,
            'error' => $e->getMessage(),
        ];
    }
}

/**
 * Get domain transfer status
 */
function clicodes_registrar_TransferSync($params)
{
    $api = new CLICodesRegistrarAPI($params);
    
    try {
        $response = $api->request('GET', '/domains/' . $params['domainname'] . '/transfer/status');
        
        return [
            'success' => true,
            'completed' => ($response['status'] == 'completed'),
            'pending' => ($response['status'] == 'pending'),
            'rejected' => ($response['status'] == 'rejected'),
            'reason' => $response['reason'] ?? null,
        ];
        
    } catch (Exception $e) {
        return [
            'success' => false,
            'error' => $e->getMessage(),
        ];
    }
}

/**
 * Sync domain data
 */
function clicodes_registrar_Sync($params)
{
    $api = new CLICodesRegistrarAPI($params);
    
    try {
        $response = $api->request('GET', '/domains/' . $params['domainname']);
        
        if ($response['success']) {
            return [
                'success' => true,
                'expirydate' => $response['expiry_date'],
                'registrationstatus' => $response['status'],
                'dnsmanagement' => $response['dns_management'] ?? false,
                'emailforwarding' => $response['email_forwarding'] ?? false,
                'idprotection' => $response['id_protection'] ?? false,
            ];
        }
        
        return [
            'success' => false,
            'error' => $response['message'] ?? 'Sync failed',
        ];
        
    } catch (Exception $e) {
        return [
            'success' => false,
            'error' => $e->getMessage(),
        ];
    }
}
```

### Step 3: Create API Client
```php
<?php
/**
 * CLICodes Registrar API Client
 */

if (!defined("WHMCS")) {
    die("This file cannot be accessed directly");
}

class CLICodesRegistrarAPI
{
    private $apiKey;
    private $apiSecret;
    private $sandbox;
    private $baseUrl;
    
    public function __construct($params)
    {
        $this->apiKey = $params['apiKey'] ?? '';
        $this->apiSecret = $params['apiSecret'] ?? '';
        $this->sandbox = $params['sandbox'] ?? false;
        
        $this->baseUrl = $this->sandbox
            ? 'https://api-sandbox.clicodes-registrar.example.com/v1'
            : 'https://api.clicodes-registrar.example.com/v1';
    }
    
    /**
     * Make API request
     */
    public function request($method, $endpoint, $data = [])
    {
        $url = $this->baseUrl . $endpoint;
        $timestamp = time();
        
        $headers = [
            'Content-Type: application/json',
            'X-Api-Key: ' . $this->apiKey,
            'X-Timestamp: ' . $timestamp,
            'X-Signature: ' . $this->generateSignature($method, $endpoint, $data, $timestamp),
        ];
        
        $ch = curl_init();
        
        $options = [
            CURLOPT_URL => $url,
            CURLOPT_RETURNTRANSFER => true,
            CURLOPT_HTTPHEADER => $headers,
            CURLOPT_TIMEOUT => 30,
            CURLOPT_SSL_VERIFYPEER => true,
        ];
        
        if ($method === 'POST' || $method === 'PUT') {
            $options[CURLOPT_POST] = true;
            $options[CURLOPT_POSTFIELDS] = json_encode($data);
        }
        
        curl_setopt_array($ch, $options);
        
        $response = curl_exec($ch);
        $httpCode = curl_getinfo($ch, CURLINFO_HTTP_CODE);
        $error = curl_error($ch);
        
        curl_close($ch);
        
        if ($error) {
            throw new Exception('API Error: ' . $error);
        }
        
        $result = json_decode($response, true);
        
        // Log the request
        logModuleCall(
            'clicodes_registrar',
            $method . ' ' . $endpoint,
            $data,
            $result,
            $result
        );
        
        return $result;
    }
    
    /**
     * Generate request signature
     */
    private function generateSignature($method, $endpoint, $data, $timestamp)
    {
        $payload = $method . "\n";
        $payload .= $endpoint . "\n";
        $payload .= json_encode($data) . "\n";
        $payload .= $timestamp;
        
        return hash_hmac('sha256', $payload, $this->apiSecret);
    }
}
```

### Step 4: Install Module
```bash
# Copy module to WHMCS
cp -r clicodes_registrar /var/www/whmcs/modules/registrars/

# Set permissions
chown -R www-data:www-data /var/www/whmcs/modules/registrars/clicodes_registrar
chmod -R 755 /var/www/whmcs/modules/registrars/clicodes_registrar

# Configure in WHMCS Admin
# Go to: Configuration > Domain Registrars
# Find CLICodes Registrar
# Enter API credentials
```

### Step 5: Test Functions
```php
// Test each function with sandbox credentials:
// 1. Register a test domain
// 2. Transfer a domain
// 3. Renew a domain
// 4. Change nameservers
// 5. Toggle registrar lock
// 6. Get WHOIS data
// 7. Sync domain data
```

## Required Functions
| Function | Required | Description |
|----------|----------|-------------|
| MetaData | Yes | Module metadata |
| getConfigArray | Yes | Configuration options |
| RegisterDomain | Yes | Register new domain |
| TransferDomain | Optional | Initiate transfer |
| RenewDomain | Yes | Renew domain |
| TransferSync | Yes | Check transfer status |
| Sync | Yes | Sync domain data |
| GetNameservers | Yes | Get nameservers |
| SaveNameservers | Yes | Set nameservers |
| GetRegistrarLock | Yes | Get lock status |
| SaveRegistrarLock | Yes | Set lock status |

## Tags
- registrar
- domain
- module-development
- development