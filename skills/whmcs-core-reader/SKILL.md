# WHMCS Core Reader Skill
# Version: 1.0 | Updated: 2026-05-28

## Purpose

This skill provides guidance for reading and understanding WHMCS core source code and sample modules.

## When to Use

- Before starting a new WHMCS module
- When you need to understand WHMCS internal patterns
- When converting modules between platforms

## Skills

### 1. Finding WHMCS Core Files

```
Core_exapm_whmcs/                      ← Sample modules (START HERE)
Core_exapm_whmcs/Whmcs Docs Dev/      ← Official WHMCS documentation
```

### 2. Core Sample Modules to Reference

| Module Type | Path | Notes |
|-------------|------|-------|
| Provisioning | `sample-provisioning-module/` | Server module example |
| Gateway | `sample-gateway-module/` | Standard payment gateway |
| Merchant | `sample-merchant-gateway/` | Card capture gateway |
| Registrar | `sample-registrar-module/` | Domain registrar |
| Addon | `sample-addon-module/` | Admin/client area |
| Notification | `sample-notification-module/` | Notification provider |
| Tokenization | `sample-tokenisation-gateway-module/` | Token-based payments |
| Remote Input | `sample-remote-input-gateway/` | iframe hosted form |
| Remote Bank | `sample-remote-bank-gateway-module/` | Bank simulation |

### 3. Reading WHMCS Core

Key files to examine:

```php
// Provisioning module key functions
sample-provisioning-module/modules/servers/provisioningmodule/provisioningmodule.php
  - provisioningmodule_MetaData()
  - provisioningmodule_ConfigOptions()
  - provisioningmodule_CreateAccount()
  - provisioningmodule_SuspendAccount()
  - provisioningmodule_UnsuspendAccount()
  - provisioningmodule_TerminateAccount()
  - provisioningmodule_ChangePassword()
  - provisioningmodule_ChangePackage()
  - provisioningmodule_AdminServices()
  - provisioningmodule_TestConnection()
  - provisioningmodule_ClientArea()

// Gateway key functions
sample-gateway-module/modules/gateways/gatewaymodule.php
  - gatewaymodule_config()
  - gatewaymodule_link()
  - gatewaymodule_capture()
  - gatewaymodule_refund()

sample-merchant-gateway/modules/gateways/merchantgateway.php
  - merchantgateway_config()
  - merchantgateway_capture_form()
  - merchantgateway_capture()
  - merchantgateway_refund()

// Addon key functions
sample-addon-module/modules/addons/addonmodule/addonmodule.php
  - addonmodule_config()
  - addonmodule_activate()
  - addonmodule_deactivate()
  - addonmodule_upgrade()
  - addonmodule_output()
  - addonmodule_clientarea()
  - addonmodule_sidebar()

// Registrar key functions
sample-registrar-module/modules/registrars/registrarmodule/registrarmodule.php
  - registrarmodule_getConfigArray()
  - registrarmodule_RegisterDomain()
  - registrarmodule_TransferDomain()
  - registrarmodule_RenewDomain()
  - registrarmodule_GetNameservers()
  - registrarmodule_SaveNameservers()
  - registrarmodule_GetContactDetails()
  - registrarmodule_SaveContactDetails()
  - registrarmodule_GetRegistrarLock()
  - registrarmodule_SaveRegistrarLock()
  - registrarmodule_GetEPPCodes()
  - registrarmodule_Sync()

// Notification key functions
sample-notification-module/modules/notifications/SampleNotificationModule/SampleNotificationModule.php
  - use DescriptionTrait (REQUIRED)
  - moduleConfiguration()
  - testConnection() throws Exception
  - notificationSettings()
  - send() throws Exception
```

### 4. Key Patterns from Core

#### Naming Convention
- Function prefix = module name (e.g., `vnpay_config`, `vnpay_link`)
- File name = `{module}.php` for gateways, `{module}/{module}.php` for others
- Class in API client = `ModuleNameApiClient` or similar

#### Return Values

```php
// Provisioning: Return string
function module_CreateAccount() {
    return 'success';  // or error message
}

// Registrar: Return array
function module_RegisterDomain() {
    return ['success' => true];  // or ['error' => 'message']
}

// Notification: Throw Exception (NOT return false)
function module_send() {
    if ($failed) {
        throw new \Exception('Failed to send');
    }
}

function module_testConnection() {
    if ($failed) {
        throw new \Exception('Connection failed');
    }
}
```

#### Database Access (Addon)

```php
use WHMCS\Database\Capsule;

// Create table in activate()
Capsule::schema()->create('mod_modulename_table', function($t) {
    $t->increments('id');
    $t->string('field');
});

// Query in module functions
Capsule::table('mod_modulename_table')
    ->where('field', 'value')
    ->get();
```

#### CSRF Protection

```php
// In output() function
if ($_SERVER['REQUEST_METHOD'] === 'POST') {
    check_token('WHMCS.admin.default');
    // Handle form
}

// In templates
<input type="hidden" name="token" value="{$token}">
```

### 5. WHMCS Variable Patterns

```php
// Provisioning params array
$params = [
    'domain' => 'example.com',
    'username' => 'user',
    'password' => 'pass', // encrypted
    'customfields' => [...],
    'configoption1' => 'value',
    'serverhostname' => 'api.provider.com',
    'serverhttpprefix' => 'https',
    'serverusername' => 'api_user',
    'serverpassword' => 'api_key',
];

// Clientarea return array
return [
    'pagetitle' => 'Page Title',
    'templatefile' => 'filename',
    'vars' => ['key' => 'value'],
    'requirelogin' => true,
];
```

### 6. Common Hooks

```php
// hooks.php
add_hook('ClientAdd', 1, function($vars) {
    // $vars['userid'], $vars['email'], etc.
});

add_hook('AfterModuleCreate', 1, function($vars) {
    // $vars['serviceid'], $vars['userid']
});

add_hook('InvoicePaid', 1, function($vars) {
    // $vars['invoiceid'], $vars['userid'], $vars['amount']
});
```

## Usage

Before creating a new WHMCS module:

1. Read the corresponding sample module
2. Identify the key functions needed
3. Use the workflow document for that module type
4. Follow the naming conventions exactly

## Quick Reference

| Module Type | File Path | Key Function Pattern |
|-------------|-----------|---------------------|
| Server | `modules/servers/{name}/{name}.php` | `{name}_CreateAccount()` |
| Gateway | `modules/gateways/{name}.php` | `{name}_config()` |
| Registrar | `modules/registrars/{name}/{name}.php` | `{name}_RegisterDomain()` |
| Addon | `modules/addons/{name}/{name}.php` | `{name}_config()` |
| Notification | `modules/notifications/{Name}/{Name}.php` | `{Name}_moduleConfiguration()` |