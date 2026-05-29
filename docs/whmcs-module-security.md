# WHMCS Module Security

## Overview

Security is critical for WHMCS modules. This guide covers best practices for securing your modules.

## Input Validation

### Validate All Input

```php
<?php
function yourmodule_processForm(array $data): array
{
    $errors = [];
    
    // Integer validation
    $id = filter_var($data['id'] ?? 0, FILTER_VALIDATE_INT);
    if ($id === false || $id <= 0) {
        $errors[] = 'Invalid ID';
    }
    
    // Email validation
    $email = filter_var($data['email'] ?? '', FILTER_VALIDATE_EMAIL);
    if ($email === false) {
        $errors[] = 'Invalid email address';
    }
    
    // String validation - whitelist characters
    $name = preg_replace('/[^a-zA-Z0-9\s\-_\.]/', '', $data['name'] ?? '');
    if (strlen($name) > 100) {
        $errors[] = 'Name too long';
    }
    
    // URL validation
    $url = filter_var($data['url'] ?? '', FILTER_VALIDATE_URL);
    if ($url === false) {
        $errors[] = 'Invalid URL';
    }
    
    // Enum validation
    $status = $data['status'] ?? '';
    $allowedStatuses = ['active', 'inactive', 'pending'];
    if (!in_array($status, $allowedStatuses)) {
        $errors[] = 'Invalid status';
    }
    
    if (!empty($errors)) {
        return ['success' => false, 'errors' => $errors];
    }
    
    return ['success' => true, 'data' => compact('id', 'email', 'name', 'url', 'status')];
}
```

## CSRF Protection

### Admin Area CSRF

```php
<?php
function yourmodule_output(array $vars): void
{
    // Check for POST request
    if ($_SERVER['REQUEST_METHOD'] === 'POST') {
        // Verify CSRF token
        if (!check_token('WHMCS.admin.default')) {
            logActivity('CSRF violation in yourmodule');
            echo '<div class="alert alert-danger">Invalid security token. Please refresh and try again.</div>';
            return;
        }
        
        // Process form
        $result = yourmodule_processForm($_POST);
        
        if ($result['success']) {
            redir('module=yourmodule&success=1');
        }
    }
    
    // Display form with token
    echo '<form method="post">';
    echo '<input type="hidden" name="token" value="' . generate_token('WHMCS.admin.default') . '">';
    // Form fields...
    echo '</form>';
}
```

### Client Area CSRF

```php
<?php
function yourmodule_clientarea(array $vars): array
{
    if ($_SERVER['REQUEST_METHOD'] === 'POST') {
        if (!check_token('WHMCS.default')) {
            return [
                'pagetitle' => 'Error',
                'templatefile' => 'error',
                'vars' => ['error' => 'Invalid security token'],
            ];
        }
        
        // Process client form
    }
    
    return [
        'pagetitle' => 'Your Module',
        'templatefile' => 'client',
        'vars' => ['token' => generate_token('WHMCS.default')],
    ];
}
```

## SQL Injection Prevention

### Use Parameterized Queries

```php
<?php
// SAFE: Parameterized query
$id = (int) $_GET['id'];
$user = Capsule::table('tblusers')
    ->where('id', $id)
    ->first();

// SAFE: Raw query with bindings
$results = Capsule::select(
    'SELECT * FROM tblusers WHERE status = ? AND name LIKE ?',
    ['active', '%' . $search . '%']
);

// UNSAFE: String concatenation (DO NOT USE)
$sql = "SELECT * FROM tblusers WHERE id = " . $_GET['id']; // WRONG!
```

### Query Builder Examples

```php
<?php
// Safe insert
Capsule::table('mod_yourmodule_data')->insert([
    'name' => $sanitizedName,
    'email' => $sanitizedEmail,
    'created_at' => date('Y-m-d H:i:s'),
]);

// Safe update
Capsule::table('mod_yourmodule_data')
    ->where('id', $id)
    ->update([
        'name' => $sanitizedName,
        'updated_at' => date('Y-m-d H:i:s'),
    ]);

// Safe delete
Capsule::table('mod_yourmodule_data')
    ->where('id', $id)
    ->delete();
```

## Authentication and Authorization

### Verify Admin Permissions

```php
<?php
function yourmodule_output(array $vars): void
{
    // Check admin is logged in
    if (!function_exists('checkPermission') || !checkPermission('Your Permission')) {
        echo 'Access denied';
        return;
    }
    
    // Or check specific admin role
    $admin = getAdminDetails();
    if ($admin['roleid'] != 1) { // Not full admin
        echo 'Access denied';
        return;
    }
}
```

### Verify Client Ownership

```php
<?php
function yourmodule_clientarea(array $vars): array
{
    $userId = (int) $_SESSION['uid'];
    
    // Verify ownership
    $service = Capsule::table('tblhosting')
        ->where('id', $serviceId)
        ->where('userid', $userId)
        ->first();
    
    if (!$service) {
        return [
            'templatefile' => 'error',
            'vars' => ['error' => 'Service not found'],
        ];
    }
    
    return [
        'templatefile' => 'service',
        'vars' => ['service' => $service],
    ];
}
```

## Secure Data Storage

### Hash Sensitive Data

```php
<?php
// Password hashing
$hashedPassword = password_hash($password, PASSWORD_DEFAULT);

// API key storage
$apiKeyHash = hash('sha256', $apiKey);
Capsule::table('mod_yourmodule_settings')->insert([
    'setting_name' => 'api_key_hash',
    'setting_value' => $apiKeyHash,
]);

// Verify API key
$storedHash = Capsule::table('mod_yourmodule_settings')
    ->where('setting_name', 'api_key_hash')
    ->value('setting_value');

if (hash_equals($storedHash, hash('sha256', $providedKey))) {
    // Valid key
}
```

### Encrypt Sensitive Data

```php
<?php
class SecureStorage {
    private string $key;
    
    public function __construct()
    {
        $this->key = getenv('ENCRYPTION_KEY');
    }
    
    public function encrypt(string $data): string
    {
        $iv = random_bytes(16);
        $encrypted = openssl_encrypt($data, 'aes-256-cbc', $this->key, 0, $iv);
        
        return base64_encode($iv . $encrypted);
    }
    
    public function decrypt(string $data): string
    {
        $decoded = base64_decode($data);
        $iv = substr($decoded, 0, 16);
        $encrypted = substr($decoded, 16);
        
        return openssl_decrypt($encrypted, 'aes-256-cbc', $this->key, 0, $iv);
    }
}
```

## Logging Security Events

```php
<?php
function logSecurityEvent(string $event, array $context): void
{
    Capsule::table('mod_security_log')->insert([
        'event' => $event,
        'admin_id' => $_SESSION['adminid'] ?? null,
        'client_id' => $_SESSION['uid'] ?? null,
        'ip_address' => $_SERVER['REMOTE_ADDR'] ?? '',
        'user_agent' => $_SERVER['HTTP_USER_AGENT'] ?? '',
        'context' => json_encode($context),
        'created_at' => date('Y-m-d H:i:s'),
    ]);
}

// Usage
logSecurityEvent('unauthorized_access', [
    'resource' => 'admin area',
    'attempted_action' => 'delete',
]);
```

## Rate Limiting

```php
<?php
class RateLimiter {
    private int $maxAttempts;
    private int $lockoutMinutes;
    
    public function __construct(int $maxAttempts = 5, int $lockoutMinutes = 15)
    {
        $this->maxAttempts = $maxAttempts;
        $this->lockoutMinutes = $lockoutMinutes;
    }
    
    public function isAllowed(string $identifier): bool
    {
        $record = Capsule::table('mod_rate_limits')
            ->where('identifier', $identifier)
            ->first();
        
        if (!$record) {
            return true;
        }
        
        if ($record->locked_until && strtotime($record->locked_until) > time()) {
            return false;
        }
        
        if ($record->attempts >= $this->maxAttempts) {
            Capsule::table('mod_rate_limits')
                ->where('identifier', $identifier)
                ->update([
                    'locked_until' => date('Y-m-d H:i:s', strtotime("+{$this->lockoutMinutes} minutes")),
                ]);
            return false;
        }
        
        return true;
    }
    
    public function recordAttempt(string $identifier): void
    {
        Capsule::table('mod_rate_limits')
            ->updateOrInsert(
                ['identifier' => $identifier],
                [
                    'attempts' => Capsule::raw('attempts + 1'),
                    'last_attempt' => date('Y-m-d H:i:s'),
                ]
            );
    }
}
```

## Best Practices

1. **Validate all input** - Never trust user data
2. **Use CSRF tokens** - Protect all forms
3. **Parameterize queries** - Prevent SQL injection
4. **Hash sensitive data** - Never store plain-text passwords
5. **Check permissions** - Verify authorization
6. **Log security events** - Track suspicious activity
7. **Rate limit** - Prevent brute force attacks

## Related Documentation

- [WHMCS Module Security Standards](/docs/module-security-standards.md)
- [WHMCS Security Best Practices](/docs/security-best-practices.md)