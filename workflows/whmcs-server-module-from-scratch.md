# WHMCS Server Module From Scratch

## Overview

This workflow guides you through building a complete WHMCS server module from scratch. Server modules allow WHMCS to communicate with your hosting infrastructure to provision and manage services automatically.

## Prerequisites

- WHMCS installation (v8.0+)
- PHP 7.4+ with cURL support
- Understanding of PHP and object-oriented programming
- Access to your hosting provider's API documentation
- Local development environment with WHMCS

## Step-by-Step Instructions

### Step 1: Set Up Module Directory Structure

Create your module in `/modules/servers/YourModuleName/`:

```
/modules/servers/YourModuleName/
    ├── yourmodule.php          # Main module file
    ├── views/
    │   └── widget.php          # Admin dashboard widget (optional)
    ├── lib/
    │   ├── ApiClient.php       # API communication class
    │   └── ConfigValidator.php # Configuration validation
    └── lang/
        └── english.php         # Language strings
```

### Step 2: Create the Main Module File

Create `yourmodule.php` with required functions:

```php
<?php
/**
 * WHMCS Server Module - YourModuleName
 *
 * @copyright Copyright (c) 2024 Your Name
 * @license https://whmcs.com/legal/
 */

if (!defined("WHMCS")) {
    die("This file cannot be accessed directly");
}

/**
 * Define module configuration.
 *
 * @return array Module configuration
 */
function YourModuleName_MetaData()
{
    return [
        'DisplayName' => 'Your Module Name',
        'APIVersion' => '1.1',
        'RequiresServer' => true,
        'DefaultNonSSLPort' => '80',
        'DefaultSSLPort' => '443',
    ];
}

/**
 * Define module configuration options.
 *
 * @return array Configuration options
 */
function YourModuleName_ConfigArray()
{
    return [
        'FriendlyName' => [
            'Type' => 'System',
            'Value' => 'Your Module Name',
        ],
        'ApiKey' => [
            'FriendlyName' => 'API Key',
            'Type' => 'password',
            'Size' => '40',
            'Description' => 'Your API key from the provider',
        ],
        'ApiEndpoint' => [
            'FriendlyName' => 'API Endpoint',
            'Type' => 'text',
            'Size' => '60',
            'Default' => 'https://api.yourprovider.com/v1',
            'Description' => 'The API endpoint URL',
        ],
        'UseSSL' => [
            'FriendlyName' => 'Use SSL',
            'Type' => 'yesno',
            'Description' => 'Connect to API using SSL',
        ],
        'TestMode' => [
            'FriendlyName' => 'Test Mode',
            'Type' => 'yesno',
            'Description' => 'Enable test/sandbox mode',
        ],
    ];
}

/**
 * Test connection to the server.
 *
 * @param array $params Module parameters
 * @return array Test result
 */
function YourModuleName_TestConnection($params)
{
    try {
        $api = new YourModuleName\ApiClient($params);
        $result = $api->testConnection();

        return [
            'success' => true,
            'message' => 'Connection successful',
        ];
    } catch (\Exception $e) {
        return [
            'success' => false,
            'error' => $e->getMessage(),
        ];
    }
}

/**
 * Create hosting account.
 *
 * @param array $params Module parameters
 * @return array Provision result
 */
function YourModuleName_CreateAccount($params)
{
    try {
        $api = new YourModuleName\ApiClient($params);

        $package = [
            'username' => $params['username'],
            'password' => $params['password'],
            'domain' => $params['domain'],
            'package' => $params['configoption1'],
        ];

        $result = $api->createAccount($package);

        if ($result['success']) {
            return [
                'success' => true,
                'accountid' => $result['account_id'],
            ];
        }

        return [
            'success' => false,
            'error' => $result['error'] ?? 'Account creation failed',
        ];
    } catch (\Exception $e) {
        return [
            'success' => false,
            'error' => $e->getMessage(),
        ];
    }
}

/**
 * Terminate hosting account.
 *
 * @param array $params Module parameters
 * @return array Termination result
 */
function YourModuleName_TerminateAccount($params)
{
    try {
        $api = new YourModuleName\ApiClient($params);
        $result = $api->terminateAccount($params['accountid']);

        return ['success' => true];
    } catch (\Exception $e) {
        return [
            'success' => false,
            'error' => $e->getMessage(),
        ];
    }
}

/**
 * Suspend hosting account.
 *
 * @param array $params Module parameters
 * @return array Suspension result
 */
function YourModuleName_SuspendAccount($params)
{
    try {
        $api = new YourModuleName\ApiClient($params);
        $result = $api->suspendAccount($params['accountid']);

        return ['success' => true];
    } catch (\Exception $e) {
        return [
            'success' => false,
            'error' => $e->getMessage(),
        ];
    }
}

/**
 * Unsuspend hosting account.
 *
 * @param array $params Module parameters
 * @return array Unsuspension result
 */
function YourModuleName_UnsuspendAccount($params)
{
    try {
        $api = new YourModuleName\ApiClient($params);
        $result = $api->unsuspendAccount($params['accountid']);

        return ['success' => true];
    } catch (\Exception $e) {
        return [
            'success' => false,
            'error' => $e->getMessage(),
        ];
    }
}

/**
 * Change hosting account password.
 *
 * @param array $params Module parameters
 * @return array Password change result
 */
function YourModuleName_ChangePassword($params)
{
    try {
        $api = new YourModuleName\ApiClient($params);
        $result = $api->changePassword($params['accountid'], $params['password']);

        return ['success' => true];
    } catch (\Exception $e) {
        return [
            'success' => false,
            'error' => $e->getMessage(),
        ];
    }
}

/**
 * Change hosting account package.
 *
 * @param array $params Module parameters
 * @return array Package change result
 */
function YourModuleName_ChangePackage($params)
{
    try {
        $api = new YourModuleName\ApiClient($params);
        $result = $api->changePackage($params['accountid'], $params['configoption1']);

        return ['success' => true];
    } catch (\Exception $e) {
        return [
            'success' => false,
            'error' => $e->getMessage(),
        ];
    }
}

/**
 * Get account details.
 *
 * @param array $params Module parameters
 * @return array Account details
 */
function YourModuleName_AccountDetails($params)
{
    try {
        $api = new YourModuleName\ApiClient($params);
        $result = $api->getAccountDetails($params['accountid']);

        return [
            'success' => true,
            'account' => $result,
        ];
    } catch (\Exception $e) {
        return [
            'success' => false,
            'error' => $e->getMessage(),
        ];
    }
}

/**
 * Usage metrics callback.
 *
 * @param array $params Module parameters
 * @return array Usage statistics
 */
function YourModuleName_UsageMetrics($params)
{
    try {
        $api = new YourModuleName\ApiClient($params);
        $result = $api->getUsageMetrics($params['accountid']);

        return [
            'success' => true,
            'metrics' => [
                'disk' => [
                    'used' => $result['disk_used'],
                    'limit' => $result['disk_limit'],
                ],
                'bandwidth' => [
                    'used' => $result['bw_used'],
                    'limit' => $result['bw_limit'],
                ],
            ],
        ];
    } catch (\Exception $e) {
        logActivity("UsageMetrics Error: " . $e->getMessage());
        return ['success' => false];
    }
}
```

### Step 3: Create the API Client Class

Create `lib/ApiClient.php`:

```php
<?php
namespace YourModuleName;

use WHMCS\Module\Server\ServersProvider;

class ApiClient
{
    private $apiKey;
    private $apiEndpoint;
    private $useSSL;
    private $testMode;

    public function __construct(array $params)
    {
        $this->apiKey = $params['serverpassword'];
        $this->apiEndpoint = rtrim($params['configoptions']['Api Endpoint'] ?? $params['serverhostname'], '/');
        $this->useSSL = $params['configoptions']['Use SSL'] ?? true;
        $this->testMode = $params['configoptions']['Test Mode'] ?? false;

        if ($this->testMode) {
            $this->apiEndpoint = 'https://sandbox.yourprovider.com/v1';
        }
    }

    /**
     * Make API request.
     *
     * @param string $endpoint
     * @param string $method
     * @param array $data
     * @return array
     * @throws \Exception
     */
    private function request(string $endpoint, string $method = 'GET', array $data = []): array
    {
        $url = $this->apiEndpoint . '/' . ltrim($endpoint, '/');

        $ch = curl_init();

        $headers = [
            'Authorization: Bearer ' . $this->apiKey,
            'Content-Type: application/json',
            'Accept: application/json',
        ];

        curl_setopt_array($ch, [
            CURLOPT_URL => $url,
            CURLOPT_RETURNTRANSFER => true,
            CURLOPT_TIMEOUT => 30,
            CURLOPT_HTTPHEADER => $headers,
            CURLOPT_SSL_VERIFYPEER => true,
            CURLOPT_SSL_VERIFYHOST => 2,
        ]);

        if ($method !== 'GET') {
            curl_setopt($ch, CURLOPT_CUSTOMREQUEST, $method);
            if (!empty($data)) {
                curl_setopt($ch, CURLOPT_POSTFIELDS, json_encode($data));
            }
        }

        $response = curl_exec($ch);
        $httpCode = curl_getinfo($ch, CURLINFO_HTTP_CODE);
        $error = curl_error($ch);

        curl_close($ch);

        if ($error) {
            throw new \Exception("cURL Error: " . $error);
        }

        $decoded = json_decode($response, true);

        if ($httpCode >= 400) {
            $message = $decoded['message'] ?? $decoded['error'] ?? 'Unknown API error';
            throw new \Exception("API Error ($httpCode): $message");
        }

        return $decoded;
    }

    /**
     * Test connection to API.
     *
     * @return array
     */
    public function testConnection(): array
    {
        return $this->request('ping');
    }

    /**
     * Create hosting account.
     *
     * @param array $package
     * @return array
     */
    public function createAccount(array $package): array
    {
        return $this->request('accounts', 'POST', $package);
    }

    /**
     * Terminate account.
     *
     * @param int $accountId
     * @return array
     */
    public function terminateAccount(int $accountId): array
    {
        return $this->request("accounts/{$accountId}", 'DELETE');
    }

    /**
     * Suspend account.
     *
     * @param int $accountId
     * @return array
     */
    public function suspendAccount(int $accountId): array
    {
        return $this->request("accounts/{$accountId}/suspend", 'POST');
    }

    /**
     * Unsuspend account.
     *
     * @param int $accountId
     * @return array
     */
    public function unsuspendAccount(int $accountId): array
    {
        return $this->request("accounts/{$accountId}/unsuspend", 'POST');
    }

    /**
     * Change account password.
     *
     * @param int $accountId
     * @param string $password
     * @return array
     */
    public function changePassword(int $accountId, string $password): array
    {
        return $this->request("accounts/{$accountId}/password", 'PUT', [
            'password' => $password,
        ]);
    }

    /**
     * Change account package.
     *
     * @param int $accountId
     * @param string $package
     * @return array
     */
    public function changePackage(int $accountId, string $package): array
    {
        return $this->request("accounts/{$accountId}/package", 'PUT', [
            'package' => $package,
        ]);
    }

    /**
     * Get account details.
     *
     * @param int $accountId
     * @return array
     */
    public function getAccountDetails(int $accountId): array
    {
        return $this->request("accounts/{$accountId}");
    }

    /**
     * Get usage metrics.
     *
     * @param int $accountId
     * @return array
     */
    public function getUsageMetrics(int $accountId): array
    {
        return $this->request("accounts/{$accountId}/metrics");
    }
}
```

### Step 4: Create Language File

Create `lang/english.php`:

```php
<?php

$_LANG = [
    'yourmodulename' => 'Your Module Name',
    'yourmodule_test_connection' => 'Test Connection',
    'yourmodule_connection_success' => 'Connection test successful!',
    'yourmodule_connection_failed' => 'Connection test failed: ',
];
```

### Step 5: Register Server Group

Navigate to WHMCS Admin > System > Servers > Add New Server and configure:
- Server Name
- Hostname
- IP Address
- Assigned Module: Your Module Name

### Step 6: Create Product/Configurable Options

1. Go to Setup > Products/Services > Products/Services
2. Create new product or edit existing
3. Set Module to "Your Module Name"
4. Configure module settings (configoptions)

## Expected Outcomes

- Server module appears in WHMCS server list
- Test connection button works from server configuration
- Products using the module can be created, suspended, terminated
- Password changes sync to the provider
- Usage metrics display correctly

## Testing Checklist

- [ ] Module files created with correct structure
- [ ] API client connects successfully to provider
- [ ] Create account provisions service correctly
- [ ] Suspend/Unsuspend works as expected
- [ ] Terminate removes account from provider
- [ ] Password change syncs correctly
- [ ] Package change updates account
- [ ] Error handling catches and reports failures
- [ ] Test connection button returns success/failure
- [ ] Usage metrics display in client area
- [ ] Logging captures all operations
