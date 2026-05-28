# WHMCS Module Security Checklist Skill
# Version: 1.0 | Updated: 2026-05-28

## Purpose

Comprehensive security checklist for WHMCS module development.

## When to Use

- Before releasing modules
- Security audits
- Code review

## Security Checklist

### Input Validation
```php
// String sanitization
$input = filter_var($_POST['input'], FILTER_SANITIZE_STRING);

// Integer validation
$id = filter_var($_POST['id'], FILTER_VALIDATE_INT);

// Email validation
$email = filter_var($_POST['email'], FILTER_VALIDATE_EMAIL);

// URL validation
$url = filter_var($_POST['url'], FILTER_VALIDATE_URL);
```

### SQL Injection Prevention
```php
// Always use parameterized queries via Capsule
Capsule::table('mod_data')
    ->where('id', $id)  // Safe
    ->whereRaw('name = ?', [$name])  // Parameterized
    ->first();
```

### XSS Prevention
```php
// In templates, always escape
{$variable|escape:'html'}
{$variable|string_format:"%s"}

// In PHP, use htmlspecialchars
echo htmlspecialchars($variable, ENT_QUOTES, 'UTF-8');
```

### CSRF Protection
```php
// Always on POST forms
check_token('WHMCS.admin.default');

// Generate token in forms
// <input type="hidden" name="token" value="{$token}">
```

### File Upload Security
```php
// Validate file uploads
$allowedTypes = ['image/png', 'image/jpeg', 'application/pdf'];
$maxSize = 5 * 1024 * 1024; // 5MB

if (!in_array($_FILES['file']['type'], $allowedTypes)) {
    throw new \Exception('Invalid file type');
}

if ($_FILES['file']['size'] > $maxSize) {
    throw new \Exception('File too large');
}
```

### Password Handling
```php
// Never store passwords in plain text
// Hash before storage
$hashed = password_hash($password, PASSWORD_ARGON2ID);

// Verify with
password_verify($input, $stored);
```

### Session Security
```php
// Regenerate session ID on login
session_regenerate_id(true);

// Set secure session cookies
ini_set('session.cookie_httponly', 1);
ini_set('session.cookie_secure', 1);
```

---

**Related Skills:**
- whmcs-security-hardening
- whmcs-deployment
- whmcs-testing-qa
