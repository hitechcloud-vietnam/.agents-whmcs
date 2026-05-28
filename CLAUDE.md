# CLAUDE.md — WHMCS Module Development Guide
# Project: HiTechCloud DevKits — WHMCS Specialist
# Owner: Pho Tue SoftWare And Technology Solutions JSC (MST: 0318222203)
# Version: 1.0 | Updated: 2026-05-28
#
# ── Role of this file ────────────────────────────────────────────────────────
# CLAUDE.md này là **technical reference** chuyên biệt cho WHMCS module development.
# Đây là phần WHMCS của project chính - tập trung vào:
#   - Provisioning modules (servers/)
#   - Payment gateways (gateways/)
#   - Registrars (registrars/)
#   - Addon modules (addons/)
#   - Notification providers (notifications/)
#   - Hooks và automation
#
# ── Read order (authority hierarchy, top wins on conflict) ──────────────────
#   1. .agents-whmcs/CLAUDE.md (this file)    — WHMCS-specific technical reference
#   2. .agents-whmcs/workflows/*.md          — Module type-specific workflows
#   3. .agents-whmcs/skills/*.md              — Skill-based development guides
#   4. .agents-whmcs/docs/*.md               — Deep-dive references
#   5. Core_exapm_whmcs/                      — WHMCS sample modules (READ ONLY)
#   6. module_done_whmcs/                      — Completed WHMCS modules (READ ONLY)
#
# ── Module Types (WHMCS) ─────────────────────────────────────────────────────
#   Server/Provisioning  → modules/servers/{module}/
#   Payment Gateway     → modules/gateways/{module}.php + callback/
#   Registrar           → modules/registrars/{module}/
#   Addon Module        → modules/addons/{module}/
#   Notification        → modules/notifications/{Provider}/
#   Hooks               → hooks.php (triggers the same functions as addon)
#   Merchant Gateway    → modules/gateways/ (with capture/refund/remote input)
#
# ── Entry Points (Slash Commands) ─────────────────────────────────────────────
#   /whmcs-server {name}     — Tạo provisioning module mới
#   /whmcs-gateway {name}    — Tạo payment gateway
#   /whmcs-addon {name}      — Tạo addon module
#   /whmcs-registrar {name}  — Tạo domain registrar
#   /whmcs-notify {name}     — Tạo notification provider

---

## 1. HIỂU WHMCS MODULE TYPES

### 1.1 Server/Provisioning Module (modules/servers/)

**File:** `modules/servers/{module}/{module}.php`

```php
// MetaData - Required
function {module}_MetaData(): array {
    return [
        'DisplayName' => 'Provider Name',
        'APIVersion'  => '1.0',
        'RequiresServer' => true,
        'DefaultNonSSLPort' => 443,
        'NonSSLPort' => 443,
        'Parameters' => ['server_username', 'server_password', 'server_access_hash'],
    ];
}

// ConfigOptions - Module settings (per product)
function {module}_ConfigOptions(array $params): array {
    return [
        'Plan' => ['Type' => 'dropdown', 'Options' => 'plan1,plan2,plan3'],
        'Image' => ['Type' => 'text', 'Default' => 'ubuntu-22.04'],
    ];
}

// Lifecycle functions
function {module}_CreateAccount(array $params): string {
    // Return 'success' or error message string
}

function {module}_SuspendAccount(array $params): string {
    return 'success';
}

function {module}_UnsuspendAccount(array $params): string {
    return 'success';
}

function {module}_TerminateAccount(array $params): string {
    return 'success';
}

function {module}_ChangePassword(array $params): string {
    // Return 'success' or error
}

function {module}_ChangePackage(array $params): string {
    // Upgrade/downgrade
    return 'success';
}

// Admin area actions
function {module}_AdminServices(array $params): array {
    return [
        'item' => 'value', // custom fields in admin
    ];
}

// Test connection
function {module}_TestConnection(array $params): array {
    return ['success' => true, 'error' => '', 'raw' => ''];
}

// Remote provisioning (for async)
function {module}_RemoteInput(array $params): array
function {module}_RemoteBackup(array $params): array
```

**Client Area Functions:**
```php
function {module}_ClientArea(array $params): array
// Return: ['pagetitle'=>'', 'templatefile'=>'', 'vars'=>[]]

function {module}_ClientAreaAllowedFunctions(): array
// Return: ['FunctionName' => 'Display Label']

function {module}_ClientAreaCustomButtonArray(): array
// Return: ['Label' => 'FunctionName']
```

### 1.2 Payment Gateway (modules/gateways/)

**Standard Gateway:**
```php
// modules/gateways/{module}.php

function {module}_config(): array {
    return [
        'FriendlyName' => ['value' => 'Display Name'],
        'description' => ['value' => 'Description'],
        'MerchantId' => ['FriendlyName' => 'Merchant ID', 'Type' => 'text'],
        'ApiKey' => ['FriendlyName' => 'API Key', 'Type' => 'password'],
        'testMode' => ['FriendlyName' => 'Test Mode', 'Type' => 'yesno'],
    ];
}

function {module}_link(array $params): string {
    // Return: HTML form/link to payment page
    // $params: invoiceid, description, amount, currency, returnurl, etc.
}

function {module}_capture($params): array {
    // Process payment
    return ['status' => 'success', 'transid' => 'xxx', 'raw' => ''];
}

function {module}_refund($params): array {
    return ['status' => 'success', 'transid' => 'xxx'];
}
```

**Callback Handler:**
```php
// modules/gateways/callback/{module}.php

function {module}_callback() {
    $gateway = $_POST; // or $_GET depending on provider

    // Validate signature/IP
    // Check transaction status
    // logTransaction()
    // addInvoicePayment()

    // WHMCS callback pattern:
    logTransaction('{Module}', $data, $status);
    // Status: 'Success', 'Failed', 'Pending', 'Cancelled'

    header('Location: ' . $systemurl . 'modules/gateways/{module}.php');
}
```

### 1.3 Registrar Module (modules/registrars/)

```php
// modules/registrars/{module}/{module}.php

function {module}_RegisterDomain(array $params): array {
    // Return: ['success' => true] or ['error' => 'Error message']
}

function {module}_TransferDomain(array $params): array {
    return ['success' => true] or ['error' => '...'];
}

function {module}_RenewDomain(array $params): array {
    return ['success' => true] or ['error' => '...'];
}

function {module}_GetNameservers(array $params): array {
    return ['ns1' => 'ns1.example.com', 'ns2' => 'ns2.example.com'];
}

function {module}_SaveNameservers(array $params): array {
    return ['success' => true] or ['error' => '...'];
}

function {module}_GetContactDetails(array $params): array {
    // Return: ['Registrant' => [...], 'Admin' => [...], ...]
}

function {module}_SaveContactDetails(array $params): array {
    return ['success' => true];
}

function {module}_RegisterNameserver(array $params): array {
    return ['success' => true];
}

function {module}_ModifyNameserver(array $params): array {
    return ['success' => true];
}

function {module}_DeleteNameserver(array $params): array {
    return ['success' => true];
}

function {module}_GetDNS(array $params): array {
    return [['hostname' => '', 'type' => 'A', 'address' => '']];
}

function {module}_SaveDNS(array $params): array {
    return ['success' => true];
}

function {module}_GetEPPCodes(array $params): array {
    return ['eppcode' => 'ABCD1234'];
}

function {module}_TransferSync(array $params): array {
    // For async transfer status
    return ['status' => 'completed', 'pending' => false];
}

function {module}_RequestDelete(array $params): array {
    return ['success' => true];
}

function {module}_GetRegistrarLock(array $params): array {
    return ['lockstatus' => 'locked'];
}

function {module}_SaveRegistrarLock(array $params): array {
    return ['success' => true];
}

function {module}_Sync(array $params): array {
    // Sync domain status with registrar
    return ['status' => 'Active', 'expiry' => '2025-05-28'];
}
```

### 1.4 Addon Module (modules/addons/)

```php
// modules/addons/{module}/{module}.php

function {module}_config(): array {
    return [
        'name' => 'Module Name',
        'description' => 'Description',
        'version' => '1.0',
        'author' => 'Author',
        'fields' => [
            'api_key' => ['FriendlyName' => 'API Key', 'Type' => 'password'],
        ],
    ];
}

function {module}_activate(): array {
    // Create tables using Capsule
    // Capsule::schema()->create('mod_modulename_table', function($t) {
    //     $t->increments('id');
    //     $t->string('field');
    // });
    return ['status' => 'success', 'description' => 'Activated'];
}

function {module}_deactivate(): array {
    // Capsule::schema()->dropIfExists('mod_modulename_table');
    return ['status' => 'success', 'description' => 'Deactivated'];
}

function {module}_upgrade(array $vars): void {
    // Handle version migrations
    // if ($vars['version'] < 200) { ... }
}

function {module}_output(array $vars): void {
    // Admin area output
    // $vars['action'] contains current action
    // Render templates

    // POST handling:
    if ($_SERVER['REQUEST_METHOD'] === 'POST') {
        check_token('WHMCS.admin.default');
        // handle form
    }

    echo $template->view('templatefile');
}

function {module}_clientarea(array $vars): array {
    return [
        'pagetitle' => 'Page Title',
        'templatefile' => 'clientpage',
        'vars' => ['key' => 'value'],
        'requirelogin' => true, // optional
    ];
}
```

**Sidebar:**
```php
function {module}_sidebar(array $vars): string {
    return '<div>Sidebar HTML</div>';
}
```

### 1.5 Notification Provider (modules/notifications/)

```php
// modules/notifications/{Provider}/{Provider}.php

namespace WHMCS\Module\Notification;

use WHMCS\Module\Contracts\NotificationModuleInterface;
use WHMCS\Notification\Contracts\NotificationInterface;
use WHMCS\Module\Notification\DescriptionTrait;

class Provider implements NotificationModuleInterface {
    use DescriptionTrait; // REQUIRED - provides getDisplayName(), getLogoFileName()

    // Module-level settings (one-time config)
    public static function moduleConfiguration(): array {
        return [
            ['Name' => 'api_key', 'Type' => 'password', 'FriendlyName' => 'API Key'],
        ];
    }

    public function testConnection(): void {
        // Throw Exception on failure
        throw new \Exception('Connection failed');
    }

    // Per-notification-rule settings
    public function notificationSettings(): array {
        return [
            ['Name' => 'channel', 'Type' => 'text', 'FriendlyName' => 'Channel ID'],
        ];
    }

    public function send(NotificationInterface $notification, array $settings): void {
        // Throw Exception on failure
        // $notification->getTitle(), getMessage(), getUrl(), getAttributes()

        // Send notification via provider
        throw new \Exception('Failed to send');
    }

    // Optional: Dynamic fields for notification rules
    public function getDynamicField(string $fieldName, array $settings): array {
        // Return field config for dynamic type
        return [];
    }
}
```

---

## 2. WHMCS HOOKS SYSTEM

### 2.1 Hook Types

**Pre-defined hooks (using `add_hook`):**

```php
// hooks.php - Hook registration file

add_hook('ClientAdd', 1, function($vars) {
    // Handle new client registration
    // $vars contains client data
});

add_hook('ClientEdit', 1, function($vars) {
    // Handle client update
});

add_hook('AfterModuleCreate', 1, function($vars) {
    // After service created
});

add_hook('AfterModuleSuspend', 1, function($vars) {
    // After service suspended
});

add_hook('AfterModuleUnsuspend', 1, function($vars) {
    // After service unsuspended
});

add_hook('AfterModuleTerminate', 1, function($vars) {
    // After service terminated
});

add_hook('InvoicePaid', 1, function($vars) {
    // After invoice payment
});

add_hook('TicketOpen', 1, function($vars) {
    // New ticket created
});

add_hook('DailyCronJob', 1, function($vars) {
    // Daily maintenance
});
```

### 2.2 Common Hook Events

| Hook | Params | Description |
|------|--------|-------------|
| `ClientAdd` | `userid`, `firstname`, `lastname`, `email` | New client registration |
| `ClientEdit` | `userid`, `params` | Client update |
| `AfterModuleCreate` | `serviceid`, `userid`, `params` | Service created |
| `AfterModuleSuspend` | `serviceid`, `userid` | Service suspended |
| `AfterModuleUnsuspend` | `serviceid`, `userid` | Service unsuspended |
| `AfterModuleTerminate` | `serviceid`, `userid` | Service terminated |
| `AfterModuleChangePassword` | `serviceid`, `password` | Password changed |
| `InvoicePaid` | `invoiceid`, `userid`, `amount` | Invoice payment received |
| `InvoiceCreation` | `invoiceid`, `userid` | Invoice created |
| `AddTransaction` | `params` | Transaction added |
| `TicketOpen` | `ticketid`, `userid`, `deptid` | New ticket |
| `TicketReply` | `ticketid`, `userid` | Ticket replied |
| `DomainTransferAccept` | `domainid` | Domain transfer accepted |
| `DomainTransferReject` | `domainid` | Domain transfer rejected |
| `DomainDeletion` | `domainid` | Domain deleted |
| `DailyCronJob` | - | Daily cron execution |
| `HourlyCronJob` | - | Hourly cron execution |
| `PreServiceDelete` | `userid`, `serviceid` | Before service deletion |
| `PostServiceDelete` | `userid`, `serviceid` | After service deletion |

### 2.3 Hook Filter Functions

```php
// Modify data before processing
add_hook('ClientAreaOutput', 1, function($vars) {
    // Modify client area output
    return $vars;
});

// Validate before action
add_hook('ProductEdit', 1, function($vars) {
    // Validate product edit
    return ['success' => true];
});
```

---

## 3. DATABASE & ORM

### 3.1 Capsule (Eloquent) Usage

```php
// Required at top
use WHMCS\Database\Capsule;

$results = Capsule::table('tblinvoices')
    ->where('status', 'Unpaid')
    ->where('userid', $userid)
    ->get();

// Insert
Capsule::table('mod_mymodule_table')->insert([
    'field' => 'value',
    'created_at' => 'now()',
]);

// Update
Capsule::table('mod_mymodule_table')
    ->where('id', $id)
    ->update(['field' => 'newvalue']);

// Delete
Capsule::table('mod_mymodule_table')
    ->where('id', $id)
    ->delete();

// Schema operations in activate()
Capsule::schema()->create('mod_mymodule_table', function($t) {
    $t->increments('id');
    $t->string('field')->default('');
    $t->text('data')->nullable();
    $t->timestamps();
});
```

### 3.2 Table Naming Convention

**REQUIRED:** All addon module tables MUST be prefixed with `mod_`:
- `mod_mymodule_settings` ✓
- `mymodule_settings` ✗ (conflict risk)

---

## 4. CONFIGURATION PATTERNS

### 4.1 Config Array Structure

```php
function {module}_config(): array {
    return [
        'FriendlyName' => [
            'Type' => 'System',
            'Value' => 'Display Name',
        ],
        'description' => [
            'Type' => 'System',
            'Value' => 'Description shown in admin',
        ],
        'YourField' => [
            'FriendlyName' => 'Field Label',
            'Type' => 'text', // text, password, yesno, dropdown
            'Size' => '20',
            'Options' => 'opt1,opt2', // for dropdown
            'Default' => 'default_value',
        ],
    ];
}
```

### 4.2 Field Types

| Type | Description | Options |
|------|-------------|---------|
| `text` | Text input | `Size`, `Default` |
| `password` | Password field | `Size` |
| `yesno` | Checkbox | - |
| `dropdown` | Select box | `Options` (comma-separated) |
| `textarea` | Multi-line text | `Cols`, `Rows` |
| `radio` | Radio buttons | `Options` |
| `checkbox` | Checkboxes | `Options` |

---

## 5. SECURITY BEST PRACTICES

### 5.1 CSRF Protection

```php
// In addon output()
if ($_SERVER['REQUEST_METHOD'] === 'POST') {
    check_token('WHMCS.admin.default');
    // Process form
}

// In templates, add token field
// <input type="hidden" name="token" value="{$token}">

// For clientarea pages
check_token('WHMCS.default');
```

### 5.2 Input Validation

```php
// Sanitize all user inputs
$input = preg_replace('/[^a-zA-Z0-9_-]/', '', $_POST['input']);
$id = (int) $_POST['id'];

// Use WHMCS sanitization
$safe = \App::sanitize('field_name', $_POST['raw_input']);
```

### 5.3 IP Whitelist for Callbacks

```php
// Verify callback IP is from payment provider
$allowed_ips = ['1.2.3.4', '5.6.7.8'];
if (!in_array($_SERVER['REMOTE_ADDR'], $allowed_ips)) {
    die('Unauthorized IP');
}
```

---

## 6. API INTEGRATION PATTERNS

### 6.1 cURL Helper

```php
private function apiCall($method, $endpoint, $data = []) {
    $ch = curl_init();
    curl_setopt_array($ch, [
        CURLOPT_URL => $this->baseUrl . $endpoint,
        CURLOPT_RETURNTRANSFER => true,
        CURLOPT_TIMEOUT => 30,
        CURLOPT_SSL_VERIFYPEER => true,
        CURLOPT_SSL_VERIFYHOST => 2,
        CURLOPT_HTTPHEADER => [
            'Content-Type: application/json',
            'Authorization: Bearer ' . $this->apiKey,
        ],
    ]);

    if ($method === 'POST') {
        curl_setopt($ch, CURLOPT_POST, true);
        curl_setopt($ch, CURLOPT_POSTFIELDS, json_encode($data));
    }

    $response = curl_exec($ch);
    $httpCode = curl_getinfo($ch, CURLINFO_HTTP_CODE);
    $error = curl_error($ch);
    curl_close($ch);

    if ($error) {
        throw new \Exception('cURL Error: ' . $error);
    }

    return json_decode($response, true);
}
```

### 6.2 Logging

```php
// Use localApi for WHMCS logging
use WHMCS\Module\Env;

logActivity("Module action: " . $result);
logTransaction('{Module}', $data, $status);
```

---

## 7. MODULE CHECKLIST

### Server/Provisioning Module
```
□ {module}_MetaData() - Required fields
□ {module}_ConfigOptions() - Product configuration
□ {module}_CreateAccount() - Return 'success' or error string
□ {module}_SuspendAccount() - Return 'success' or error string
□ {module}_UnsuspendAccount() - Return 'success' or error string
□ {module}_TerminateAccount() - Return 'success' or error string
□ {module}_ChangePassword() - Return 'success' or error string
□ {module}_ChangePackage() - Return 'success' or error string
□ {module}_TestConnection() - Return ['success' => bool]
□ Client area functions if needed
□ Logo file (logo.png, 80x80px)
□ whmcs.json metadata
```

### Payment Gateway
```
□ {module}_config() - Configuration fields
□ {module}_link() - Return HTML form to payment page
□ {module}_capture() - Process payment, return ['status' => 'success']
□ {module}_refund() - Process refund
□ modules/gateways/callback/{module}.php - Callback handler
□ logTransaction() calls
□ IP whitelist for callback
□ Signature verification
```

### Registrar
```
□ {module}_RegisterDomain() - Return ['success' => true] or ['error' => '...']
□ {module}_TransferDomain() - Return ['success' => true] or ['error' => '...']
□ {module}_RenewDomain() - Return ['success' => true] or ['error' => '...']
□ {module}_GetNameservers() / _SaveNameservers()
□ {module}_GetContactDetails() / _SaveContactDetails()
□ {module}_GetDNS() / _SaveDNS()
□ {module}_GetRegistrarLock() / _SaveRegistrarLock()
□ {module}_Sync() - Sync domain status
□ EPP code handling
□ Logo file
```

### Addon Module
```
□ {module}_config() - Module configuration
□ {module}_activate() - Create tables with mod_ prefix
□ {module}_deactivate() - Drop tables
□ {module}_upgrade() - Version migrations
□ {module}_output() - Admin area with CSRF check
□ {module}_clientarea() - Client-facing page if needed
□ check_token() on all POST requests
□ lang/english.php for translations
```

### Notification Provider
```
□ use DescriptionTrait (REQUIRED)
□ implements NotificationModuleInterface
□ static function moduleConfiguration()
□ testConnection() - throw Exception on failure
□ notificationSettings()
□ send() - throw Exception on failure
□ logo.png (80x80px)
□ whmcs.json
```

---

## 8. FILE STRUCTURE

```
modules/
├── servers/
│   └── {module}/
│       ├── {module}.php      ← Main module file
│       ├── lib/
│       │   └── ApiClient.php ← API client class
│       ├── templates/
│       │   ├── overview.tpl   ← Client area template
│       │   └── error.tpl
│       ├── logo.png          ← 80x80px logo
│       └── hooks.php         ← Hooks file (optional)
│
├── gateways/
│   ├── {module}.php          ← Gateway module
│   └── callback/
│       └── {module}.php      ← Callback handler
│
├── registrars/
│   └── {module}/
│       ├── {module}.php      ← Registrar module
│       ├── lib/
│       │   └── ApiClient.php
│       └── logo.png
│
├── addons/
│   └── {module}/
│       ├── {module}.php      ← Addon module
│       ├── hooks.php         ← Hooks
│       ├── lib/
│       ├── templates/
│       │   ├── admin/
│       │   └── clientarea/
│       ├── lang/
│       │   └── english.php
│       └── logo.png
│
└── notifications/
    └── {Provider}/
        ├── {Provider}.php    ← Notification provider
        ├── logo.png           ← 80x80px
        └── whmcs.json
```

---

## 9. OUTPUT DIRECTORIES

```
module_dev_whmcs/              ← WHMCS modules đang dev (VIẾT VÀO ĐÂY)
module_done_whmcs/             ← WHMCS modules hoàn chỉnh (READ ONLY)
Core_exapm_whmcs/              ← WHMCS sample modules (READ ONLY)
.agents-whmcs/                 ← This directory - WHMCS agent configs
```

---

## 10. WHMCS CLIENT AREA TEMPLATES

### Standard Template Variables

```smarty
{$clientarea->serviceid}
{$clientarea->userid}
{$clientarea->domain}
{$clientarea->username}
{$clientarea->password} {* encrypted *}
{$clientarea->status}
{$clientarea->module}
{$clientarea->customfields}
```

### Client Area Array Return

```php
function {module}_clientarea(array $params): array {
    return [
        'pagetitle' => 'Service Details',
        'templatefile' => 'serviceview',
        'vars' => [
            'service' => $serviceData,
            'status' => $params['status'],
            'custombutton' => 'Reboot',
        ],
        'breadcrumb' => '...', // optional
    ];
}
```

---

## 11. ERRORS & EXCEPTIONS

### Module Function Returns

```php
// Success
return 'success';

// Error
return 'Error: Something went wrong';

// Array success (registrar/notification)
return ['success' => true];

// Array error
return ['error' => 'Error message'];
```

### Exception Handling

```php
try {
    // API call
} catch (\Exception $e) {
    logActivity('Module Error: ' . $e->getMessage());
    return 'Error: ' . $e->getMessage();
}
```

---

## 12. WHMCS CORE REFERENCES

### Key Classes

```php
// Capsule (Database)
use WHMCS\Database\Capsule;
Capsule::table('tbl...')

// Config
$config = \Config::getInstance();
$config->get('SettingName');

// Lang
Lang::trans('key');

// Utility
\Illuminate\Support\Facades\Crypt::encrypt($data);

// API (local)
localAPI('command', $params, $adminUsername);
```

### Useful Functions

```php
// Get current user
$userid = session_get('uid');

// Get admin
$admin = Admin::find($adminId);

// Log
logActivity('Message');
logTransaction('Module', $data, $status);

// Currency
$currency = Currency::format($amount, $currencyId);

// Date
date('Y-m-d H:i:s');
```

---

Last updated: 2026-05-28