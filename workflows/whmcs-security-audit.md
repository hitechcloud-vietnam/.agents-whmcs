# WHMCS Module Security Audit Workflow
# Version: 1.0 | Updated: 2026-05-28

## Purpose

Comprehensive security audit for WHMCS modules.

## Steps

### 1. Code Review Checklist

- [ ] No hardcoded credentials
- [ ] All user inputs sanitized
- [ ] SQL queries parameterized
- [ ] Output escaped (XSS prevention)
- [ ] CSRF tokens on all forms
- [ ] File uploads validated
- [ ] Session handling secure

### 2. Input Validation

```php
// Safe input handling
$email = filter_var($_POST['email'], FILTER_VALIDATE_EMAIL);
$id = (int) $_POST['id'];
$name = preg_replace('/[^a-zA-Z0-9 ]/', '', $_POST['name']);
```

### 3. SQL Injection Prevention

```php
// Bad
Capsule::table('users')
    ->whereRaw("name = '$name'")
    ->get();

// Good
Capsule::table('users')
    ->where('name', $name)
    ->get();
```

### 4. File Upload Security

```php
function validateUpload(array $file): bool {
    // Check file type
    $allowedTypes = ['image/png', 'image/jpeg', 'application/pdf'];
    if (!in_array($file['type'], $allowedTypes)) {
        return false;
    }

    // Check file size
    if ($file['size'] > 5 * 1024 * 1024) {
        return false;
    }

    // Check file extension
    $ext = pathinfo($file['name'], PATHINFO_EXTENSION);
    $allowedExt = ['png', 'jpg', 'pdf'];
    if (!in_array($ext, $allowedExt)) {
        return false;
    }

    return true;
}
```

### 5. API Key Security

```php
// Encrypt API keys
$encrypted = Crypt::encrypt($apiKey);

// Decrypt for use
$apiKey = Crypt::decrypt($encrypted);

// Never log secrets
logActivity('API call made'); // Good
logActivity("API Key: $apiKey"); // Bad
```

### 6. Output

Complete security audit report with:
- Vulnerability assessment
- Security recommendations
- Remediation steps
