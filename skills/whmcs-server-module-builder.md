# WHMCS Server Module Builder

## Concept

A WHMCS server module (provisioning module) handles communication between WHMCS and your hosting server. It manages account creation, suspension, termination, and other lifecycle operations. Server modules follow a standardized interface defined by WHMCS.

## File Structure

```
/modules/servers/yourmodule/
├── yourmodule.php          # Main module file
├── README.md               # Documentation
├── VERSION.txt             # Version info
└── templates/
    └── client/
        └── overview.tpl    # Client area template
```

## Core Module Structure

```php
<?php
/**
 * Module Name: Your Module
 * Version: 1.0.0
 * Description: Server provisioning module for...
 * Author: Your Name
 * Author URI: https://example.com
 */

if (!defined("WHMCS")) {
    die("This file cannot be accessed directly");
}

// Module configuration
function yourmodule_MetaData()
{
    return [
        'DisplayName' => 'Your Module',
        'APIVersion' => '1.1',
        'RequiresServer' => true,
        'DefaultNonSSLPort' => 'port',
        'DefaultSSLPort' => 'ssl_port',
    ];
}

// Create account
function yourmodule_CreateAccount(array $params)
{
    try {
        // Validate parameters
        $username = $params['username'];
        $password = $params['password'];
        $domain = $params['domain'];
        
        // Connect to server
        $server = new YourModuleAPI([
            'host' => $params['serverip'],
            'username' => $params['serverusername'],
            'password' => decrypt($params['serverpassword']),
            'port' => $params['serverport'],
        ]);
        
        // Create account
        $result = $server->createAccount([
            'username' => $username,
            'password' => $password,
            'domain' => $domain,
            'plan' => $params['configoption1'],
        ]);
        
        if (!$result['success']) {
            return [
                'success' => false,
                'error' => $result['error'],
            ];
        }
        
        return ['success' => true];
    } catch (\Exception $e) {
        logActivity("yourmodule_CreateAccount Error: " . $e->getMessage());
        return ['success' => false, 'error' => $e->getMessage()];
    }
}

// Suspend account
function yourmodule_SuspendAccount(array $params)
{
    try {
        $server = new YourModuleAPI($params);
        $server->suspendAccount($params['username']);
        return ['success' => true];
    } catch (\Exception $e) {
        return ['success' => false, 'error' => $e->getMessage()];
    }
}

// Unsuspend account
function yourmodule_UnsuspendAccount(array $params)
{
    try {
        $server = new YourModuleAPI($params);
        $server->unsuspendAccount($params['username']);
        return ['success' => true];
    } catch (\Exception $e) {
        return ['success' => false, 'error' => $e->getMessage()];
    }
}

// Terminate account
function yourmodule_TerminateAccount(array $params)
{
    try {
        $server = new YourModuleAPI($params);
        $server->deleteAccount($params['username']);
        return ['success' => true];
    } catch (\Exception $e) {
        return ['success' => false, 'error' => $e->getMessage()];
    }
}

// Change password
function yourmodule_ChangePassword(array $params)
{
    try {
        $server = new YourModuleAPI($params);
        $server->changePassword($params['username'], $params['password']);
        return ['success' => true];
    } catch (\Exception $e) {
        return ['success' => false, 'error' => $e->getMessage()];
    }
}

// Change package
function yourmodule_ChangePackage(array $params)
{
    try {
        $server = new YourModuleAPI($params);
        $server->changePlan($params['username'], $params['configoption1']);
        return ['success' => true];
    } catch (\Exception $e) {
        return ['success' => false, 'error' => $e->getMessage()];
    }
}

// Client area (single sign-on)
function yourmodule_ClientArea(array $params)
{
    return [
        'templatefile' => 'client/overview',
        'vars' => [
            'serverUrl' => $params['serverurl'],
            'username' => $params['username'],
        ],
    ];
}

// Admin area button
function yourmodule_AdminCustomButtonArray()
{
    return [
        'Reboot Server' => 'reboot',
    ];
}

// Reboot function
function yourmodule_Reboot(array $params)
{
    try {
        $server = new YourModuleAPI($params);
        $server->reboot($params['username']);
        return ['success' => true];
    } catch (\Exception $e) {
        return ['success' => false, 'error' => $e->getMessage()];
    }
}

// Usage update (for billing)
function yourmodule_UsageUpdate($params)
{
    try {
        $server = new YourModuleAPI($params);
        $usage = $server->getUsage($params['username']);
        
        return [
            'success' => true,
            'metrics' => [
                'disk' => $usage['disk_used'],
                'bw' => $usage['bandwidth_used'],
            ],
        ];
    } catch (\Exception $e) {
        return ['success' => false, 'error' => $e->getMessage()];
    }
}
```

## API Client Class

```php
<?php
namespace WHMCS\Module\Server\YourModule;

class YourModuleAPI
{
    private $host;
    private $port;
    private $apiKey;
    private $socket;
    
    public function __construct(array $config)
    {
        $this->host = $config['host'];
        $this->port = $config['port'] ?? 2087;
        $this->apiKey = $config['api_key'] ?? $config['password'];
    }
    
    public function connect()
    {
        $this->socket = @fsockopen($this->host, $this->port, $errno, $errstr, 30);
        if (!$this->socket) {
            throw new \Exception("Connection failed: $errstr");
        }
        return $this;
    }
    
    public function createAccount(array $data)
    {
        $this->connect();
        
        $payload = json_encode([
            'action' => 'create_account',
            'username' => $data['username'],
            'password' => $data['password'],
            'domain' => $data['domain'],
        ]);
        
        $response = $this->send($payload);
        
        if (isset($response['error'])) {
            throw new \Exception($response['error']);
        }
        
        return $response;
    }
    
    private function send(string $payload)
    {
        fwrite($this->socket, $payload);
        $response = fgets($this->socket);
        fclose($this->socket);
        
        return json_decode($response, true);
    }
}
```

## Configuration Options

```php
function yourmodule_ConfigArray()
{
    return [
        'FriendlyName' => [
            'Type' => 'System',
            'Value' => 'Your Module',
        ],
        'api_key' => [
            'Type' => 'password',
            'FriendlyName' => 'API Key',
            'Description' => 'Your API key from the control panel',
        ],
        'default_plan' => [
            'Type' => 'dropdown',
            'FriendlyName' => 'Default Plan',
            'Options' => [
                'starter' => 'Starter',
                'professional' => 'Professional',
                'enterprise' => 'Enterprise',
            ],
        ],
    ];
}
```

## Testing

```php
// Test module via admin
function yourmodule_TestConnection(array $params)
{
    try {
        $server = new YourModuleAPI($params);
        $result = $server->testConnection();
        
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
```

## Step-by-Step Implementation

1. Create module directory in `/modules/servers/yourmodule/`
2. Create main module file with all required functions
3. Implement API client class for server communication
4. Add configuration options in `ConfigArray()`
5. Create client area templates
6. Add module metadata
7. Test in WHMCS admin area
8. Verify all functions work correctly

## Examples from Real Implementations

### cPanel Module Pattern
```php
// cPanel uses XML API
function cpanel_CreateAccount(array $params)
{
    $cpanel = new \StdClass();
    $cpanel->host = $params['serverip'];
    $cpanel->port = 2083;
    $cpanel->user = $params['serverusername'];
    $cpanel->pass = decrypt($params['serverpassword']);
    
    return localapi($cpanel, 'createacct', [
        'username' => $params['username'],
        'password' => $params['password'],
        'domain' => $params['domain'],
    ]);
}
```

### Plesk Module Pattern
```php
// Plesk uses REST API
function plesk_CreateAccount(array $params)
{
    $client = new \GuzzleHttp\Client([
        'base_uri' => 'https://' . $params['serverip'] . ':8443/',
        'verify' => false,
    ]);
    
    $response = $client->post('api/v2/customers', [
        'auth' => [$params['serverusername'], decrypt($params['serverpassword'])],
        'json' => [
            'name' => $params['clientsdetails']['fullname'],
            'login' => $params['username'],
        ],
    ]);
    
    return ['success' => true];
}
```

## Implementation Checklist

- [ ] Create module directory structure
- [ ] Implement MetaData function
- [ ] Implement CreateAccount function
- [ ] Implement SuspendAccount function
- [ ] Implement UnsuspendAccount function
- [ ] Implement TerminateAccount function
- [ ] Implement ChangePassword function
- [ ] Implement ChangePackage function
- [ ] Add configuration options
- [ ] Create API client class
- [ ] Add error handling and logging
- [ ] Test connection function
- [ ] Client area template (if needed)
- [ ] Admin custom buttons (if needed)
- [ ] Test in staging environment
- [ ] Document installation instructions