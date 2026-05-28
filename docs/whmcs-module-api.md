# WHMCS Module API Reference

**Version:** 8.x | **Updated:** 2026-05-29
**Related Skills:** `whmcs-server-builder`, `whmcs-gateway-builder`, `whmcs-registrar-builder`

---

## Overview

The WHMCS Module API defines the interface for creating various module types. Each module type (Provisioning, Payment Gateway, Registrar, Addon) has specific functions that WHMCS will call at different stages of operation.

---

## Provisioning/Server Modules

### Module File Location

```
modules/servers/{module_name}/{module_name}.php
```

### Required Functions

```php
<?php
/**
 * MetaData - Module information
 */
function {ModuleName}_MetaData(): array
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
            'server_hostname',
            'serverSecure',
        ],
    ];
}

/**
 * ConfigOptions - Module settings per product
 */
function {ModuleName}_ConfigOptions(array $params): array
{
    return [
        'Plan' => [
            'Type' => 'dropdown',
            'Options' => 'starter,business,enterprise',
            'Default' => 'starter',
            'Description' => 'Select the hosting plan',
        ],
        'Image' => [
            'Type' => 'text',
            'Default' => 'ubuntu-22.04',
            'Description' => 'OS/Image identifier',
        ],
        'AutoSetup' => [
            'Type' => 'yesno',
            'Description' => 'Automatically setup on order',
        ],
    ];
}

/**
 * CreateAccount - Provision new service
 * @param array $params Service and server parameters
 * @return string 'success' or error message
 */
function {ModuleName}_CreateAccount(array $params): string
{
    $server = $params['server'];
    $service = $params['service'];

    try {
        $api = new ProviderApi($server);
        $result = $api->createServer([
            'hostname' => $params['customfields']['hostname'] ?? $service->domain,
            'plan' => $params['configoption1'],
            'image' => $params['configoption2'],
            'username' => $params['username'],
            'password' => $params['password'],
        ]);

        if ($result['status'] === 'active') {
            return 'success';
        }

        return 'Error: ' . ($result['error'] ?? 'Unknown error');

    } catch (\Exception $e) {
        return 'Error: ' . $e->getMessage();
    }
}

/**
 * SuspendAccount - Suspend service
 */
function {ModuleName}_SuspendAccount(array $params): string
{
    try {
        $api = new ProviderApi($params['server']);
        $api->suspendServer($params['serviceid']);

        return 'success';

    } catch (\Exception $e) {
        return 'Error: ' . $e->getMessage();
    }
}

/**
 * UnsuspendAccount - Reactivate service
 */
function {ModuleName}_UnsuspendAccount(array $params): string
{
    try {
        $api = new ProviderApi($params['server']);
        $api->unsuspendServer($params['serviceid']);

        return 'success';

    } catch (\Exception $e) {
        return 'Error: ' . $e->getMessage();
    }
}

/**
 * TerminateAccount - Delete service
 */
function {ModuleName}_TerminateAccount(array $params): string
{
    try {
        $api = new ProviderApi($params['server']);
        $api->deleteServer($params['serviceid']);

        return 'success';

    } catch (\Exception $e) {
        return 'Error: ' . $e->getMessage();
    }
}

/**
 * ChangePassword - Update service password
 */
function {ModuleName}_ChangePassword(array $params): string
{
    try {
        $api = new ProviderApi($params['server']);
        $api->updatePassword($params['serviceid'], $params['password']);

        return 'success';

    } catch (\Exception $e) {
        return 'Error: ' . $e->getMessage();
    }
}

/**
 * ChangePackage - Upgrade/downgrade service
 */
function {ModuleName}_ChangePackage(array $params): string
{
    try {
        $api = new ProviderApi($params['server']);
        $api->resizeServer($params['serviceid'], $params['configoption1']);

        return 'success';

    } catch (\Exception $e) {
        return 'Error: ' . $e->getMessage();
    }
}

/**
 * TestConnection - Verify server credentials
 * @return array ['success' => bool, 'error' => string]
 */
function {ModuleName}_TestConnection(array $params): array
{
    try {
        $api = new ProviderApi($params['server']);
        $result = $api->ping();

        if ($result['status'] === 'ok') {
            return ['success' => true];
        }

        return ['success' => false, 'error' => 'Connection failed'];

    } catch (\Exception $e) {
        return ['success' => false, 'error' => $e->getMessage()];
    }
}

/**
 * ClientArea - Service management page
 * @return array ['pagetitle', 'templatefile', 'vars']
 */
function {ModuleName}_ClientArea(array $params): array
{
    $service = Capsule::table('tblhosting')
        ->where('id', $params['serviceid'])
        ->first();

    return [
        'pagetitle' => 'Service Control Panel',
        'templatefile' => 'templates/clientarea',
        'vars' => [
            'server_status' => $service->domainstatus,
            'server_ip' => $service->dedicatedip,
            'control_panel_url' => 'https://panel.example.com/server/' . $service->id,
        ],
    ];
}
```

---

## Service Provider API Interface

### Interface Definition

```php
<?php
namespace WHMCS\Module\Contracts;

/**
 * Server module interface
 */
interface ServerModuleInterface
{
    /**
     * Get module metadata
     */
    public static function MetaData(): array;

    /**
     * Get configuration options
     */
    public function ConfigOptions(array $params): array;

    /**
     * Create account
     */
    public function CreateAccount(array $params): string;

    /**
     * Suspend account
     */
    public function SuspendAccount(array $params): string;

    /**
     * Unsuspend account
     */
    public function UnsuspendAccount(array $params): string;

    /**
     * Terminate account
     */
    public function TerminateAccount(array $params): string;

    /**
     * Change password
     */
    public function ChangePassword(array $params): string;

    /**
     * Change package
     */
    public function ChangePackage(array $params): string;
}
```

### Implementing the Interface

```php
<?php
namespace WHMCS\Module\Server;

use WHMCS\Module\Contracts\ServerModuleInterface;

class Provider implements ServerModuleInterface
{
    private array $params;
    private ApiClient $api;

    public function __construct(array $params)
    {
        $this->params = $params;
        $this->api = new ApiClient($params['server']);
    }

    public static function MetaData(): array
    {
        return [
            'DisplayName' => 'Provider Name',
            'APIVersion' => '1.1',
            'RequiresServer' => true,
        ];
    }

    public function ConfigOptions(array $params): array
    {
        return [
            'Plan' => ['Type' => 'dropdown', 'Options' => 'plan1,plan2'],
        ];
    }

    public function CreateAccount(array $params): string
    {
        $this->api->createServer($params);
        return 'success';
    }

    // ... implement other methods
}
```

---

## Module Parameters

### Standard Parameters

```php
<?php
/**
 * Parameters available in all module functions
 */
function example_module_CreateAccount(array $params): string
{
    // Server information
    $serverId = $params['serverid'];          // Server ID
    $serverHostname = $params['serverhostname'];  // Server hostname
    $serverIp = $params['serverip'];          // Server IP
    $serverUsername = $params['serverusername'];  // Username
    $serverPassword = $params['serverpassword'];  // Password
    $serverAccessHash = $params['serveraccesshash'];  // Access hash
    $serverSecure = $params['serversecure'];   // SSL flag

    // Service information
    $serviceId = $params['serviceid'];        // Service ID
    $serviceUsername = $params['username'];    // Service username
    $servicePassword = $params['password'];    // Service password
    $serviceDomain = $params['domain'];        // Service domain
    $serviceStatus = $params['status'];        // Current status

    // Product information
    $productId = $params['pid'];              // Product ID
    $productName = $params['productname'];     // Product name
    $configOptions = $params['configoptions'];  // Configurable options
    $configOption1 = $params['configoption1'];  // First config option

    // Client information
    $clientId = $params['userid'];            // Client ID
    $clientEmail = $params['clientsdetails']['email'];  // Client email

    // Custom fields
    $customFields = $params['customfields'];    // Array of custom fields
}
```

---

## Module Logging

### logModuleCall

```php
<?php
function {ModuleName}_CreateAccount(array $params): string
{
    logModuleCall(
        'ProviderName',
        'CreateAccount',
        $params,
        null,  // Will be filled with response
        null,
        ['serverpassword', 'password']  // Fields to redact
    );

    try {
        $result = $this->api->createServer($params);

        logModuleCall(
            'ProviderName',
            'CreateAccount',
            $params,
            $result
        );

        return 'success';

    } catch (\Exception $e) {
        logModuleCall(
            'ProviderName',
            'CreateAccount',
            $params,
            ['error' => $e->getMessage()]
        );

        return 'Error: ' . $e->getMessage();
    }
}
```

---

## Addon Module Interface

```php
<?php
/**
 * Addon Module Functions
 */

/**
 * config - Module configuration
 */
function {Module}_config(): array
{
    return [
        'name' => ['Type' => 'System', 'Value' => 'My Addon'],
        'description' => ['Type' => 'System', 'Value' => 'Description'],
        'version' => ['Type' => 'System', 'Value' => '1.0'],
    ];
}

/**
 * activate - Create database tables
 */
function {Module}_activate(): array
{
    Capsule::schema()->create('mod_module_data', function($t) {
        $t->increments('id');
        $t->string('user_id')->unsigned();
        $t->text('data');
        $t->timestamps();
    });

    return [
        'status' => 'success',
        'description' => 'Module activated successfully',
    ];
}

/**
 * deactivate - Drop database tables
 */
function {Module}_deactivate(): array
{
    Capsule::schema()->dropIfExists('mod_module_data');

    return [
        'status' => 'success',
        'description' => 'Module deactivated',
    ];
}

/**
 * upgrade - Handle version migrations
 */
function {Module}_upgrade(array $vars): void
{
    $currentVersion = $vars['version'];

    if ($currentVersion < 200) {
        Capsule::schema()->table('mod_module_data', function($t) {
            $t->string('new_field')->nullable()->after('data');
        });
    }
}

/**
 * output - Admin area output
 */
function {Module}_output(array $vars): void
{
    $action = $_GET['action'] ?? 'index';

    if ($_SERVER['REQUEST_METHOD'] === 'POST') {
        check_token('WHMCS.admin.default');
    }

    switch ($action) {
        case 'settings':
            $this->renderSettings();
            break;
        default:
            $this->renderDashboard();
    }
}

/**
 * clientarea - Client-facing page
 */
function {Module}_clientarea(array $vars): array
{
    $userId = $_SESSION['uid'];

    return [
        'pagetitle' => 'My Addon',
        'templatefile' => 'templates/clientarea',
        'vars' => [
            'stats' => $this->getUserStats($userId),
        ],
    ];
}
```

---

## registrar Module Interface

```php
<?php
/**
 * Registrar Module Functions
 * All functions return: ['success' => true] or ['error' => 'message']
 */

/**
 * RegisterDomain - New domain registration
 */
function {Registrar}_RegisterDomain(array $params): array
{
    try {
        $api = new RegistrarApi($params);

        $result = $api->registerDomain([
            'domain' => $params['domain'],
            'registration_period' => $params['regperiod'],
            'registrant' => $params['contactdetails'],
        ]);

        if ($result['success']) {
            return ['success' => true];
        }

        return ['error' => $result['message'] ?? 'Registration failed'];

    } catch (\Exception $e) {
        return ['error' => $e->getMessage()];
    }
}

/**
 * TransferDomain - Initiate transfer
 */
function {Registrar}_TransferDomain(array $params): array
{
    $result = $this->api->transfer([
        'domain' => $params['domain'],
        'auth_code' => $params['transfersecret'],
    ]);

    return ['success' => $result['success']];
}

/**
 * RenewDomain - Renew registration
 */
function {Registrar}_RenewDomain(array $params): array
{
    $result = $this->api->renew([
        'domain' => $params['domain'],
        'period' => $params['regperiod'],
    ]);

    return ['success' => $result['success'], 'expiry' => $result['expiry']];
}

/**
 * GetNameservers
 */
function {Registrar}_GetNameservers(array $params): array
{
    $result = $this->api->getNameservers($params['domain']);

    return [
        'ns1' => $result['ns1'],
        'ns2' => $result['ns2'],
        'ns3' => $result['ns3'] ?? '',
        'ns4' => $result['ns4'] ?? '',
    ];
}

/**
 * SaveNameservers
 */
function {Registrar}_SaveNameservers(array $params): array
{
    $this->api->setNameservers($params['domain'], [
        $params['ns1'],
        $params['ns2'],
        $params['ns3'] ?? null,
        $params['ns4'] ?? null,
    ]);

    return ['success' => true];
}

/**
 * GetRegistrarLock
 */
function {Registrar}_GetRegistrarLock(array $params): array
{
    $locked = $this->api->getLockStatus($params['domain']);

    return ['lockstatus' => $locked ? 'locked' : 'unlocked'];
}

/**
 * SaveRegistrarLock
 */
function {Registrar}_SaveRegistrarLock(array $params): array
{
    $this->api->setLock(
        $params['domain'],
        $params['lockstatus'] === 'locked'
    );

    return ['success' => true];
}

/**
 * GetEPPCodes
 */
function {Registrar}_GetEPPCodes(array $params): array
{
    $code = $this->api->getAuthCode($params['domain']);

    return ['eppcode' => $code];
}

/**
 * Sync - Sync domain status
 */
function {Registrar}_Sync(array $params): array
{
    $status = $this->api->getDomainStatus($params['domain']);

    return [
        'status' => $status['status'],
        'expiry' => $status['expiry'],
        'next_due' => $status['next_due'],
    ];
}
```

---

## Best Practices

1. **Return values** - Return 'success' or error string (not array)
2. **Logging** - Use logModuleCall for debugging
3. **Exception handling** - Catch all exceptions and return error
4. **Parameter validation** - Validate all input parameters
5. **Transaction auditing** - Log all API operations
6. **Timeout handling** - Set appropriate timeouts
7. **Retry logic** - Implement retry for transient failures
8. **CSRF protection** - Always check tokens in addons

---

## Related Documentation

- [Module Groups Reference](module-groups.md)
- [Provisioning Module Guide](../devkits/provisioning-module)
- [Payment Gateway Guide](payment-gateway-developer-guide.md)
- [Registrar Module Guide](registrar-module-developer-guide.md)
