# WHMCS Provisioning Module Development

## Overview

Provisioning modules (server modules) manage the automated creation, suspension, and termination of hosting accounts on external servers.

## Module Structure

```
modules/servers/yourmodule/
├── yourmodule.php          # Main module file
├── includes/
│   └── api_client.php      # API client class
├── templates/
│   └── clientarea.tpl      # Client area template
└── lang/
    └── english.php         # Language file
```

## Required Functions

### MetaData

```php
<?php
function yourmodule_MetaData(): array
{
    return [
        'DisplayName' => 'Provider Name',
        'APIVersion' => '1.1',
        'RequiresServer' => true,
        'DefaultNonSSLPort' => 443,
        'NonSSLPort' => 443,
        'Parameters' => [
            'server_username',
            'server_password',
            'server_access_hash',
        ],
    ];
}
```

### ConfigOptions

```php
<?php
function yourmodule_ConfigOptions(array $params): array
{
    $config = [
        'Plan' => [
            'Type' => 'dropdown',
            'Options' => [
                'starter' => 'Starter - 1 CPU, 1GB RAM',
                'basic' => 'Basic - 2 CPU, 2GB RAM',
                'professional' => 'Professional - 4 CPU, 4GB RAM',
            ],
            'Default' => 'starter',
            'Description' => 'Server Plan',
        ],
        'Operating System' => [
            'Type' => 'dropdown',
            'Options' => 'Ubuntu 22.04,Ubuntu 20.04,CentOS 8,Debian 11',
            'Default' => 'Ubuntu 22.04',
        ],
        'Backup Options' => [
            'Type' => 'yesno',
            'Default' => 'yes',
            'Description' => 'Enable automated backups',
        ],
        'Custom Variable' => [
            'Type' => 'text',
            'Size' => '20',
            'Default' => '',
            'Description' => 'Custom setting description',
        ],
    ];
    
    return $config;
}
```

## Lifecycle Functions

### CreateAccount

```php
<?php
function yourmodule_CreateAccount(array $params): string
{
    try {
        $api = new ProviderApiClient($params);
        
        // Prepare server creation parameters
        $serverParams = [
            'hostname' => $params['domain'],
            'plan' => $params['configoption1'],
            'os' => $params['configoption2'],
            'backups' => $params['configoption3'] === 'on',
            'username' => $params['username'],
            'password' => $params['password'],
            'email' => $params['clientsdetails']['email'],
        ];
        
        // Create server
        $result = $api->createServer($serverParams);
        
        if (!$result['success']) {
            return 'Error: ' . ($result['error'] ?? 'Unknown error');
        }
        
        // Log successful creation
        logModuleCall(
            'yourmodule',
            'CreateAccount',
            $params,
            $result,
            $result
        );
        
        return 'success';
        
    } catch (Exception $e) {
        logModuleCall(
            'yourmodule',
            'CreateAccount',
            $params,
            [],
            $e->getMessage()
        );
        
        return 'Error: ' . $e->getMessage();
    }
}
```

### SuspendAccount

```php
<?php
function yourmodule_SuspendAccount(array $params): string
{
    try {
        $api = new ProviderApiClient($params);
        
        $result = $api->suspendServer($params['domain']);
        
        if (!$result['success']) {
            return 'Error: ' . ($result['error'] ?? 'Failed to suspend');
        }
        
        logModuleCall(
            'yourmodule',
            'SuspendAccount',
            $params,
            $result,
            $result
        );
        
        return 'success';
        
    } catch (Exception $e) {
        logModuleCall(
            'yourmodule',
            'SuspendAccount',
            $params,
            [],
            $e->getMessage()
        );
        
        return 'Error: ' . $e->getMessage();
    }
}
```

### UnsuspendAccount

```php
<?php
function yourmodule_UnsuspendAccount(array $params): string
{
    try {
        $api = new ProviderApiClient($params);
        
        $result = $api->unsuspendServer($params['domain']);
        
        if (!$result['success']) {
            return 'Error: ' . ($result['error'] ?? 'Failed to unsuspend');
        }
        
        logModuleCall(
            'yourmodule',
            'UnsuspendAccount',
            $params,
            $result,
            $result
        );
        
        return 'success';
        
    } catch (Exception $e) {
        return 'Error: ' . $e->getMessage();
    }
}
```

### TerminateAccount

```php
<?php
function yourmodule_TerminateAccount(array $params): string
{
    try {
        $api = new ProviderApiClient($params);
        
        // Create backup before termination
        $backupResult = $api->createBackup($params['domain'], [
            'type' => 'final',
            'retention' => 7,
        ]);
        
        // Terminate server
        $result = $api->terminateServer($params['domain']);
        
        if (!$result['success']) {
            return 'Error: ' . ($result['error'] ?? 'Failed to terminate');
        }
        
        logModuleCall(
            'yourmodule',
            'TerminateAccount',
            $params,
            $result,
            $result
        );
        
        return 'success';
        
    } catch (Exception $e) {
        return 'Error: ' . $e->getMessage();
    }
}
```

### ChangePassword

```php
<?php
function yourmodule_ChangePassword(array $params): string
{
    try {
        $api = new ProviderApiClient($params);
        
        $result = $api->changeRootPassword($params['domain'], $params['password']);
        
        if (!$result['success']) {
            return 'Error: ' . ($result['error'] ?? 'Failed to change password');
        }
        
        return 'success';
        
    } catch (Exception $e) {
        return 'Error: ' . $e->getMessage();
    }
}
```

### ChangePackage

```php
<?php
function yourmodule_ChangePackage(array $params): string
{
    try {
        $api = new ProviderApiClient($params);
        
        // Upgrade/downgrade server resources
        $result = $api->resizeServer($params['domain'], [
            'plan' => $params['configoption1'],
            'backup_enabled' => $params['configoption3'] === 'on',
        ]);
        
        if (!$result['success']) {
            return 'Error: ' . ($result['error'] ?? 'Failed to change package');
        }
        
        return 'success';
        
    } catch (Exception $e) {
        return 'Error: ' . $e->getMessage();
    }
}
```

### TestConnection

```php
<?php
function yourmodule_TestConnection(array $params): array
{
    try {
        $api = new ProviderApiClient($params);
        
        $result = $api->testConnection();
        
        if ($result['success']) {
            return [
                'success' => true,
                'error' => '',
            ];
        }
        
        return [
            'success' => false,
            'error' => $result['error'] ?? 'Connection failed',
        ];
        
    } catch (Exception $e) {
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
class ProviderApiClient
{
    private string $apiKey;
    private string $apiUrl;
    private int $serverId;
    
    public function __construct(array $params)
    {
        $this->apiKey = $params['serveraccesshash'];
        $this->apiUrl = 'https://api.provider.com/v1';
        $this->serverId = $params['serverid'];
    }
    
    public function createServer(array $data): array
    {
        return $this->request('POST', '/servers', [
            'name' => $data['hostname'],
            'flavor' => $data['plan'],
            'image' => $data['os'],
            'backups' => $data['backups'],
            'user_data' => $this->generateUserData($data),
        ]);
    }
    
    public function suspendServer(string $hostname): array
    {
        return $this->request('POST', "/servers/{$hostname}/suspend");
    }
    
    public function unsuspendServer(string $hostname): array
    {
        return $this->request('POST', "/servers/{$hostname}/unsuspend");
    }
    
    public function terminateServer(string $hostname): array
    {
        return $this->request('DELETE', "/servers/{$hostname}");
    }
    
    public function changeRootPassword(string $hostname, string $password): array
    {
        return $this->request('POST', "/servers/{$hostname}/password", [
            'password' => $password,
        ]);
    }
    
    public function resizeServer(string $hostname, array $data): array
    {
        return $this->request('POST', "/servers/{$hostname}/resize", $data);
    }
    
    public function getServerStats(string $hostname): array
    {
        return $this->request('GET', "/servers/{$hostname}/stats");
    }
    
    public function testConnection(): array
    {
        return $this->request('GET', '/ping');
    }
    
    private function request(string $method, string $endpoint, array $data = []): array
    {
        $ch = curl_init($this->apiUrl . $endpoint);
        
        $headers = [
            'Authorization: Bearer ' . $this->apiKey,
            'Content-Type: application/json',
        ];
        
        curl_setopt_array($ch, [
            CURLOPT_RETURNTRANSFER => true,
            CURLOPT_HTTPHEADER => $headers,
            CURLOPT_TIMEOUT => 60,
        ]);
        
        if ($method === 'POST') {
            curl_setopt($ch, CURLOPT_POST, true);
            curl_setopt($ch, CURLOPT_POSTFIELDS, json_encode($data));
        } elseif ($method === 'DELETE') {
            curl_setopt($ch, CURLOPT_CUSTOMREQUEST, 'DELETE');
        }
        
        $response = curl_exec($ch);
        $httpCode = curl_getinfo($ch, CURLINFO_HTTP_CODE);
        $error = curl_error($ch);
        curl_close($ch);
        
        if ($error) {
            return [
                'success' => false,
                'error' => $error,
            ];
        }
        
        $result = json_decode($response, true);
        
        return [
            'success' => $httpCode >= 200 && $httpCode < 300,
            'data' => $result,
            'error' => $result['error'] ?? null,
        ];
    }
    
    private function generateUserData(array $data): string
    {
        return "#!/bin/bash\n"
            . "echo '{$data['username']}:{$data['password']}' | chpasswd\n"
            . "apt-get update && apt-get install -y curl\n";
    }
}
```

## Client Area

```php
<?php
function yourmodule_ClientArea(array $params): array
{
    $api = new ProviderApiClient($params);
    $stats = $api->getServerStats($params['domain']);
    
    return [
        'pagetitle' => 'Server Management',
        'templatefile' => 'templates/clientarea',
        'vars' => [
            'serverStats' => $stats['data'] ?? [],
            'serverStatus' => $stats['success'] ? 'online' : 'offline',
        ],
    ];
}
```

## Best Practices

1. **Always log operations** - Use logModuleCall for debugging
2. **Handle timeouts** - Set appropriate CURL timeouts
3. **Return error strings** - Return 'Error: message' on failure
4. **Test thoroughly** - Test all lifecycle operations
5. **Implement retry logic** - Handle transient failures
6. **Secure credentials** - Never log sensitive data

## Related Documentation

- [WHMCS Module Security](/docs/whmcs-module-security.md)
- [WHMCS Module API](/docs/whmcs-module-api.md)