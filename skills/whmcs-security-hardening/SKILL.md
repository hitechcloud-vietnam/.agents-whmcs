# WHMCS Security Hardening Skill
# Version: 1.0 | Updated: 2026-05-28

## Purpose

Guide for securing WHMCS modules against common vulnerabilities and attack vectors.

## When to Use

- Before releasing any module
- When adding new features that handle sensitive data
- When integrating with external APIs
- When processing payments

## Security Checklist

### 1. Input Validation

```php
// Always validate and sanitize user input

// Integer validation
$serviceId = filter_input(INPUT_POST, 'serviceid', FILTER_VALIDATE_INT);
if ($serviceId === false || $serviceId <= 0) {
    throw new \Exception('Invalid service ID');
}

// String sanitization
$domain = filter_input(INPUT_POST, 'domain', FILTER_SANITIZE_URL);
// or more specific
$domain = preg_replace('/[^a-zA-Z0-9.-]/', '', $_POST['domain']);

// Email validation
$email = filter_input(INPUT_POST, 'email', FILTER_VALIDATE_EMAIL);
if ($email === false) {
    throw new \Exception('Invalid email address');
}

// Enum validation
$allowedStatuses = ['pending', 'active', 'suspended', 'terminated'];
$status = $_POST['status'];
if (!in_array($status, $allowedStatuses)) {
    throw new \Exception('Invalid status');
}
```

### 2. CSRF Protection

```php
// In output() function - check token
function module_output($vars) {
    if ($_SERVER['REQUEST_METHOD'] === 'POST') {
        check_token('WHMCS.admin.default');
        // Handle POST
    }
}

// In templates - add token field
/*
<input type="hidden" name="token" value="{$token}">
*/

// For client area
if ($_SERVER['REQUEST_METHOD'] === 'POST') {
    check_token('WHMCS.default');
}
```

### 3. SQL Injection Prevention

```php
// Use prepared statements with Capsule
use WHMCS\Database\Capsule;

// Good - parameterized query
$results = Capsule::table('mod_module_table')
    ->where('user_id', $userId)
    ->where('status', 'active')
    ->get();

// For complex queries
$stmt = Capsule::connection()->getPdo()->prepare(
    "SELECT * FROM mod_module_data WHERE user_id = :uid AND type = :type"
);
$stmt->execute(['uid' => $userId, 'type' => $type]);
$results = $stmt->fetchAll();

// Never do this:
// BAD: Capsule::select("SELECT * FROM table WHERE name = '$name'");
```

### 4. XSS Prevention

```php
// Always escape output in templates
{$variable|escape:'html'}
{$variable|escape:'javascript'}
{$variable|escape:'url'}

// In PHP when building HTML
htmlspecialchars($variable, ENT_QUOTES, 'UTF-8');

// Never do this:
// BAD: echo "Welcome, " . $_POST['name'];
// GOOD: echo "Welcome, " . htmlspecialchars($_POST['name'], ENT_QUOTES, 'UTF-8');
```

### 5. Payment Gateway Security

```php
// Callback IP validation
function validateCallbackIp(array $allowedIps): bool {
    $clientIp = $_SERVER['REMOTE_ADDR'];
    return in_array($clientIp, $allowedIps);
}

// Signature validation
function validateSignature(array $data, string $secret): bool {
    $receivedSig = $data['signature'] ?? '';
    ksort($data);
    unset($data['signature']);

    $signData = http_build_query($data);
    $expectedSig = strtoupper(hash_hmac('sha256', $signData, $secret));

    return hash_equals($expectedSig, $receivedSig);
}

// Replay attack prevention
function isTransactionProcessed(string $transId): bool {
    return Capsule::table('mod_module_transactions')
        ->where('transaction_id', $transId)
        ->exists();
}

function markTransactionProcessed(string $transId): void {
    Capsule::table('mod_module_transactions')->insert([
        'transaction_id' => $transId,
        'processed_at' => date('Y-m-d H:i:s'),
    ]);
}
```

### 6. Password Handling

```php
// Never store plaintext passwords
// Use encrypted storage for API credentials

function saveEncryptedCredential(string $key, string $value): void {
    $encrypted = \Illuminate\Support\Facades\Crypt::encrypt($value);
    Capsule::table('mod_module_settings')
        ->updateOrInsert(
            ['setting_key' => $key],
            ['setting_value' => $encrypted]
        );
}

function getDecryptedCredential(string $key): string {
    $encrypted = Capsule::table('mod_module_settings')
        ->where('setting_key', $key)
        ->value('setting_value');

    return $encrypted ? \Illuminate\Support\Facades\Crypt::decrypt($encrypted) : '';
}
```

### 7. File Upload Security

```php
// Validate uploaded files
function validateFileUpload(array $file): bool {
    // Check file size (max 2MB)
    if ($file['size'] > 2 * 1024 * 1024) {
        return false;
    }

    // Check MIME type
    $allowedTypes = ['image/jpeg', 'image/png', 'image/gif'];
    $finfo = finfo_open(FILEINFO_MIME_TYPE);
    $mimeType = finfo_file($finfo, $file['tmp_name']);
    finfo_close($finfo);

    if (!in_array($mimeType, $allowedTypes)) {
        return false;
    }

    // Check file extension
    $allowedExtensions = ['jpg', 'jpeg', 'png', 'gif'];
    $extension = strtolower(pathinfo($file['name'], PATHINFO_EXTENSION));
    if (!in_array($extension, $allowedExtensions)) {
        return false;
    }

    return true;
}

// Generate safe filename
function generateSafeFilename(string $originalName): string {
    $extension = strtolower(pathinfo($originalName, PATHINFO_EXTENSION));
    $safeName = preg_replace('/[^a-zA-Z0-9]/', '', pathinfo($originalName, PATHINFO_FILENAME));
    return $safeName . '_' . time() . '.' . $extension;
}
```

### 8. API Security

```php
// Always use HTTPS
$ch = curl_init();
curl_setopt_array($ch, [
    CURLOPT_URL => 'https://api.example.com/endpoint',
    CURLOPT_SSL_VERIFYPEER => true,   // Never disable in production
    CURLOPT_SSL_VERIFYHOST => 2,
]);

// Validate response structure
function validateApiResponse(array $response): bool {
    if (!isset($response['success']) || !isset($response['data'])) {
        return false;
    }
    return true;
}

// Rate limiting
function checkRateLimit(string $identifier, int $maxRequests, int $windowSeconds): bool {
    $key = 'rate_limit_' . $identifier;
    $cache = Capsule::cache()->get($key);

    if ($cache && $cache['count'] >= $maxRequests) {
        if (time() - $cache['start'] < $windowSeconds) {
            return false; // Rate limited
        }
    }

    // Update cache
    $newCache = [
        'count' => ($cache['count'] ?? 0) + 1,
        'start' => $cache['start'] ?? time(),
    ];
    Capsule::cache()->put($key, $newCache, $windowSeconds);

    return true;
}
```

## Security Checklist Summary

### Input/Output
- [ ] All user input validated
- [ ] Output escaped in templates
- [ ] SQL injection prevented
- [ ] XSS prevented

### Authentication
- [ ] CSRF tokens on all POST
- [ ] Session validation
- [ ] API key storage encrypted

### Payment Security
- [ ] Callback IP validation
- [ ] Signature verification
- [ ] Replay attack prevention
- [ ] No sensitive data in URLs

### General
- [ ] HTTPS enforced
- [ ] SSL verification enabled
- [ ] Error messages don't leak internals
- [ ] Logging doesn't expose secrets

---

**Related Skills:**
- whmcs-testing-qa
- whmcs-validator
- whmcs-gateway-security