# WHMCS Provisioning Module From Scratch

## Description
Create a custom server provisioning/suspending module for WHMCS from scratch.

## Prerequisites
- WHMCS installation
- Server API documentation
- PHP 8.1+
- SSH/API access to server

## Module Structure
```
modules/servers/
└── clicodes_provisioning/
    ├── clicodes_provisioning.php    # Main module
    ├── API/
    │   └── Client.php               # API client
    ├── functions.php                # Helper functions
    ├── LICENSE
    └── README.md
```

## Steps

### Step 1: Create Module Directory
```bash
mkdir -p /var/www/whmcs/modules/servers/clicodes_provisioning
mkdir -p /var/www/whmcs/modules/servers/clicodes_provisioning/API
```

### Step 2: Create Main Module File
```php
<?php
/**
 * WHMCS Server Provisioning Module - CLICodes Provisioning
 *
 * @copyright Copyright (c) 2024 Your Company
 * @version 1.0.0
 */

if (!defined("WHMCS")) {
    die("This file cannot be accessed directly");
}

/**
 * Define module configuration
 */
function clicodes_provisioning_MetaData()
{
    return [
        'DisplayName' => 'CLICodes Provisioning',
        'APIVersion' => '1.1',
        'RequiresServer' => true,
        'AutoSetup' => false,
        'DefaultNonSSLPort' => 80,
        'DefaultSSLPort' => 443,
    ];
}

/**
 * Get server configuration
 */
function clicodes_provisioning_getConfigArray($params)
{
    return [
        'FriendlyName' => [
            'Type' => 'System',
            'Value' => 'CLICodes Server',
        ],
        'Description' => [
            'Type' => 'System',
            'Value' => 'Connect to CLICodes provisioning API.',
        ],
        'apiKey' => [
            'Type' => 'text',
            'Label' => 'API Key',
            'Description' => 'Your CLICodes API key.',
        ],
        'apiSecret' => [
            'Type' => 'password',
            'Label' => 'API Secret',
            'Description' => 'Your CLICodes API secret.',
        ],
        'serverHost' => [
            'Type' => 'text',
            'Label' => 'Server Hostname',
            'Description' => 'Primary server hostname.',
        ],
        'serverPort' => [
            'Type' => 'text',
            'Label' => 'API Port',
            'Description' => 'API port (default: 443)',
            'Default' => '443',
        ],
        'useSSL' => [
            'Type' => 'yesno',
            'Label' => 'Use SSL',
            'Description' => 'Use SSL for API connection.',
            'Default' => 'on',
        ],
        'packagePrefix' => [
            'Type' => 'text',
            'Label' => 'Package Prefix',
            'Description' => 'Prefix for package names in API.',
            'Default' => 'whmcs_',
        ],
    ];
}

/**
 * Create account
 */
function clicodes_provisioning_CreateAccount($params)
{
    $api = new CLICodesProvisioningAPI($params);
    
    try {
        // Prepare provisioning data
        $data = [
            'username' => $params['username'],
            'password' => $params['password'],
            'package' => $params['configoption1'], // Package ID
            'domain' => $params['domain'],
            'email' => $params['clientsdetails']['email'],
            'first_name' => $params['clientsdetails']['firstname'],
            'last_name' => $params['clientsdetails']['lastname'],
            'ip_address' => $params['serverip'],
        ];
        
        $response = $api->request('POST', '/accounts', $data);
        
        if ($response['success']) {
            return [
                'success' => true,
                'accountid' => $response['account_id'],
                'username' => $params['username'],
                'password' => $params['password'],
            ];
        }
        
        return [
            'success' => false,
            'error' => $response['message'] ?? 'Creation failed',
        ];
        
    } catch (Exception $e) {
        return [
            'success' => false,
            'error' => $e->getMessage(),
        ];
    }
}

/**
 * Terminate account
 */
function clicodes_provisioning_TerminateAccount($params)
{
    $api = new CLICodesProvisioningAPI($params);
    
    try {
        $response = $api->request('DELETE', '/accounts/' . $params['username']);
        
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
 * Suspend account
 */
function clicodes_provisioning_SuspendAccount($params)
{
    $api = new CLICodesProvisioningAPI($params);
    
    try {
        $response = $api->request('POST', '/accounts/' . $params['username'] . '/suspend', [
            'reason' => $params['suspendreason'] ?? 'Payment issue',
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
 * Unsuspend account
 */
function clicodes_provisioning_UnsuspendAccount($params)
{
    $api = new CLICodesProvisioningAPI($params);
    
    try {
        $response = $api->request('POST', '/accounts/' . $params['username'] . '/unsuspend');
        
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
 * Change password
 */
function clicodes_provisioning_ChangePassword($params)
{
    $api = new CLICodesProvisioningAPI($params);
    
    try {
        $response = $api->request('PUT', '/accounts/' . $params['username'] . '/password', [
            'password' => $params['password'],
        ]);
        
        if ($response['success']) {
            return [
                'success' => true,
            ];
        }
        
        return [
            'success' => false,
            'error' => $response['message'] ?? 'Password change failed',
        ];
        
    } catch (Exception $e) {
        return [
            'success' => false,
            'error' => $e->getMessage(),
        ];
    }
}

/**
 * Change package
 */
function clicodes_provisioning_ChangePackage($params)
{
    $api = new CLICodesProvisioningAPI($params);
    
    try {
        $response = $api->request('PUT', '/accounts/' . $params['username'] . '/package', [
            'package' => $params['configoption1'],
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
 * Get usage statistics
 */
function clicodes_provisioning_GetUsage($params)
{
    $api = new CLICodesProvisioningAPI($params);
    
    try {
        $response = $api->request('GET', '/accounts/' . $params['username'] . '/usage');
        
        if ($response['success']) {
            return [
                'success' => true,
                'diskusage' => $response['disk_usage'] ?? 0,
                'disklimit' => $response['disk_limit'] ?? 0,
                'bwusage' => $response['bandwidth_usage'] ?? 0,
                'bwlimit' => $response['bandwidth_limit'] ?? 0,
                'lastupdate' => $response['last_updated'] ?? date('Y-m-d H:i:s'),
            ];
        }
        
        return [
            'success' => false,
            'error' => $response['message'] ?? 'Failed to get usage',
        ];
        
    } catch (Exception $e) {
        return [
            'success' => false,
            'error' => $e->getMessage(),
        ];
    }
}

/**
 * Server statics
 */
function clicodes_provisioning_ServerStats($params)
{
    $api = new CLICodesProvisioningAPI($params);
    
    try {
        $response = $api->request('GET', '/server/stats');
        
        if ($response['success']) {
            $html = '<div class="server-stats">';
            $html .= '<h4>Server Status</h4>';
            $html .= '<p>CPU: ' . ($response['cpu_usage'] ?? 'N/A') . '%</p>';
            $html .= '<p>Memory: ' . ($response['memory_usage'] ?? 'N/A') . '%</p>';
            $html .= '<p>Disk: ' . ($response['disk_usage'] ?? 'N/A') . '%</p>';
            $html .= '<p>Uptime: ' . ($response['uptime'] ?? 'N/A') . '</p>';
            $html .= '</div>';
            
            return $html;
        }
        
        return '<p>Unable to load server statistics.</p>';
        
    } catch (Exception $e) {
        return '<p>Error: ' . $e->getMessage() . '</p>';
    }
}

/**
 * Admin custom button
 */
function clicodes_provisioning_AdminCustomButton($params)
{
    $api = new CLICodesProvisioningAPI($params);
    
    $action = $params['action'] ?? '';
    
    try {
        switch ($action) {
            case 'reboot':
                $response = $api->request('POST', '/accounts/' . $params['username'] . '/reboot');
                $message = 'Server rebooted successfully';
                break;
                
            case 'reinstall':
                $response = $api->request('POST', '/accounts/' . $params['username'] . '/reinstall');
                $message = 'Reinstall initiated';
                break;
                
            case 'console':
                $response = $api->request('GET', '/accounts/' . $params['username'] . '/console');
                return 'Console URL: ' . ($response['console_url'] ?? 'N/A');
                
            default:
                return 'Unknown action';
        }
        
        return [
            'success' => $response['success'],
            'message' => $response['message'] ?? $message,
        ];
        
    } catch (Exception $e) {
        return [
            'success' => false,
            'message' => $e->getMessage(),
        ];
    }
}

/**
 * Usage update (called by cron)
 */
function clicodes_provisioning_UsageUpdate($params)
{
    $api = new CLICodesProvisioningAPI($params);
    
    $results = ['success' => 0, 'failed' => 0];
    
    $services = Capsule::table('tblhosting')
        ->where('server', $params['serverid'])
        ->where('domainstatus', 'Active')
        ->get();
    
    foreach ($services as $service) {
        try {
            $response = $api->request('GET', '/accounts/' . $service->username . '/usage');
            
            if ($response['success']) {
                Capsule::table('tblhosting')
                    ->where('id', $service->id)
                    ->update([
                        'diskusage' => $response['disk_usage'] ?? 0,
                        'disklimit' => $response['disk_limit'] ?? 0,
                        'bwusage' => $response['bandwidth_usage'] ?? 0,
                        'bwlimit' => $response['bandwidth_limit'] ?? 0,
                        'lastupdate' => date('Y-m-d H:i:s'),
                    ]);
                
                $results['success']++;
            } else {
                $results['failed']++;
            }
            
        } catch (Exception $e) {
            $results['failed']++;
            logActivity('Usage update failed for ' . $service->username . ': ' . $e->getMessage());
        }
    }
    
    return $results;
}
```

### Step 3: Create API Client
```php
<?php
/**
 * CLICodes Provisioning API Client
 */

if (!defined("WHMCS")) {
    die("This file cannot be accessed directly");
}

class CLICodesProvisioningAPI
{
    private $apiKey;
    private $apiSecret;
    private $serverHost;
    private $serverPort;
    private $useSSL;
    private $baseUrl;
    
    public function __construct($params)
    {
        $this->apiKey = $params['apiKey'] ?? '';
        $this->apiSecret = $params['apiSecret'] ?? '';
        $this->serverHost = $params['serverHost'] ?? '';
        $this->serverPort = $params['serverPort'] ?? 443;
        $this->useSSL = ($params['useSSL'] ?? 'on') === 'on';
        
        $protocol = $this->useSSL ? 'https' : 'http';
        $this->baseUrl = $protocol . '://' . $this->serverHost . ':' . $this->serverPort;
    }
    
    public function request($method, $endpoint, $data = [])
    {
        $url = $this->baseUrl . $endpoint;
        $timestamp = time();
        
        $headers = [
            'Content-Type: application/json',
            'X-API-Key: ' . $this->apiKey,
            'X-Timestamp: ' . $timestamp,
            'X-Signature: ' . $this->generateSignature($method, $endpoint, $data, $timestamp),
        ];
        
        $ch = curl_init();
        
        $options = [
            CURLOPT_URL => $url,
            CURLOPT_RETURNTRANSFER => true,
            CURLOPT_HTTPHEADER => $headers,
            CURLOPT_TIMEOUT => 60,
            CURLOPT_SSL_VERIFYPEER => true,
        ];
        
        if ($method === 'POST') {
            $options[CURLOPT_POST] = true;
            $options[CURLOPT_POSTFIELDS] = json_encode($data);
        } elseif ($method === 'PUT') {
            $options[CURLOPT_CUSTOMREQUEST] = 'PUT';
            $options[CURLOPT_POSTFIELDS] = json_encode($data);
        } elseif ($method === 'DELETE') {
            $options[CURLOPT_CUSTOMREQUEST] = 'DELETE';
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
        
        logModuleCall(
            'clicodes_provisioning',
            $method . ' ' . $endpoint,
            $data,
            $result,
            $httpCode
        );
        
        return $result;
    }
    
    private function generateSignature($method, $endpoint, $data, $timestamp)
    {
        $payload = $method . "\n";
        $payload .= $endpoint . "\n";
        $payload .= json_encode($data) . "\n";
        $payload .= $timestamp . "\n";
        $payload .= $this->apiKey;
        
        return hash_hmac('sha256', $payload, $this->apiSecret);
    }
}
```

### Step 4: Install Module
```bash
# Copy module
cp -r clicodes_provisioning /var/www/whmcs/modules/servers/

# Set permissions
chown -R www-data:www-data /var/www/whmcs/modules/servers/clicodes_provisioning
chmod -R 755 /var/www/whmcs/modules/servers/clicodes_provisioning

# Configure in WHMCS Admin
# Go to: Configuration > Servers
# Add new server with module type "CLICodes Provisioning"
```

## Required Functions
| Function | Required | Description |
|----------|----------|-------------|
| MetaData | Yes | Module metadata |
| getConfigArray | Yes | Configuration |
| CreateAccount | Yes | Create service |
| TerminateAccount | Yes | Terminate service |
| SuspendAccount | Yes | Suspend service |
| UnsuspendAccount | Yes | Unsuspend service |
| ChangePassword | Yes | Update password |
| ChangePackage | Yes | Change package |
| GetUsage | No | Get resource usage |
| ServerStats | No | Show server stats |

## Tags
- provisioning
- server
- module-development
- development