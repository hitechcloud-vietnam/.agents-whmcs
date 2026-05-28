# WHMCS Module Validator Skill
# Version: 1.0 | Updated: 2026-05-28

## Purpose

This skill validates WHMCS modules for correctness, security, and WHMCS standards compliance.

## When to Use

- After creating a new WHMCS module
- Before releasing a module
- During code review

## Validation Checklist

### 1. Naming Conventions

```php
// File naming
✓ modules/servers/{module}/{module}.php        ← Lowercase module name
✓ modules/gateways/{module}.php                  ← Lowercase
✓ modules/registrars/{module}/{module}.php        ← Lowercase
✗ modules/servers/{Module}/{Module}.php          ← WRONG
✗ modules/addons/{module}/module.php             ← WRONG (should be {module}.php)

// Function naming
✓ function vnpay_config()                        ← Prefix matches file
✓ function vnpay_link(array $params)
✗ function vnPay_config()                        ← WRONG case
```

### 2. Provisioning Module Checklist

```php
// Required functions
✓ {module}_MetaData()                            ← Has DisplayName
✓ {module}_ConfigOptions(array $params): array   ← Returns config
✓ {module}_CreateAccount(array $params): string  ← Returns 'success' or error
✓ {module}_SuspendAccount(array $params): string
✓ {module}_UnsuspendAccount(array params): string
✓ {module}_TerminateAccount(array $params): string
✓ {module}_ChangePassword(array $params): string
✓ {module}_ChangePackage(array $params): string
✓ {module}_TestConnection(array $params): array  ← Returns ['success' => bool]

// Optional but common
○ {module}_ClientArea(array $params): array
○ {module}_AdminServices(array $params): array

// File existence
✓ logo.png (80x80px)
```

### 3. Payment Gateway Checklist

```php
// Required functions
✓ {module}_config(): array                        ← Has FriendlyName
✓ {module}_link(array $params): string           ← Returns HTML form
✓ {module}_refund(array $params): array          ← Returns ['status' => ...]

// Callback (for redirect gateways)
✓ modules/gateways/callback/{module}.php
✓ logTransaction() called
✓ IP validation (if provider provides IPs)

// Tokenization gateway
✓ {module}_capture_form(array $params): array
✓ {module}_capture_token(array $params): array
✓ {module}_capture(array $params): array

// Security
✓ Signature validation in callback
✓ No hardcoded credentials
```

### 4. Registrar Module Checklist

```php
// Required functions
✓ {module}_getConfigArray(): array
✓ {module}_RegisterDomain(array $params): array   ← ['success' => true] or ['error' => '...']
✓ {module}_TransferDomain(array $params): array
✓ {module}_RenewDomain(array $params): array
✓ {module}_GetNameservers(array $params): array
✓ {module}_SaveNameservers(array $params): array
✓ {module}_GetContactDetails(array $params): array
✓ {module}_SaveContactDetails(array $params): array
✓ {module}_GetRegistrarLock(array $params): array
✓ {module}_SaveRegistrarLock(array $params): array
✓ {module}_GetEPPCodes(array $params): array      ← ['eppcode' => '...']
✓ {module}_Sync(array $params): array             ← ['status' => ..., 'expiry' => ...]

// EPP specific
○ {module}_RegisterNameserver(array $params)
○ {module}_ModifyNameserver(array $params)
○ {module}_DeleteNameserver(array $params)
```

### 5. Addon Module Checklist

```php
// Required functions
✓ {module}_config(): array
✓ {module}_activate(): array                     ← Returns ['status' => ..., 'description' => ...]
✓ {module}_deactivate(): array
✓ {module}_output(array $vars): void             ← Admin output
○ {module}_upgrade(array $vars): void

// Table naming (CRITICAL)
✓ Table names prefixed with mod_{module}_         ← mod_vnpay_settings, NOT vnpay_settings

// CSRF protection
✓ check_token() on POST requests
✓ <input type="hidden" name="token" value="{$token}">

// Client area (if needed)
○ {module}_clientarea(array $vars): array
○ requirelogin if needed

// Language
○ lang/english.php
```

### 6. Notification Provider Checklist

```php
// CRITICAL - these are REQUIRED
✓ use DescriptionTrait;                           ← NOT optional!
✓ implements NotificationModuleInterface

// Required methods
✓ public static function moduleConfiguration(): array
✓ public function testConnection(): void         ← Throw Exception, NOT return false
✓ public function notificationSettings(): array
✓ public function send(NotificationInterface $notification, array $settings): void
    ← Throw Exception, NOT return false

// From DescriptionTrait
✓ public function getDisplayName(): string
✓ public function getLogoFileName(): string       ← Returns 'logo.png'

// File existence
✓ logo.png (80x80px)
✓ whmcs.json
```

### 7. Security Validation

```php
// CSRF
✓ check_token() in all POST handlers
✓ Token field in all forms

// Input sanitization
✓ htmlspecialchars() on user output
✓ preg_replace() / intval() for numeric input
✓ pdo::quote() for SQL (or use Capsule)

// Passwords
✓ No plaintext password storage
✓ Encrypted storage for sensitive data

// API secrets
✓ No hardcoded API keys in code
✓ Configuration in admin panel only

// Callbacks
✓ Signature validation
✓ IP whitelist (if available)
✓ Replay attack prevention (check transaction ID)
```

### 8. Error Handling

```php
// Provisioning - return string
function module_CreateAccount($params) {
    try {
        // ...
        return 'success';
    } catch (\Exception $e) {
        return 'Error: ' . $e->getMessage();
    }
}

// Registrar - return array
function module_RegisterDomain($params) {
    try {
        // ...
        return ['success' => true];
    } catch (\Exception $e) {
        return ['error' => $e->getMessage()];
    }
}

// Notification - throw Exception
function module_send($notification, $settings) {
    if ($failed) {
        throw new \Exception('Failed to send notification');
    }
}

function module_testConnection() {
    if ($failed) {
        throw new \Exception('Connection failed');
    }
    // Don't return anything on success
}
```

### 9. Database Patterns (Addon)

```php
// ✓ Correct
Capsule::schema()->create('mod_vnpay_transactions', function($t) {
    $t->increments('id');
    $t->string('transaction_id')->unique();
    $t->timestamps();
});

// ✗ Wrong
Capsule::schema()->create('vnpay_transactions', function($t) {  // Missing mod_ prefix
    // ...
});
```

### 10. File Structure Validation

```
modules/
├── servers/
│   └── {module}/
│       ├── {module}.php        ✓ Required
│       ├── logo.png            ✓ 80x80px
│       ├── lib/                ○ (optional)
│       └── templates/          ○ (optional)
│
├── gateways/
│   ├── {module}.php            ✓ Required
│   └── callback/
│       └── {module}.php        ✓ (if redirect)
│
├── registrars/
│   └── {module}/
│       ├── {module}.php        ✓ Required
│       └── logo.png            ✓ (optional)
│
├── addons/
│   └── {module}/
│       ├── {module}.php        ✓ Required
│       ├── hooks.php           ○ (optional)
│       └── lang/
│           └── english.php     ○ (optional)
│
└── notifications/
    └── {Provider}/
        ├── {Provider}.php      ✓ Required
        ├── logo.png            ✓ 80x80px
        └── whmcs.json          ✓ Required
```

## Common Mistakes to Avoid

1. **Notification modules missing `use DescriptionTrait`**
   - Module won't appear in WHMCS admin

2. **Addon tables without `mod_` prefix**
   - Risk of conflict with WHMCS tables

3. **Provisioning module returning array instead of string**
   - WHMCS expects `'success'` or error message string

4. **Registrar functions returning string instead of array**
   - WHMCS expects `['success' => true]` or `['error' => '...']`

5. **Notification functions returning false instead of throwing**
   - WHMCS catches exceptions, not return values

6. **Missing logo.png**
   - Module won't show correctly in admin

7. **Wrong file naming**
   - Use `{module}.php` not `module.php`

## Usage

After creating a WHMCS module:

1. Review this checklist section by section
2. Fix any issues found
3. Test the module in a WHMCS installation
4. Verify all features work correctly