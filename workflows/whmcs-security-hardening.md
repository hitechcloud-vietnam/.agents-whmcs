# WHMCS Security Hardening Workflow

## Purpose

Comprehensive guide to hardening WHMCS installations against common security threats, including server configuration, file permissions, access controls, and monitoring.

## Prerequisites

- WHMCS installation with root/admin access
- Server SSH access
- Understanding of web server configuration
- SSL certificate for HTTPS
- Backup access

## Workflow Steps

### Step 1: File System Security

Configure secure file permissions:

```bash
#!/bin/bash
# whmcs_secure.sh - Apply security permissions

WHMCS_ROOT="/var/www/whmcs"
WEB_USER="www-data"

# Set ownership
chown -R $WEB_USER:$WEB_USER $WHMCS_ROOT

# Configuration files - read-only
chmod 400 $WHMCS_ROOT/includes/config.php
chmod 400 $WHMCS_ROOT/includes/dbconnect.php

# Sensitive directories
chmod 550 $WHMCS_ROOT/includes
chmod 550 $WHMCS_ROOT/vendor
chmod 550 $WHMCS_ROOT/storage

# Upload directories - writable only
chmod -R 755 $WHMCS_ROOT/downloads
chmod -R 755 $WHMCS_ROOT/templates_c
chmod -R 755 $WHMCS_ROOT/cache

# PHP files - readable only
find $WHMCS_ROOT -name "*.php" -exec chmod 440 {} \;

# Allow template modification
chown -R $WEB_USER:$WEB_USER $WHMCS_ROOT/templates/*/header.html.twig
chmod 660 $WHMCS_ROOT/templates/*/header.html.twig
```

### Step 2: PHP Configuration Hardening

Secure PHP settings:

```ini
; /etc/php/8.1/fpm/php.ini or .user.ini

; Disable dangerous functions
disable_functions = exec,passthru,shell_exec,system,proc_open,popen,curl_exec,curl_multi_exec,parse_ini_file,show_source

; Restrict filesystem access
open_basedir = /var/www/whmcs:/tmp:/usr/share/php:/var/www/whmcs/storage

; Session security
session.cookie_httponly = 1
session.cookie_secure = 1
session.use_strict_mode = 1
session.gc_maxlifetime = 1800

; Upload security
file_uploads = 0
upload_max_filesize = 0
post_max_size = 0

; Disable remote code execution
allow_url_fopen = 0
allow_url_include = 0

; Error handling - hide from users
display_errors = 0
display_startup_errors = 0
log_errors = 1
error_reporting = E_ALL & ~E_DEPRECATED & ~E_STRICT

; Limit resource usage
max_execution_time = 30
max_input_time = 30
memory_limit = 128M
```

### Step 3: Web Server Configuration

Nginx secure configuration:

```nginx
# /etc/nginx/sites-available/whmcs

server {
    listen 443 ssl http2;
    server_name whmcs.example.com;
    
    root /var/www/whmcs;
    index index.php;
    
    # SSL Configuration
    ssl_certificate /etc/ssl/certs/whmcs.crt;
    ssl_certificate_key /etc/ssl/private/whmcs.key;
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256;
    ssl_prefer_server_ciphers off;
    
    # Security headers
    add_header X-Frame-Options "SAMEORIGIN" always;
    add_header X-Content-Type-Options "nosniff" always;
    add_header X-XSS-Protection "1; mode=block" always;
    add_header Referrer-Policy "strict-origin-when-cross-origin" always;
    add_header Content-Security-Policy "default-src 'self'; script-src 'self' 'unsafe-inline'; style-src 'self' 'unsafe-inline';" always;
    
    # Block sensitive files
    location ~ /\. {
        deny all;
    }
    
    location ~* \.(env|log|config\.php|dbconnect\.php)$ {
        deny all;
    }
    
    # PHP processing
    location ~ \.php$ {
        include snippets/fastcgi-php.conf;
        fastcgi_pass unix:/var/run/php/php8.1-fpm.sock;
        fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
        include fastcgi_params;
        
        # Security limits
        fastcgi_read_timeout 30;
        fastcgi_buffer_size 128k;
        fastcgi_buffers 256 16k;
    }
    
    # Deny access to sensitive directories
    location ~ /(vendor|includes|storage|resources)/ {
        deny all;
    }
}
```

### Step 4: WHMCS Configuration Hardening

Configure WHMCS security settings:

```php
// includes/config.php additions

// Force HTTPS
define('FORCESSL', true);

// Disable frontend browser caching
define('DISABLE_CACHE_HEADERS', true);

// API restrictions
define('API_IP_ACCESS_RESTRICTION', '1.2.3.4/24'); // Your IP range

// Admin security
$admin_folder = 'admin_' . substr(md5($cc_encryption_hash), 0, 8);
define('ADMIN_FOLDER', $admin_folder);

// Disable XML-RPC
define('DISABLE_XMLRPC', true);

// Disable pingback
define('DISABLE_PINGBACK', true);

// Session security
ini_set('session.cookie_httponly', 1);
ini_set('session.cookie_secure', 1);
ini_set('session.use_strict_mode', 1);

// Disable error display
ini_set('display_errors', 0);
error_reporting(0);
```

### Step 5: Database Security

Secure database configuration:

```sql
-- Create dedicated WHMCS database user
CREATE USER 'whmcs_user'@'localhost' IDENTIFIED BY 'strong_random_password';
GRANT SELECT, INSERT, UPDATE, DELETE ON whmcs_database.* TO 'whmcs_user'@'localhost';
FLUSH PRIVILEGES;

-- Enable encryption at rest (MySQL 8.0+)
ALTER TABLE tblclients ENCRYPTION='Y';
ALTER TABLE tblaccounts ENCRYPTION='Y';

-- Set up audit logging
CREATE TABLE security_audit_log (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    event_type VARCHAR(100) NOT NULL,
    user_id INT,
    ip_address VARCHAR(45),
    user_agent TEXT,
    details JSON,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    INDEX idx_created (created_at),
    INDEX idx_user (user_id),
    INDEX idx_event (event_type)
);
```

### Step 6: Access Control Implementation

Implement IP whitelisting and access controls:

```php
// includes/security/access_control.php

class AccessControl
{
    private static $whitelist = [];
    private static $blacklist = [];
    
    /**
     * Initialize access control lists
     */
    public static function init(): void
    {
        self::$whitelist = self::loadWhitelist();
        self::$blacklist = self::loadBlacklist();
    }
    
    /**
     * Validate admin access
     */
    public static function validateAdminAccess(): bool
    {
        self::init();
        
        $ip = $_SERVER['REMOTE_ADDR'] ?? '';
        
        // Check blacklist first
        if (self::isBlacklisted($ip)) {
            self::logAccess($ip, 'blocked_blacklist');
            return false;
        }
        
        // Check whitelist (if configured)
        if (!empty(self::$whitelist) && !self::isWhitelisted($ip)) {
            self::logAccess($ip, 'not_whitelisted');
            return false;
        }
        
        self::logAccess($ip, 'allowed');
        return true;
    }
    
    /**
     * Check if IP is blacklisted
     */
    private static function isBlacklisted(string $ip): bool
    {
        foreach (self::$blacklist as $rule) {
            if (self::ipMatchesRule($ip, $rule)) {
                return true;
            }
        }
        return false;
    }
    
    /**
     * Check if IP is whitelisted
     */
    private static function isWhitelisted(string $ip): bool
    {
        foreach (self::$whitelist as $rule) {
            if (self::ipMatchesRule($ip, $rule)) {
                return true;
            }
        }
        return false;
    }
    
    /**
     * Match IP against rule (supports CIDR)
     */
    private static function ipMatchesRule(string $ip, string $rule): bool
    {
        // Exact match
        if ($ip === $rule) {
            return true;
        }
        
        // CIDR notation
        if (strpos($rule, '/') !== false) {
            [$subnet, $mask] = explode('/', $rule);
            return (ip2long($ip) & ~((1 << (32 - $mask)) - 1)) === ip2long($subnet);
        }
        
        return false;
    }
    
    private static function loadWhitelist(): array
    {
        $config = Capsule::table('tblconfiguration')
            ->where('setting', 'admin_ip_whitelist')
            ->value('value');
        
        return $config ? array_filter(array_map('trim', explode(',', $config))) : [];
    }
    
    private static function loadBlacklist(): array
    {
        $config = Capsule::table('tblconfiguration')
            ->where('setting', 'admin_ip_blacklist')
            ->value('value');
        
        return $config ? array_filter(array_map('trim', explode(',', $config))) : [];
    }
    
    private static function logAccess(string $ip, string $status): void
    {
        Capsule::table('security_audit_log')->insert([
            'event_type' => 'admin_access_' . $status,
            'ip_address' => $ip,
            'user_agent' => $_SERVER['HTTP_USER_AGENT'] ?? '',
            'details' => json_encode(['path' => $_SERVER['REQUEST_URI'] ?? '']),
            'created_at' => date('Y-m-d H:i:s'),
        ]);
    }
}

/**
 * Admin access hook
 */
add_hook('AdminAreaPage', 1, function($vars) {
    if (!AccessControl::validateAdminAccess()) {
        header('HTTP/1.1 403 Forbidden');
        die('Access denied from your IP address.');
    }
});
```

### Step 7: Two-Factor Authentication

Enforce 2FA for admin accounts:

```php
// Enforce 2FA for all admin users
add_hook('AdminLogin', 1, function($vars) {
    // Get admin email
    $admin = Capsule::table('tbladminlogins')
        ->where('id', $_SESSION['adminid'])
        ->first();
    
    // Check if 2FA is enabled
    $twoFactorEnabled = Capsule::table('tbladmin_security')
        ->where('admin_id', $_SESSION['adminid'])
        ->where('two_factor_enabled', 1)
        ->exists();
    
    if (!$twoFactorEnabled) {
        // Force 2FA setup
        return [
            'redirect' => 'security.php?force_setup=1',
        ];
    }
});

/**
 * Generate backup codes
 */
function generateBackupCodes(int $adminId, int $count = 10): array
{
    $codes = [];
    
    for ($i = 0; $i < $count; $i++) {
        $code = strtoupper(substr(md5(uniqid(mt_rand(), true)), 0, 8));
        $codes[] = $code;
    }
    
    // Store hashed codes
    Capsule::table('tbladmin_backup_codes')->where('admin_id', $adminId)->delete();
    
    foreach ($codes as $code) {
        Capsule::table('tbladmin_backup_codes')->insert([
            'admin_id' => $adminId,
            'code_hash' => password_hash($code, PASSWORD_DEFAULT),
            'used' => 0,
            'created_at' => date('Y-m-d H:i:s'),
        ]);
    }
    
    return $codes; // Return plaintext for user to save
}
```

### Step 8: Security Monitoring

Set up security event monitoring:

```php
// includes/security/monitor.php

class SecurityMonitor
{
    /**
     * Log security events
     */
    public static function log(string $event, array $context = []): void
    {
        Capsule::table('security_events')->insert([
            'event_type' => $event,
            'severity' => self::determineSeverity($event),
            'ip_address' => $_SERVER['REMOTE_ADDR'] ?? '',
            'user_id' => $_SESSION['uid'] ?? null,
            'admin_id' => $_SESSION['adminid'] ?? null,
            'context' => json_encode($context),
            'user_agent' => $_SERVER['HTTP_USER_AGENT'] ?? '',
            'created_at' => date('Y-m-d H:i:s'),
        ]);
    }
    
    private static function determineSeverity(string $event): string
    {
        $critical = [
            'login_failed', 'admin_login_failed', 'brute_force_detected',
            'sql_injection_attempt', 'xss_attempt', 'csrf_attempt',
        ];
        
        $high = [
            'privilege_escalation', 'data_exfiltration', 'unauthorized_access',
        ];
        
        if (in_array($event, $critical)) return 'critical';
        if (in_array($event, $high)) return 'high';
        
        return 'medium';
    }
}

/**
 * Monitor failed login attempts
 */
add_hook('LoginFail', 1, function($vars) {
    $ip = $_SERVER['REMOTE_ADDR'] ?? '';
    
    // Count recent failures
    $recentFailures = Capsule::table('security_events')
        ->where('event_type', 'login_failed')
        ->where('ip_address', $ip)
        ->where('created_at', '>', date('Y-m-d H:i:s', strtotime('-15 minutes')))
        ->count();
    
    if ($recentFailures >= 5) {
        SecurityMonitor::log('brute_force_detected', [
            'failures' => $recentFailures,
        ]);
        
        // Add to temporary blacklist
        Capsule::table('security_temp_blacklist')->insert([
            'ip_address' => $ip,
            'expires_at' => date('Y-m-d H:i:s', strtotime('+1 hour')),
        ]);
    }
});

/**
 * Cron job to clean up expired blocks
 */
add_hook('DailyCronJob', 1, function($vars) {
    Capsule::table('security_temp_blacklist')
        ->where('expires_at', '<', date('Y-m-d H:i:s'))
        ->delete();
});
```

## Best Practices

1. **Regular security audits** - Monthly vulnerability scans
2. **Keep software updated** - WHMCS, PHP, database, server
3. **Use strong passwords** - Minimum 16 characters, complex
4. **Enable 2FA everywhere** - Admin, staff, and client accounts
5. **Limit login attempts** - Block after failed attempts
6. **Rotate credentials** - Change API keys and passwords regularly
7. **Monitor access logs** - Watch for suspicious patterns
8. **Backup regularly** - Test restoration procedures

## Common Pitfalls to Avoid

1. **Default admin credentials** - Always change default passwords
2. **Leaving debug mode on** - Exposes sensitive information
3. **Overly permissive permissions** - Principle of least privilege
4. **Ignoring SSL certificates** - Use HTTPS everywhere
5. **Not logging security events** - You won't know what's happening
6. **Skipping 2FA** - Major security gap
7. **Shared credentials** - Each user should have own account
8. **Unpatched software** - Keep everything updated
