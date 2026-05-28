# WHMCS Security Audit Checklist
# Version: 1.0 | Updated: 2026-05-28

## Purpose

Comprehensive security audit checklist for WHMCS modules and installations.

## Module Security

### Code Review Checklist

```
Input Validation:
□ All user inputs are validated
□ SQL queries use parameterized statements
□ File uploads are validated and sanitized
□ URLs are validated before use
□ Email addresses are properly validated
□ Phone numbers are validated

Output Encoding:
□ HTML output is escaped (htmlspecialchars)
□ JavaScript output is escaped or encoded
□ URL parameters are properly encoded
□ JSON responses are properly formatted
□ SQL outputs are not directly displayed

Authentication & Authorization:
□ Admin actions require permission checks
□ Client actions verify ownership
□ API endpoints require authentication
□ Session tokens are properly validated
□ Passwords are not logged or exposed

Data Protection:
□ Sensitive data is encrypted at rest
□ API keys are stored encrypted
□ Customer data is not exposed in logs
□ File permissions are correct
□ Database backups are secured
```

### SQL Injection Prevention

```php
// Bad - Vulnerable to SQL injection
$userId = $_GET['id'];
$query = "SELECT * FROM tblclients WHERE id = $userId";
$result = Capsule::select($query);

// Good - Parameterized query
$userId = (int) $_GET['id'];
$result = Capsule::table('tblclients')
    ->where('id', $userId)
    ->first();

// Good - Using bindings
$email = $_GET['email'];
$result = Capsule::select(
    "SELECT * FROM tblclients WHERE email = ?",
    [$email]
);
```

### XSS Prevention

```php
// Bad - Vulnerable to XSS
echo "<div>" . $_POST['name'] . "</div>";

// Good - Output is escaped
echo "<div>" . htmlspecialchars($_POST['name'], ENT_QUOTES, 'UTF-8') . "</div>";

// Good - Using WHMCS template system
$smarty->assign('client_name', $client->name);
// In template: {$client_name|escape:'html'}
```

## Server Security

### File Permissions
```bash
# Correct permissions
chmod 755 /var/www/whmcs/modules/servers/     # Directories
chmod 644 /var/www/whmcs/modules/servers/*/*.php # Files

# Remove execute permission from config
chmod 640 /var/www/whmcs/configuration.php

# Set owner correctly
chown -R www-data:www-data /var/www/whmcs/modules/
```

### PHP Configuration
```ini
; php.ini recommendations
display_errors = Off
log_errors = On
error_log = /var/log/php_errors.log
expose_php = Off
allow_url_fopen = Off (if not needed)

; Session security
session.cookie_httponly = 1
session.cookie_secure = 1
session.use_strict_mode = 1

; File upload limits
upload_max_filesize = 2M
max_file_uploads = 20
```

## Database Security

```sql
-- Use least privilege principle
GRANT SELECT, INSERT, UPDATE, DELETE ON whmcs_db.* TO 'whmcs_user'@'localhost';

-- Restrict module access
GRANT SELECT, INSERT, UPDATE, DELETE ON whmcs_db.mod_mymodule.* TO 'module_user'@'localhost';

-- Regular password rotation
ALTER USER 'whmcs_user'@'localhost' IDENTIFIED BY 'new_strong_password';
FLUSH PRIVILEGES;
```

## API Security

```php
<?php
// API authentication
function validateApiKey(string $apiKey): ?array {
    $apiKey = hash('sha256', $apiKey);

    return Capsule::table('mod_api_keys')
        ->where('key_hash', $apiKey)
        ->where('is_active', 1)
        ->where('expires_at', '>', date('Y-m-d H:i:s'))
        ->first();
}

// Rate limiting
function checkRateLimit(string $identifier, int $limit = 100): bool {
    $key = "rate_limit_{$identifier}";
    $count = (int) Capsule::table('mod_rate_limits')
        ->where('key', $key)
        ->where('window_start', '>', date('Y-m-d H:i:s', strtotime('-1 hour')))
        ->value('count');

    if ($count >= $limit) {
        return false;
    }

    Capsule::table('mod_rate_limits')->updateOrInsert(
        ['key' => $key],
        ['count' => $count + 1, 'window_start' => date('Y-m-d H:i:s')]
    );

    return true;
}
```

## Payment Gateway Security

```php
<?php
// PCI DSS compliance checklist for gateways

// Never store full card numbers
// Bad: Capsule::table('mod_payments')->insert(['card' => '4111111111111111']);
// Good: Store only last 4 digits
Capsule::table('mod_payments')->insert(['card_last4' => '1111']);

// Validate payment responses
function validateGatewayResponse(array $data, string $secret): bool {
    $signature = $data['signature'] ?? '';
    unset($data['signature']);

    ksort($data);
    $payload = http_build_query($data);

    $expected = hash_hmac('sha256', $payload, $secret);

    return hash_equals($expected, $signature);
}

// IP whitelisting for callbacks
function validateCallbackIP(): bool {
    $allowedIPs = [
        '203.0.113.0',
        '198.51.100.0',
    ];

    $clientIP = $_SERVER['REMOTE_ADDR'];

    return in_array($clientIP, $allowedIPs);
}
```

## Audit Logging

```php
<?php
// Comprehensive audit logging
function auditLog(string $action, array $context = []): void {
    $log = [
        'timestamp' => date('Y-m-d H:i:s'),
        'user_id' => $_SESSION['uid'] ?? null,
        'admin_id' => $_SESSION['adminid'] ?? null,
        'action' => $action,
        'ip_address' => $_SERVER['REMOTE_ADDR'],
        'user_agent' => $_SERVER['HTTP_USER_AGENT'] ?? '',
        'context' => $context,
    ];

    Capsule::table('mod_audit_log')->insert($log);

    // Also log to system
    logActivity("AUDIT: {$action} - " . json_encode($context));
}

// Audit sensitive actions
add_hook('AfterModuleCreate', 1, function($vars) {
    auditLog('module_create', [
        'service_id' => $vars['serviceid'],
        'domain' => $vars['domain'],
    ]);
});
```

## Regular Security Tasks

```bash
#!/bin/bash
# security-audit.sh

echo "=== WHMCS Security Audit ==="
echo ""

# Check file integrity
echo "1. Checking file integrity..."
find /var/www/whmcs/modules -name "*.php" -exec md5sum {} \; > /tmp/current_hashes.txt
diff /tmp/baseline_hashes.txt /tmp/current_hashes.txt || echo "Changes detected!"

# Check permissions
echo "2. Checking permissions..."
find /var/www/whmcs -name "*.php" -perm 777 -ls 2>/dev/null || echo "No world-writable PHP files"

# Check for suspicious files
echo "3. Scanning for suspicious files..."
find /var/www/whmcs -name "*.php" -mtime -1 -ls 2>/dev/null

# Check failed login attempts
echo "4. Checking failed logins..."
grep "Failed login" /var/www/whmcs/logs/*.log | tail -20

# Check database for anomalies
echo "5. Database checks..."
mysql -u whmcs_user -p whmcs_db -e "SELECT COUNT(*) as cnt FROM tblactivitylog WHERE date > DATE_SUB(NOW(), INTERVAL 1 HOUR)"
```

---

**Related Skills:**
- whmcs-security-hardening
- whmcs-gdpr-compliance
- whmcs-audit-log