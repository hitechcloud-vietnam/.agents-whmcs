# WHMCS Provisioning Module API

Complete reference for server/provisioning module development in WHMCS.

## Overview

Provisioning modules control the creation, management, and termination of hosting accounts on remote servers.

## Module Structure

### Required Files

```
modules/servers/YourModule/
├── yourmodule.php          # Main module file
├── includes/
│   └── functions.php       # Optional helper functions
└── lib/
    └── Client.php          # Optional library class
```

### Basic Module Template

```php
<?php
/**
 * WHMCS Provisioning Module
 * 
 * Module: YourModule
 * Version: 1.0
 */

if (!defined("WHMCS")) {
    die("This file cannot be accessed directly");
}

/**
 * Define module metadata
 */
function yourmodule_MetaData()
{
    return [
        'DisplayName' => 'Your Module Name',
        'APIVersion' => '1.1',
        'RequiresServer' => true,
        'DefaultNonSSLPort' => '2083',
        'DefaultSSLPort' => '2083',
        'ServiceSingleSignOn' => true,
        'AdminSingleSignOn' => true,
    ];
}

/**
 * Define configuration options
 */
function yourmodule_ConfigArray()
{
    return [
        'FriendlyName' => [
            'Type' => 'System',
            'Value' => 'Your Module',
        ],
        'ServerIP' => [
            'FriendlyName' => 'Server IP',
            'Type' => 'text',
            'Size' => '30',
            'Description' => 'Enter the server IP address',
        ],
        'ServerPort' => [
            'FriendlyName' => 'API Port',
            'Type' => 'text',
            'Size' => '10',
            'Default' => '2087',
        ],
        'Username' => [
            'FriendlyName' => 'API Username',
            'Type' => 'text',
            'Size' => '30',
        ],
        'Password' => [
            'FriendlyName' => 'API Password',
            'Type' => 'password',
            'Size' => '30',
        ],
        'AccessHash' => [
            'FriendlyName' => 'Access Hash',
            'Type' => 'textarea',
            'Rows' => '5',
            'Description' => 'Optional access hash for authentication',
        ],
        'Secure' => [
            'FriendlyName' => 'Use SSL',
            'Type' => 'yesno',
            'Description' => 'Use SSL connection for API',
        ],
    ];
}
```

## Module Functions

### CreateAccount

Creates a new hosting account.

```php
/**
 * Create account callback
 * 
 * @param array $params Module parameters
 * @return array Result
 */
function yourmodule_CreateAccount(array $params)
{
    try {
        // Validate parameters
        if (empty($params['domain'])) {
            return ['error' => 'Domain is required'];
        }
        
        // Connect to server
        $api = new YourModuleAPI($params);
        
        // Create account
        $result = $api->createAccount([
            'domain' => $params['domain'],
            'username' => $params['username'],
            'password' => decrypt($params['password']),
            'email' => $params['clientsdetails']['email'],
            'plan' => $params['configoption1'],
        ]);
        
        if ($result['success']) {
            return [
                'success' => true,
                'accountid' => $params['serviceid'],
            ];
        }
        
        return ['error' => $result['message'] ?? 'Account creation failed'];
        
    } catch (Exception $e) {
        return ['error' => $e->getMessage()];
    }
}
```

### SuspendAccount

Suspends a hosting account.

```php
/**
 * Suspend account callback
 * 
 * @param array $params Module parameters
 * @return array Result
 */
function yourmodule_SuspendAccount(array $params)
{
    try {
        $api = new YourModuleAPI($params);
        
        $result = $api->suspendAccount([
            'username' => $params['username'],
            'reason' => $params['suspendreason'] ?? 'Account suspended',
        ]);
        
        return ['success' => $result['success'] ?? true];
        
    } catch (Exception $e) {
        return ['error' => $e->getMessage()];
    }
}
```

### UnsuspendAccount

Unsuspends a hosting account.

```php
/**
 * Unsuspend account callback
 * 
 * @param array $params Module parameters
 * @return array Result
 */
function yourmodule_UnsuspendAccount(array $params)
{
    try {
        $api = new YourModuleAPI($params);
        
        $result = $api->unsuspendAccount([
            'username' => $params['username'],
        ]);
        
        return ['success' => $result['success'] ?? true];
        
    } catch (Exception $e) {
        return ['error' => $e->getMessage()];
    }
}
```

### TerminateAccount

Terminates a hosting account.

```php
/**
 * Terminate account callback
 * 
 * @param array $params Module parameters
 * @return array Result
 */
function yourmodule_TerminateAccount(array $params)
{
    try {
        $api = new YourModuleAPI($params);
        
        $result = $api->terminateAccount([
            'username' => $params['username'],
        ]);
        
        return ['success' => $result['success'] ?? true];
        
    } catch (Exception $e) {
        return ['error' => $e->getMessage()];
    }
}
```

### ChangePassword

Changes the account password.

```php
/**
 * Change password callback
 * 
 * @param array $params Module parameters
 * @return array Result
 */
function yourmodule_ChangePassword(array $params)
{
    try {
        $api = new YourModuleAPI($params);
        
        $result = $api->changePassword([
            'username' => $params['username'],
            'password' => decrypt($params['password']),
        ]);
        
        return ['success' => $result['success'] ?? true];
        
    } catch (Exception $e) {
        return ['error' => $e->getMessage()];
    }
}
```

### ChangePackage

Changes the hosting package/plan.

```php
/**
 * Change package callback
 * 
 * @param array $params Module parameters
 * @return array Result
 */
function yourmodule_ChangePackage(array $params)
{
    try {
        $api = new YourModuleAPI($params);
        
        $result = $api->changePackage([
            'username' => $params['username'],
            'plan' => $params['configoption1'],
        ]);
        
        return ['success' => $result['success'] ?? true];
        
    } catch (Exception $e) {
        return ['error' => $e->getMessage()];
    }
}
```

### ServiceSingleSignOn

Provides SSO login to the service.

```php
/**
 * Single sign-on callback
 * 
 * @param array $params Module parameters
 * @return array Result
 */
function yourmodule_ServiceSingleSignOn(array $params)
{
    try {
        $api = new YourModuleAPI($params);
        
        $result = $api->createSession([
            'username' => $params['username'],
            'password' => decrypt($params['password']),
        ]);
        
        return [
            'success' => true,
            'redirecturl' => $result['session_url'],
        ];
        
    } catch (Exception $e) {
        return ['error' => $e->getMessage()];
    }
}
```

### AdminSingleSignOn

Provides SSO login to admin panel.

```php
/**
 * Admin single sign-on callback
 * 
 * @param array $params Module parameters
 * @return array Result
 */
function yourmodule_AdminSingleSignOn(array $params)
{
    try {
        $api = new YourModuleAPI($params);
        
        $result = $api->createAdminSession([
            'username' => $params['username'],
            'password' => decrypt($params['password']),
        ]);
        
        return [
            'success' => true,
            'redirecturl' => $result['admin_url'],
        ];
        
    } catch (Exception $e) {
        return ['error' => $e->getMessage()];
    }
}
```

### UsageUpdate

Reports service usage stats.

```php
/**
 * Usage update callback
 * 
 * @param array $params Module parameters
 * @return array Result
 */
function yourmodule_UsageUpdate(array $params)
{
    try {
        $api = new YourModuleAPI($params);
        
        $usage = $api->getUsage([
            'username' => $params['username'],
        ]);
        
        return [
            'success' => true,
            'diskusage' => $usage['disk'],
            'disklimit' => $usage['disk_limit'],
            'bwusage' => $usage['bandwidth'],
            'bwlimit' => $usage['bw_limit'],
        ];
        
    } catch (Exception $e) {
        return ['error' => $e->getMessage()];
    }
}
```

## API Client Example

```php
<?php
/**
 * API Client for YourModule
 */
class YourModuleAPI
{
    private $params;
    private $baseUrl;
    private $apiKey;
    
    public function __construct(array $params)
    {
        $this->params = $params;
        
        $secure = !empty($params['configoptions']['Secure']) || 
                  !empty($params['configoption6']);
        $protocol = $secure ? 'https' : 'http';
        $port = $params['configoption3'] ?? '2087';
        
        $this->baseUrl = "{$protocol}://{$params['serverip']}:{$port}";
        $this->apiKey = $params['configoption2'];
    }
    
    public function request(string $endpoint, array $data = []): array
    {
        $ch = curl_init();
        
        curl_setopt($ch, CURLOPT_URL, $this->baseUrl . $endpoint);
        curl_setopt($ch, CURLOPT_POST, true);
        curl_setopt($ch, CURLOPT_POSTFIELDS, http_build_query($data));
        curl_setopt($ch, CURLOPT_RETURNTRANSFER, true);
        curl_setopt($ch, CURLOPT_TIMEOUT, 60);
        curl_setopt($ch, CURLOPT_SSL_VERIFYPEER, false);
        curl_setopt($ch, CURLOPT_HTTPHEADER, [
            'Authorization: Bearer ' . $this->apiKey
        ]);
        
        $response = curl_exec($ch);
        $error = curl_error($ch);
        curl_close($ch);
        
        if ($error) {
            throw new Exception("API Error: {$error}");
        }
        
        return json_decode($response, true) ?? [];
    }
    
    public function createAccount(array $data): array
    {
        return $this->request('/api/account/create', $data);
    }
    
    public function suspendAccount(array $data): array
    {
        return $this->request('/api/account/suspend', $data);
    }
    
    public function unsuspendAccount(array $data): array
    {
        return $this->request('/api/account/unsuspend', $data);
    }
    
    public function terminateAccount(array $data): array
    {
        return $this->request('/api/account/terminate', $data);
    }
    
    public function changePassword(array $data): array
    {
        return $this->request('/api/account/password', $data);
    }
    
    public function changePackage(array $data): array
    {
        return $this->request('/api/account/package', $data);
    }
    
    public function getUsage(array $data): array
    {
        return $this->request('/api/account/usage', $data);
    }
}
```

## Testing Your Module

```php
/**
 * Test module connectivity
 */
function yourmodule_TestConnection(array $params): array
{
    try {
        $api = new YourModuleAPI($params);
        
        $result = $api->request('/api/system/ping');
        
        if ($result['status'] === 'ok') {
            return [
                'success' => true,
                'message' => 'Connection successful',
            ];
        }
        
        return ['error' => 'Connection test failed'];
        
    } catch (Exception $e) {
        return ['error' => $e->getMessage()];
    }
}
```

## Best Practices

1. **Always return arrays** - Never throw uncaught exceptions
2. **Log module calls** - Use logModuleCall for debugging
3. **Handle timeouts** - Set appropriate timeouts for API calls
4. **Validate credentials** - Test connection before saving config
5. **Encrypt sensitive data** - Never log passwords
6. **Use transactions** - Handle partial failures gracefully

## Related Documentation

- [whmcs-module-parameters.md](whmcs-module-parameters.md)
- [whmcs-module-error-handling.md](whmcs-module-error-handling.md)
- [whmcs-module-lifecycle.md](whmcs-module-lifecycle.md)