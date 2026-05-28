# CLAUDE.md — WHMCS Module Development Guide
# Project: HiTechCloud DevKits — WHMCS Specialist
# Owner: Pho Tue SoftWare And Technology Solutions JSC (MST: 0318222203)
# Version: 2.0 | Updated: 2026-05-28
#
# ── Role of this file ────────────────────────────────────────────────────────
# CLAUDE.md is the **technical reference** for WHMCS module development.
# This agent focuses exclusively on WHMCS modules:
#   - Provisioning modules (servers/)
#   - Payment gateways (gateways/)
#   - Registrars (registrars/)
#   - Addon modules (addons/)
#   - Notification providers (notifications/)
#   - Hooks and automation
#
# ── Read order (authority hierarchy, top wins on conflict) ──────────────────
#   1. .agents-whmcs/CLAUDE.md (this file)  — WHMCS technical reference
#   2. .agents-whmcs/devkits/*.md           — Complete module templates
#   3. .agents-whmcs/workflows/*.md         — Module type workflows
#   4. .agents-whmcs/skills/*.md            — Specialized development guides
#   5. .agents-whmcs/docs/*.md             — Reference documentation
#
# ── Module Types (WHMCS) ─────────────────────────────────────────────────────
#   Server/Provisioning  → modules/servers/{module}/
#   Payment Gateway       → modules/gateways/{module}.php + callback/
#   Registrar            → modules/registrars/{module}/
#   Addon Module         → modules/addons/{module}/
#   Notification         → modules/notifications/{Provider}/
#   Hooks                → hooks.php
#   Merchant Gateway     → modules/gateways/ (capture/refund/remote input)

---

## 1. MODULE TYPES

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

// Lifecycle functions - Return 'success' or error string
function {module}_CreateAccount(array $params): string { }
function {module}_SuspendAccount(array $params): string { }
function {module}_UnsuspendAccount(array $params): string { }
function {module}_TerminateAccount(array $params): string { }
function {module}_ChangePassword(array $params): string { }
function {module}_ChangePackage(array $params): string { }

// Test connection - Return ['success' => bool, 'error' => string]
function {module}_TestConnection(array $params): array { }

// Client area - Return ['pagetitle', 'templatefile', 'vars']
function {module}_ClientArea(array $params): array { }
```

### 1.2 Payment Gateway (modules/gateways/)

**Standard Gateway:**
```php
function {module}_config(): array {
    return [
        'FriendlyName' => ['value' => 'Display Name'],
        'MerchantId' => ['FriendlyName' => 'Merchant ID', 'Type' => 'text'],
        'ApiKey' => ['FriendlyName' => 'API Key', 'Type' => 'password'],
    ];
}

// Return HTML form to payment page
function {module}_link(array $params): string { }

// Optional: Process refund
function {module}_refund($params): array {
    return ['status' => 'success', 'transid' => 'xxx'];
}
```

**Callback Handler:**
```php
// modules/gateways/callback/{module}.php
function {module}_callback() {
    $gateway = $_POST;
    // Validate signature, check status
    // logTransaction()
    // addInvoicePayment()
    header('Location: ' . $systemurl . '/viewinvoice.php?id=' . $orderId);
}
```

### 1.3 Registrar Module (modules/registrars/)

**Return Array:** `['success' => true]` or `['error' => 'Error message']`

```php
function {module}_RegisterDomain(array $params): array { }
function {module}_TransferDomain(array $params): array { }
function {module}_RenewDomain(array $params): array { }
function {module}_GetNameservers(array $params): array {
    return ['ns1' => 'ns1.example.com', 'ns2' => 'ns2.example.com'];
}
function {module}_SaveNameservers(array $params): array { }
function {module}_GetContactDetails(array $params): array { }
function {module}_SaveContactDetails(array $params): array { }
function {module}_GetDNS(array $params): array {
    return [['hostname' => '', 'type' => 'A', 'address' => '']];
}
function {module}_SaveDNS(array $params): array { }
function {module}_GetRegistrarLock(array $params): array {
    return ['lockstatus' => 'locked'];
}
function {module}_SaveRegistrarLock(array $params): array { }
function {module}_GetEPPCodes(array $params): array {
    return ['eppcode' => 'ABCD1234'];
}
function {module}_Sync(array $params): array {
    return ['status' => 'Active', 'expiry' => '2025-05-28'];
}
```

### 1.4 Addon Module (modules/addons/)

```php
function {module}_config(): array {
    return [
        'name' => 'Module Name',
        'description' => 'Description',
        'version' => '1.0',
    ];
}

function {module}_activate(): array {
    // Create tables with mod_ prefix
    Capsule::schema()->create('mod_{module}_data', function($t) {
        $t->increments('id');
        $t->string('field');
    });
    return ['status' => 'success', 'description' => 'Activated'];
}

function {module}_deactivate(): array {
    // Drop tables
    Capsule::schema()->dropIfExists('mod_{module}_data');
    return ['status' => 'success'];
}

function {module}_upgrade(array $vars): void {
    // Version migrations: if ($vars['version'] < 200) { ... }
}

function {module}_output(array $vars): void {
    // CSRF check is REQUIRED for POST
    if ($_SERVER['REQUEST_METHOD'] === 'POST') {
        check_token('WHMCS.admin.default');
    }
}

function {module}_clientarea(array $vars): array {
    return ['pagetitle' => '', 'templatefile' => '', 'vars' => []];
}
```

### 1.5 Notification Provider (modules/notifications/)

```php
namespace WHMCS\Module\Notification\{Provider};

use WHMCS\Module\Contracts\NotificationModuleInterface;
use WHMCS\Notification\Contracts\NotificationInterface;
use WHMCS\Module\Notification\DescriptionTrait;

class Provider implements NotificationModuleInterface {
    use DescriptionTrait; // REQUIRED

    public static function moduleConfiguration(): array {
        return [['Name' => 'api_key', 'Type' => 'password', 'FriendlyName' => 'API Key']];
    }

    public function testConnection(): void {
        throw new \Exception('Connection failed'); // NOT return false
    }

    public function notificationSettings(): array {
        return [['Name' => 'channel', 'Type' => 'text', 'FriendlyName' => 'Channel ID']];
    }

    public function send(NotificationInterface $notification, array $settings): void {
        throw new \Exception('Failed to send'); // NOT return false
    }
}
```

---

## 2. HOOKS SYSTEM

```php
add_hook('ClientAdd', 1, function($vars) { });
add_hook('AfterModuleCreate', 1, function($vars) { });
add_hook('AfterModuleSuspend', 1, function($vars) { });
add_hook('AfterModuleTerminate', 1, function($vars) { });
add_hook('InvoicePaid', 1, function($vars) { });
add_hook('TicketOpen', 1, function($vars) { });
add_hook('DailyCronJob', 1, function($vars) { });
```

---

## 3. DATABASE (Capsule/Eloquent)

```php
use WHMCS\Database\Capsule;

// Query
Capsule::table('tblinvoices')->where('userid', $userid)->get();

// Insert
Capsule::table('mod_mymodule_table')->insert(['field' => '...']);

// Update
Capsule::table('mod_mymodule_table')->where('id', $id)->update(['field' => '...']);

// Schema
Capsule::schema()->create('mod_mymodule_table', function($t) {
    $t->increments('id');
    $t->string('field');
    $t->timestamps();
});
```

**RULE:** Addon tables MUST use `mod_` prefix: `mod_{module}_table_name`

---

## 4. SECURITY

### CSRF Protection (Addon modules)
```php
if ($_SERVER['REQUEST_METHOD'] === 'POST') {
    check_token('WHMCS.admin.default'); // Admin area
    check_token('WHMCS.default'); // Client area
}
```

### Input Validation
```php
$id = (int) $_POST['id'];
$input = preg_replace('/[^a-zA-Z0-9_-]/', '', $_POST['input']);
```

### Callback IP Whitelist
```php
$allowed_ips = ['1.2.3.4'];
if (!in_array($_SERVER['REMOTE_ADDR'], $allowed_ips)) {
    die('Unauthorized');
}
```

---

## 5. API INTEGRATION

```php
private function apiCall(string $method, string $endpoint, array $data = []): array {
    $ch = curl_init();
    curl_setopt_array($ch, [
        CURLOPT_URL => $this->baseUrl . $endpoint,
        CURLOPT_RETURNTRANSFER => true,
        CURLOPT_TIMEOUT => 30,
        CURLOPT_POST => ($method === 'POST'),
        CURLOPT_POSTFIELDS => json_encode($data),
        CURLOPT_HTTPHEADER => ['Authorization: Bearer ' . $this->apiKey],
    ]);
    $response = curl_exec($ch);
    curl_close($ch);
    return json_decode($response, true);
}
```

---

## 6. LOGGING

```php
logActivity("Module action: " . $result);
logTransaction('{Module}', $data, $status);
```

---

## 7. RETURN VALUE SUMMARY

| Module Type | Success | Error |
|------------|---------|-------|
| Server | `'success'` | `'Error: message'` |
| Registrar | `['success' => true]` | `['error' => 'message']` |
| Addon | `['status' => 'success']` | `['status' => 'error']` |
| Notification | throw Exception (on failure) |

---

## 8. KEY REFERENCE LOCATIONS

```
devkits/              ← Complete module templates
├── provisioning-module/   - Server module
├── gateway-module/       - Payment gateway (5 types)
├── registrar-module/      - Domain registrar
├── addon-module/          - Addon module
└── notification-module/   - Notification provider

skills/              ← Specialized development guides (56 skills)
workflows/           ← Step-by-step development workflows (8 guides)
docs/                ← Reference documentation
```

Last updated: 2026-05-28
