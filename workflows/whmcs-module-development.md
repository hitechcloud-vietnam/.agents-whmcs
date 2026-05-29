# WHMCS Module Development Workflow

## Purpose
Create custom WHMCS modules (server, gateway, registrar)

## Prerequisites
- WHMCS installed
- PHP development experience
- Understanding of WHMCS module structure

## Step 1: Understand Module Types

| Type | Location | Purpose |
|------|----------|---------|
| Server | /modules/servers/ | Provisioning |
| Gateway | /modules/gateways/ | Payments |
| Registrar | /modules/registrars/ | Domain registration |
| Addon | /modules/addons/ | Additional features |
| Widget | /modules/widgets/ | Dashboard widgets |

## Step 2: Create Server Module Directory

```bash
mkdir -p /var/www/whmcs/modules/servers/my_server_module
```

## Step 3: Create Module Definition

```php
<?php
// module.php

if (!defined("WHMCS")) {
    die("Direct access not allowed");
}

function myServerModule_config()
{
    return [
        'name' => 'My Server Module',
        'description' => 'Custom server provisioning module',
        'version' => '1.0.0',
        'author' => 'Your Name',
        'fields' => [
            'ServerName' => [
                'Type' => 'text',
                'Size' => '35',
                'Default' => '',
                'Description' => 'Server hostname',
            ],
            'APIKey' => [
                'Type' => 'password',
                'Size' => '50',
                'Description' => 'API Key for authentication',
            ],
        ],
    ];
}
```

## Step 4: Create Provisioning Functions

```php
<?php
// module.php (continued)

function myServerModule_CreateAccount($params)
{
    $server = $params['server'];
    $apiKey = $params['serveraccesshash'];
    $username = $params['username'];
    $password = $params['password'];
    
    try {
        // Call your API to create account
        $result = callProvisioningAPI($server, $apiKey, [
            'action' => 'create_account',
            'username' => $username,
            'password' => $password,
            'package' => $params['configoption1'],
        ]);
        
        if ($result['success']) {
            return 'success';
        } else {
            return ['error' => $result['message']];
        }
    } catch (Exception $e) {
        return ['error' => $e->getMessage()];
    }
}

function myServerModule_TerminateAccount($params)
{
    // Terminate account logic
    return 'success';
}

function myServerModule_SuspendAccount($params)
{
    // Suspend account logic
    return 'success';
}

function myServerModule_UnsuspendAccount($params)
{
    // Unsuspend account logic
    return 'success';
}

function myServerModule_ChangePassword($params)
{
    // Change password logic
    return 'success';
}

function myServerModule_ChangePackage($params)
{
    // Change package/upgrade logic
    return 'success';
}

function myServerModule_ClientArea($params)
{
    // Custom client area page
    return [
        'templatefile' => 'templates/clientarea',
        'vars' => [
            'serverUrl' => $params['server'],
        ],
    ];
}
```

## Step 5: Create Admin Area Page

```php
function myServerModule_AdminServicesTabFields($params)
{
    return [
        'Server ID' => '<input type="text" value="12345" readonly>',
    ];
}
```

## Step 6: Create Client Area Template

```smarty
// templates/clientarea.tpl

<div class="module-client-area">
    <h2>My Server Module</h2>
    <p>Server URL: {$serverUrl}</p>
    <a href="{$serverUrl}/client" class="btn btn-primary">
        Open Control Panel
    </a>
</div>
```

## Step 7: Create Usage Metrics (Optional)

```php
function myServerModule_UsageUpdate($params)
{
    $data = fetchBandwidthUsage($params);
    
    return [
        'diskusage' => $data['disk'],
        'disklimit' => $data['disk_limit'],
        'bwusage' => $data['bandwidth'],
        'bwlimit' => $data['bandwidth_limit'],
    ];
}
```

## Step 8: Create Module Logo

```bash
mkdir -p /var/www/whmcs/modules/servers/my_server_module/assets
# Add logo.png (64x64)
```

## Step 9: Register Module

Navigate to: Setup > Servers > Create New Server

1. Select module: "My Server Module"
2. Configure settings
3. Save

## Step 10: Test Module

1. Create test product with module
2. Order product
3. Verify provisioning
4. Test suspend/unsuspend
5. Test termination

## Module Development Checklist

- [ ] Directory structure created
- [ ] Module definition created
- [ ] Create account function
- [ ] Terminate function
- [ ] Suspend/Unsuspend functions
- [ ] Password change function
- [ ] Package change function
- [ ] Client area page
- [ ] Templates created
- [ ] Module registered
- [ ] Module tested
