# WHMCS Security Best Practices

**Version:** 8.x
**Updated:** 2026-05-28
**Related Skills:** `config-constants-reference`, `performance-optimization`

## Overview

Securing WHMCS requires a multi-layered approach including server configuration, application hardening, access controls, and monitoring. This guide covers comprehensive security measures.

## Server-Level Security

### PHP Configuration

```ini
; php.ini or .user.ini

; Disable dangerous functions
disable_functions = proc_open, popen, exec, system, passthru, shell_exec, info, posix_mkfifo, posix_getpwuid, posix_kill, posix_setpgid, posix_setsid, posix_setuid, posix_getpgid, posix_getpwnam, escapeshellcmd, escapeshellarg, show_source

; Session security
session.cookie_httponly = 1
session.cookie_secure = 1
session.use_strict_mode = 1
session.cookie_samesite = Strict

; File upload security
file_uploads = 0
upload_max_filesize = 0
post_max_size = 0

; Disable remote code execution
allow_url_fopen = 0
allow_url_include = 0

; Error handling (hide from users)
display_errors = 0
display_startup_errors = 0
log_errors = 1
error_log = /var/log/php_errors.log

; Memory and execution
memory_limit = 256M
max_execution_time = 30
max_input_time = 60

; Open base directory
open_basedir = /home/user/public_html:/tmp:/usr/share/php
```

### Web Server Configuration

#### Apache (.htaccess)

```apache
# Prevent access to sensitive files
<FilesMatch "\.(env|log|sql|bak|swp|ini|fla|psd|otf|ttf|key)$">
    Order Allow,Deny
    Deny from all
</FilesMatch>

# Prevent directory browsing
Options -Indexes

# Disable server signature
ServerSignature Off

# Prevent clickjacking
Header always set X-Frame-Options "SAMEORIGIN"

# XSS Protection
Header always set X-XSS-Protection "1; mode=block"

# Prevent MIME sniffing
Header always set X-Content-Type-Options "nosniff"

# Force HTTPS
Header always set Strict-Transport-Security "max-age=31536000"

# Content Security Policy
Header always set Content-Security-Policy "default-src 'self'; script-src 'self' 'unsafe-inline' https://cdn.whmcs.com; style-src 'self' 'unsafe-inline';"

# Block access to admin by IP
<Files "admin/*">
    Order Deny,Allow
    Deny from all
    Allow from 192.168.1.0/24
    Allow from 10.0.0.0/8
</Files>

# Protect configuration files
<Files "configuration.php">
    Order Deny,Allow
    Deny from all
</Files>
```

#### Nginx

```nginx
server {
    # Security headers
    add_header X-Frame-Options "SAMEORIGIN" always;
    add_header X-Content-Type-Options "nosniff" always;
    add_header X-XSS-Protection "1; mode=block" always;
    add_header Strict-Transport-Security "max-age=31536000" always;
    add_header Content-Security-Policy "default-src 'self';" always;

    # Block sensitive files
    location ~ /\.(env|log|sql|bak|swp|ini|fla|psd)$ {
        deny all;
    }

    # Admin access restriction
    location /admin {
        allow 192.168.1.0/24;
        allow 10.0.0.0/8;
        deny all;
    }

    # Protect configuration
    location ~ /(configuration\.php|dbconnect\.php)$ {
        deny all;
    }

    # Disable directory listing
    autoindex off;
}
```

## WHMCS Configuration Security

### Essential Security Constants

```php
// configuration.php

// Generate new encryption key (required for security)
$cc_encryption_hash = bin2hex(random_bytes(32));

// Disable brute force (enable recommended features)
$disable_whmcs_brand = true;

// API restrictions
$api_access_controls = [
    'allowed_ips' => ['192.168.1.0/24', '10.0.0.0/8'],
    'denied_ips' => [],
    'allowed_referers' => ['https://yourdomain.com'],
];

// Session security
$session_validation = [
    'ip_validation' => true,
    'user_agent_validation' => true,
    'regenerate_on_login' => true,
];

// CSRF protection
$csrf_protection = [
    'enabled' => true,
    'token_lifetime' => 7200, // 2 hours
    'regenerate_on_submit' => true,
];

// Two-factor authentication enforcement
$two_factor_required = [
    'admin_users' => true,
    'client_users' => false, // Optional for clients
    'api_access' => true,
];
```

### Advanced Security Configuration

```php
// includes/additional_security.php

// Require 2FA for admin access
add_hook('AdminLogin', 1, function($vars) {
    if (!preg_match('/^[0-9]{6}$/', $_POST['twofa'])) {
        return ['error' => 'Invalid 2FA code'];
    }
    return true;
});

// IP-based access control
add_hook('AdminAreaAccess', 1, function($vars) {
    $allowed_ips = ['192.168.1.0/24'];

    if (!ipInCIDRBlocks($_SERVER['REMOTE_ADDR'], $allowed_ips)) {
        http_response_code(403);
        die('Access Denied');
    }
});

// Rate limiting for API
add_hook('ApiGatekeeper', 1, function($vars) {
    $cache = new \WHMCS\Cache\RedisAdapter();
    $key = 'api_rate_' . md5($vars['api_key']);

    $attempts = $cache->get($key) ?? 0;

    if ($attempts > 60) { // 60 requests per minute
        throw new \Exception('Rate limit exceeded');
    }

    $cache->set($key, $attempts + 1, 60); // TTL: 60 seconds
});
```

## File System Security

### Permissions

```bash
#!/bin/bash
# secure_whmcs.sh

WHMCSPATH="/home/user/public_html/whmcs"

# Set ownership
chown -R user:www-data "$WHMCSPATH"

# Set directory permissions (755)
find "$WHMCSPATH" -type d -exec chmod 755 {} \;

# Set file permissions (644)
find "$WHMCSPATH" -type f -exec chmod 644 {} \;

# Special permissions for specific files
chmod 400 "$WHMCSPATH/configuration.php"
chmod 400 "$WHMCSPATH/includes/dbconnect.php"
chmod 400 "$WHMCSPATH/vendor/autoload.php"

# Directories requiring write access
chmod 775 "$WHMCSPATH/storage"
chmod 775 "$WHMCSPATH/vendor/whmcs/whmcs-core/attachments"
chmod 775 "$WHMCSPATH/vendor/whmcs/whmcs-core/downloads"
chmod 775 "$WHMCSPATH/vendor/whmcs/whmcs-core/templates_c"
```

### Critical Files to Protect

```php
// includes/hooks/file_protection.php

add_hook('AfterApplicationOutput', 1, function($vars) {
    // Check for suspicious file access attempts
    $suspiciousPatterns = [
        '/\.env/',
        '/\.git/',
        '/wp-admin/',
        '/\.htaccess/',
        '/\.htpasswd/',
    ];

    $requestedUri = $_SERVER['REQUEST_URI'] ?? '';

    foreach ($suspiciousPatterns as $pattern) {
        if (strpos($requestedUri, $pattern) !== false) {
            http_response_code(404);
            logActivity('Blocked suspicious access: ' . $requestedUri);
            exit;
        }
    }
});
```

## Database Security

### Secure Database Connection

```php
// configuration.php - Use SSL connection

$db_host = 'localhost';
$db_username = 'whmcs_user';
$db_password = 'secure_password_here';
$db_name = 'whmcs_database';

// MySQL SSL configuration
$db_ssl = [
    'enabled' => true,
    'ca_cert' => '/path/to/ca-cert.pem',
    'client_cert' => '/path/to/client-cert.pem',
    'client_key' => '/path/to/client-key.pem',
];

// PDO options for security
$pdo_options = [
    PDO::ATTR_ERRMODE => PDO::ERRMODE_EXCEPTION,
    PDO::ATTR_EMULATE_PREPARES => false,
    PDO::ATTR_DEFAULT_FETCH_MODE => PDO::FETCH_ASSOC,
];
```

### Database User Permissions

```sql
-- Create dedicated database user with minimal permissions
CREATE USER 'whmcs_user'@'localhost' IDENTIFIED BY 'secure_password';

-- Grant necessary permissions only
GRANT SELECT, INSERT, UPDATE, DELETE ON whmcs_database.*
    TO 'whmcs_user'@'localhost';

-- For migrations/updates, temporarily grant CREATE
GRANT CREATE ON whmcs_database.* TO 'whmcs_user'@'localhost';

-- After updates, revoke CREATE
REVOKE CREATE ON whmcs_database.* FROM 'whmcs_user'@'localhost';

FLUSH PRIVILEGES;
```

## Access Control

### IP Whitelisting for Admin

```php
// includes/hooks/ip_whitelist.php

add_hook('AdminAreaPage', 1, function($vars) {
    $whitelist = [
        '192.168.1.0/24',  // Office network
        '10.0.0.0/8',      // VPN range
        '203.0.113.50',    // Specific admin IP
    ];

    $clientIp = $_SERVER['REMOTE_ADDR'];

    if (!ipInCIDRBlocks($clientIp, $whitelist)) {
        // Log unauthorized access attempt
        logActivity('Unauthorized admin access from: ' . $clientIp);

        // Redirect or show error
        if (!headers_sent()) {
            header('Location: /');
            exit;
        }
    }
});

function ipInCIDRBlocks($ip, $cidrs): bool
{
    foreach ($cidrs as $cidr) {
        if (ipInCIDR($ip, $cidr)) {
            return true;
        }
    }
    return false;
}

function ipInCIDR($ip, $cidr): bool
{
    list($subnet, $mask) = explode('/', $cidr);
    $maskBits = ~((1 << (32 - $mask)) - 1);
    return (ip2long($ip) & $maskBits) == (ip2long($subnet) & $maskBits);
}
```

### API Key Security

```php
// includes/hooks/api_security.php

add_hook('ApiAuthentication', 1, function($vars) {
    $apiKey = $vars['api_key'] ?? '';
    $apiSecret = $vars['api_secret'] ?? '';

    // Verify API key exists and is active
    $apiCredential = Capsule::table('tblapi_access')
        ->where('api_key', $apiKey)
        ->where('active', 1)
        ->first();

    if (!$apiCredential) {
        throw new Exception('Invalid or inactive API key');
    }

    // Verify secret
    if (!password_verify($apiSecret, $apiCredential->api_secret_hash)) {
        throw new Exception('Invalid API secret');
    }

    // Check IP restrictions
    if ($apiCredential->allowed_ips) {
        $allowedIps = explode(',', $apiCredential->allowed_ips);
        if (!in_array($_SERVER['REMOTE_ADDR'], $allowedIps)) {
            throw new Exception('IP not allowed for API access');
        }
    }

    // Update last access time
    Capsule::table('tblapi_access')
        ->where('api_key', $apiKey)
        ->update(['last_access' => date('Y-m-d H:i:s')]);

    return true;
});
```

## Monitoring and Logging

### Security Event Logging

```php
// includes/hooks/security_logging.php

// Log all admin logins
add_hook('AdminLoginSuccess', 1, function($vars) {
    logActivity('Successful admin login', $vars['admin_id']);

    Capsule::table('tblactivity_log')->insert([
        'date' => date('Y-m-d H:i:s'),
        'description' => 'Admin login successful',
        'username' => $vars['username'],
        'ipaddr' => $_SERVER['REMOTE_ADDR'],
        'userid' => $vars['admin_id'],
        'extra' => json_encode([
            'user_agent' => $_SERVER['HTTP_USER_AGENT'] ?? 'Unknown',
        ]),
    ]);
});

// Log failed login attempts
add_hook('AdminLoginFail', 1, function($vars) {
    logActivity('Failed admin login attempt: ' . $vars['username']);

    Capsule::table('tbladmin_logins')->insert([
        'admin_id' => 0,
        'date' => date('Y-m-d H:i:s'),
        'ipaddress' => $_SERVER['REMOTE_ADDR'],
        'success' => 0,
        'username' => $vars['username'],
    ]);
});

// Log sensitive actions
add_hook('AdminClientDeletion', 1, function($vars) {
    logActivity('Client deleted', $vars['client_id']);

    Capsule::table('tblsecurity_log')->insert([
        'timestamp' => date('Y-m-d H:i:s'),
        'admin_id' => $_SESSION['adminid'] ?? 0,
        'action' => 'client_deletion',
        'entity_type' => 'client',
        'entity_id' => $vars['client_id'],
        'ip_address' => $_SERVER['REMOTE_ADDR'],
    ]);
});
```

### Intrusion Detection

```php
// includes/hooks/intrusion_detection.php

class IntrusionDetector
{
    protected $maxAttempts = 5;
    protected $lockoutDuration = 900; // 15 minutes
    protected $suspiciousPatterns = [
        'UNION\s+SELECT',
        'DROP\s+TABLE',
        'EXEC\s*\(',
        'eval\(',
        'base64_decode',
        'shell_exec',
        '../..',
        '/etc/passwd',
        'php://input',
    ];

    public function check(): bool
    {
        // Check request for suspicious patterns
        $this->checkRequestPatterns();

        // Check for brute force attempts
        $this->checkBruteForce();

        // Check for unusual activity
        $this->checkUnusualActivity();

        return true;
    }

    protected function checkRequestPatterns(): void
    {
        $request = file_get_contents('php://input');
        $request .= http_build_query($_GET) . http_build_query($_POST);

        foreach ($this->suspiciousPatterns as $pattern) {
            if (preg_match('/' . $pattern . '/i', $request)) {
                $this->logIntrusion('Suspicious pattern detected', [
                    'pattern' => $pattern,
                    'request' => substr($request, 0, 500),
                ]);
            }
        }
    }

    protected function checkBruteForce(): void
    {
        $ip = $_SERVER['REMOTE_ADDR'];
        $cacheKey = 'brute_force_' . md5($ip);

        $cache = new \WHMCS\Cache\RedisAdapter();
        $attempts = $cache->get($cacheKey) ?? 0;

        if ($attempts > $this->maxAttempts) {
            $this->logIntrusion('Brute force detected', [
                'attempts' => $attempts,
                'ip' => $ip,
            ]);
        }
    }

    protected function logIntrusion(string $type, array $details): void
    {
        Capsule::table('tblsecurity_log')->insert([
            'timestamp' => date('Y-m-d H:i:s'),
            'type' => $type,
            'ip_address' => $_SERVER['REMOTE_ADDR'],
            'details' => json_encode($details),
        ]);

        // Optional: Send alert
        $this->sendAlert($type, $details);
    }

    protected function sendAlert(string $type, array $details): void
    {
        // Send email or Slack notification
        $message = "Security Alert: {$type}\n\nIP: {$_SERVER['REMOTE_ADDR']}\n";
        $message .= "Details: " . json_encode($details, JSON_PRETTY_PRINT);

        mail('security@example.com', 'WHMCS Security Alert', $message);
    }
}

add_hook('AfterApplicationInit', 1, function() {
    $detector = new IntrusionDetector();
    $detector->check();
});
```

## Regular Security Maintenance

### Monthly Security Checklist

```bash
#!/bin/bash
# security_check.sh

LOGFILE="/var/log/whmcs_security.log"

# Check for unauthorized file changes
echo "=== Checking file integrity ===" >> $LOGFILE
find /home/user/public_html/whmcs -type f -name "*.php" -mtime -30 -md5sum > /tmp/checksums.txt
# Compare with known good checksums

# Review failed login attempts
echo "=== Failed Logins ===" >> $LOGFILE
grep "Failed admin login" /home/user/public_html/whmcs/storage/logs/admin.log | tail -20 >> $LOGFILE

# Check for unusual API activity
echo "=== API Activity ===" >> $LOGFILE
# Review API logs for anomalies

# Verify permissions
echo "=== Permission Check ===" >> $LOGFILE
stat -c "%a %n" /home/user/public_html/whmcs/configuration.php >> $LOGFILE

# Email report
cat $LOGFILE | mail -s "WHMCS Security Report" admin@example.com
```

## Best Practices Summary

1. **Always use HTTPS** with valid SSL certificates
2. **Keep WHMCS updated** to latest stable version
3. **Use strong passwords** (16+ characters, complex)
4. **Enable Two-Factor Authentication** for all admin accounts
5. **Implement IP whitelisting** for admin access
6. **Regular security audits** and log reviews
7. **Backup regularly** and test restoration procedures
8. **Use minimal file permissions** (644 files, 755 directories)
9. **Secure database** with SSL and minimal privileges
10. **Monitor continuously** for suspicious activity

## Related Documentation

- [Config Constants Reference](config-constants-reference.md)
- [Performance Optimization](performance-optimization.md)
- [Troubleshooting Guide](troubleshooting-guide.md)
