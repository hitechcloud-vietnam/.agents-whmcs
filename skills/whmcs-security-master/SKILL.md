# WHMCS Security Master

## Overview
Master skill for WHMCS security implementation. Covers security best practices, input validation, authentication, and protection mechanisms.

## Security Hooks

```php
<?php
// /includes/hooks/security_hooks.php

/**
 * IP-based access control
 */
add_hook('ClientLogin', 1, function(array $params) {
    $client = \WHMCS\User\Client::find($params['userid']);
    $currentIp = $_SERVER['REMOTE_ADDR'];

    // Check if IP is allowed
    $allowedIps = $client->allowed_ips ? json_decode($client->allowed_ips, true) : [];

    if (!empty($allowedIps) && !in_array($currentIp, $allowedIps)) {
        // Log suspicious login
        logActivity("Blocked login attempt from unauthorized IP: {$currentIp} for client: {$client->email}");

        // Send alert email
        send_email('SecurityAlert', $client->id, [
            'ip' => $currentIp,
            'location' => getIpLocation($currentIp),
        ]);

        return false;
    }

    return true;
});

/**
 * Login attempt limiting
 */
add_hook('ClientLogin', 1, function(array $params) {
    $ip = $_SERVER['REMOTE_ADDR'];
    $key = 'login_attempts_' . md5($ip);

    $attempts = (int) \Illuminate\Database\Capsule\Manager::table('mod_security_log')
        ->where('ip_address', $ip)
        ->where('created_at', '>=', date('Y-m-d H:i:s', strtotime('-15 minutes')))
        ->count();

    if ($attempts >= 5) {
        // Block the login
        return [
            'success' => false,
            'error' => 'Too many login attempts. Please try again in 15 minutes.',
        ];
    }

    return true;
});

/**
 * Log all admin actions
 */
add_hook('AdminAreaPage', 1, function(array $params) {
    if ($params['action'] === 'logout') {
        return;
    }

    $admin = \App::getFromRequest('adminid');

    if ($admin) {
        \Illuminate\Database\Capsule\Manager::table('mod_security_admin_log')
            ->insert([
                'admin_id' => $admin,
                'action' => $params['action'],
                'page' => $params['filename'],
                'ip_address' => $_SERVER['REMOTE_ADDR'],
                'user_agent' => $_SERVER['HTTP_USER_AGENT'] ?? '',
                'created_at' => date('Y-m-d H:i:s'),
            ]);
    }
});

/**
 * CSRF protection
 */
add_hook('ClientAreaPage', 1, function(array $params) {
    // Generate CSRF token if not exists
    if (!isset($_SESSION['csrf_token'])) {
        $_SESSION['csrf_token'] = bin2hex(random_bytes(32));
    }

    return true;
});

/**
 * Validate CSRF token on form submission
 */
function validateCsrfToken(string $token): bool
{
    return hash_equals($_SESSION['csrf_token'] ?? '', $token);
}

/**
 * XSS protection - sanitize output
 */
function sanitizeOutput(string $input): string
{
    return htmlspecialchars($input, ENT_QUOTES | ENT_HTML5, 'UTF-8');
}

/**
 * SQL injection prevention - use parameterized queries
 */
function safeQuery(string $table, array $data, string $whereField, $whereValue): array
{
    return \Illuminate\Database\Capsule\Manager::table($table)
        ->where($whereField, $whereValue)
        ->first();
}
```

## Security Module Class

```php
<?php
// /modules/addons/SecurityModule/SecurityModule.php

if (!defined("WHMCS")) {
    die("This file cannot be accessed directly");
}

function SecurityModule_config()
{
    return [
        'name' => 'Security Module',
        'description' => 'Enhanced security features for WHMCS',
        'version' => '1.0.0',
        'fields' => [
            'enableRateLimiting' => [
                'FriendlyName' => 'Enable Rate Limiting',
                'Type' => 'yesno',
                'Default' => 'on',
            ],
            'maxLoginAttempts' => [
                'FriendlyName' => 'Max Login Attempts',
                'Type' => 'text',
                'Default' => '5',
            ],
            'lockoutDuration' => [
                'FriendlyName' => 'Lockout Duration (minutes)',
                'Type' => 'text',
                'Default' => '15',
            ],
            'enable2FA' => [
                'FriendlyName' => 'Force 2FA for Admins',
                'Type' => 'yesno',
                'Default' => 'on',
            ],
            'ipWhitelist' => [
                'FriendlyName' => 'Admin IP Whitelist',
                'Type' => 'textarea',
                'Description' => 'One IP per line',
            ],
            'enableAuditLog' => [
                'FriendlyName' => 'Enable Audit Logging',
                'Type' => 'yesno',
                'Default' => 'on',
            ],
            'blockedCountries' => [
                'FriendlyName' => 'Blocked Countries',
                'Type' => 'text',
                'Description' => 'Comma-separated country codes',
            ],
        ],
    ];
}

function SecurityModule_activate()
{
    // Create security log table
    \Illuminate\Database\Capsule\Manager::statement("
        CREATE TABLE IF NOT EXISTS `mod_security_log` (
            `id` INT AUTO_INCREMENT PRIMARY KEY,
            `event_type` VARCHAR(50) NOT NULL,
            `user_id` INT DEFAULT NULL,
            `ip_address` VARCHAR(45) NOT NULL,
            `user_agent` VARCHAR(500) DEFAULT NULL,
            `details` TEXT,
            `created_at` DATETIME DEFAULT CURRENT_TIMESTAMP,
            INDEX `idx_ip` (`ip_address`),
            INDEX `idx_user` (`user_id`),
            INDEX `idx_event` (`event_type`)
        ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4
    ");

    // Create blocked IPs table
    \Illuminate\Database\Capsule\Manager::statement("
        CREATE TABLE IF NOT EXISTS `mod_security_blocked_ips` (
            `id` INT AUTO_INCREMENT PRIMARY KEY,
            `ip_address` VARCHAR(45) NOT NULL UNIQUE,
            `reason` VARCHAR(255) DEFAULT NULL,
            `expires_at` DATETIME DEFAULT NULL,
            `created_at` DATETIME DEFAULT CURRENT_TIMESTAMP
        ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4
    ");

    return ['status' => 'success'];
}

function SecurityModule_deactivate()
{
    return ['status' => 'success'];
}

/**
 * Check if IP is blocked
 */
function isIpBlocked(string $ip): bool
{
    $blocked = \Illuminate\Database\Capsule\Manager::table('mod_security_blocked_ips')
        ->where('ip_address', $ip)
        ->where(function($query) {
            $query->whereNull('expires_at')
                ->orWhere('expires_at', '>', date('Y-m-d H:i:s'));
        })
        ->first();

    return !empty($blocked);
}

/**
 * Block an IP address
 */
function blockIp(string $ip, string $reason = '', ?DateTime $expires = null): bool
{
    return \Illuminate\Database\Capsule\Manager::table('mod_security_blocked_ips')
        ->updateOrInsert(
            ['ip_address' => $ip],
            [
                'reason' => $reason,
                'expires_at' => $expires ? $expires->format('Y-m-d H:i:s') : null,
                'created_at' => date('Y-m-d H:i:s'),
            ]
        );
}

/**
 * Unblock an IP address
 */
function unblockIp(string $ip): bool
{
    return \Illuminate\Database\Capsule\Manager::table('mod_security_blocked_ips')
        ->where('ip_address', $ip)
        ->delete() > 0;
}

/**
 * Log security event
 */
function logSecurityEvent(string $eventType, ?int $userId = null, ?string $ip = null, ?string $details = null): void
{
    \Illuminate\Database\Capsule\Manager::table('mod_security_log')
        ->insert([
            'event_type' => $eventType,
            'user_id' => $userId,
            'ip_address' => $ip ?? $_SERVER['REMOTE_ADDR'] ?? 'unknown',
            'user_agent' => $_SERVER['HTTP_USER_AGENT'] ?? '',
            'details' => $details,
            'created_at' => date('Y-m-d H:i:s'),
        ]);
}

/**
 * Get IP reputation
 */
function getIpReputation(string $ip): array
{
    // Check local blocklist
    if (isIpBlocked($ip)) {
        return [
            'blocked' => true,
            'reason' => 'Blocked locally',
            'score' => 0,
        ];
    }

    // Check for suspicious patterns
    $recentAttempts = \Illuminate\Database\Capsule\Manager::table('mod_security_log')
        ->where('ip_address', $ip)
        ->where('created_at', '>=', date('Y-m-d H:i:s', strtotime('-1 hour')))
        ->count();

    if ($recentAttempts > 50) {
        return [
            'blocked' => false,
            'suspicious' => true,
            'reason' => 'High request volume',
            'score' => 30,
        ];
    }

    return [
        'blocked' => false,
        'suspicious' => false,
        'score' => 100,
    ];
}
```

## Password Security

```php
<?php
// /includes/security/password_functions.php

/**
 * Hash password using bcrypt
 */
function hashPassword(string $password): string
{
    return password_hash($password, PASSWORD_BCRYPT, [
        'cost' => 12,
    ]);
}

/**
 * Verify password
 */
function verifyPassword(string $password, string $hash): bool
{
    return password_verify($password, $hash);
}

/**
 * Check if password needs rehashing
 */
function needsRehash(string $hash): bool
{
    return password_needs_rehash($hash, PASSWORD_BCRYPT, ['cost' => 12]);
}

/**
 * Validate password strength
 */
function validatePasswordStrength(string $password): array
{
    $errors = [];

    if (strlen($password) < 8) {
        $errors[] = 'Password must be at least 8 characters';
    }

    if (!preg_match('/[A-Z]/', $password)) {
        $errors[] = 'Password must contain at least one uppercase letter';
    }

    if (!preg_match('/[a-z]/', $password)) {
        $errors[] = 'Password must contain at least one lowercase letter';
    }

    if (!preg_match('/[0-9]/', $password)) {
        $errors[] = 'Password must contain at least one number';
    }

    if (!preg_match('/[^A-Za-z0-9]/', $password)) {
        $errors[] = 'Password must contain at least one special character';
    }

    return $errors;
}

/**
 * Generate secure random password
 */
function generateSecurePassword(int $length = 16): string
{
    $chars = 'abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ0123456789!@#$%^&*()_+-=';

    $password = '';
    $max = strlen($chars) - 1;

    for ($i = 0; $i < $length; $i++) {
        $password .= $chars[random_int(0, $max)];
    }

    return $password;
}

/**
 * Check password against common breaches
 */
function isPasswordBreached(string $password): bool
{
    $hash = strtoupper(sha1($password));
    $prefix = substr($hash, 0, 5);
    $suffix = substr($hash, 5);

    $ch = curl_init("https://api.pwnedpasswords.com/range/{$prefix}");
    curl_setopt_array($ch, [
        CURLOPT_RETURNTRANSFER => true,
        CURLOPT_FOLLOWLOCATION => true,
    ]);

    $response = curl_exec($ch);
    curl_close($ch);

    if (!$response) {
        return false;
    }

    $hashes = explode("\r\n", $response);
    foreach ($hashes as $hashLine) {
        [$hashSuffix, $count] = explode(':', $hashLine);
        if (strtoupper($hashSuffix) === $suffix) {
            return true;
        }
    }

    return false;
}
```

## Input Validation

```php
<?php
// /includes/security/validation.php

class Validation
{
    /**
     * Sanitize email address
     */
    public static function sanitizeEmail(string $email): string
    {
        $email = filter_var($email, FILTER_SANITIZE_EMAIL);
        return filter_var($email, FILTER_VALIDATE_EMAIL) ? $email : '';
    }

    /**
     * Sanitize string
     */
    public static function sanitizeString(string $input): string
    {
        return htmlspecialchars(strip_tags(trim($input)), ENT_QUOTES, 'UTF-8');
    }

    /**
     * Sanitize integer
     */
    public static function sanitizeInt($input): int
    {
        return (int) filter_var($input, FILTER_SANITIZE_NUMBER_INT);
    }

    /**
     * Sanitize URL
     */
    public static function sanitizeUrl(string $url): string
    {
        $url = filter_var($url, FILTER_SANITIZE_URL);

        if (!filter_var($url, FILTER_VALIDATE_URL)) {
            return '';
        }

        // Only allow http and https
        $parsed = parse_url($url);
        if (!isset($parsed['scheme']) || !in_array($parsed['scheme'], ['http', 'https'])) {
            return '';
        }

        return $url;
    }

    /**
     * Validate domain name
     */
    public static function validateDomain(string $domain): bool
    {
        return filter_var($domain, FILTER_VALIDATE_DOMAIN, FILTER_FLAG_HOSTNAME) !== false;
    }

    /**
     * Validate phone number
     */
    public static function validatePhone(string $phone): bool
    {
        // E.164 format
        return preg_match('/^\+?[1-9]\d{1,14}$/', preg_replace('/[^\d+]/', '', $phone)) === 1;
    }

    /**
     * Validate IP address
     */
    public static function validateIp(string $ip): bool
    {
        return filter_var($ip, FILTER_VALIDATE_IP) !== false;
    }

    /**
     * Validate UUID
     */
    public static function validateUuid(string $uuid): bool
    {
        return preg_match('/^[0-9a-f]{8}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{12}$/i', $uuid) === 1;
    }

    /**
     * Sanitize HTML content
     */
    public static function sanitizeHtml(string $html): string
    {
        $config = HTMLPurifier_Config::createDefault();
        $purifier = new HTMLPurifier($config);

        return $purifier->purify($html);
    }

    /**
     * Validate CSRF token
     */
    public static function validateCsrf(string $token): bool
    {
        return isset($_SESSION['csrf_token']) && hash_equals($_SESSION['csrf_token'], $token);
    }

    /**
     * Generate CSRF token
     */
    public static function generateCsrf(): string
    {
        if (empty($_SESSION['csrf_token'])) {
            $_SESSION['csrf_token'] = bin2hex(random_bytes(32));
        }

        return $_SESSION['csrf_token'];
    }

    /**
     * Validate array keys
     */
    public static function validateArrayKeys(array $data, array $required): bool
    {
        foreach ($required as $key) {
            if (!isset($data[$key]) || $data[$key] === '') {
                return false;
            }
        }

        return true;
    }
}
```

## Two-Factor Authentication

```php
<?php
// /includes/security/two_factor.php

class TwoFactorAuth
{
    /**
     * Generate secret
     */
    public static function generateSecret(int $length = 32): string
    {
        $chars = 'ABCDEFGHIJKLMNOPQRSTUVWXYZ234567';
        $secret = '';

        for ($i = 0; $i < $length; $i++) {
            $secret .= $chars[random_int(0, strlen($chars) - 1)];
        }

        return $secret;
    }

    /**
     * Get TOTP URI for QR code
     */
    public static function getTotpUri(string $secret, string $accountName, string $issuer = 'WHMCS'): string
    {
        $encodedIssuer = urlencode($issuer);
        $encodedAccount = urlencode($accountName);

        return "otpauth://totp/{$encodedIssuer}:{$encodedAccount}?secret={$secret}&issuer={$encodedIssuer}&algorithm=SHA1&digits=6&period=30";
    }

    /**
     * Verify TOTP code
     */
    public static function verifyTotp(string $secret, string $code, int $window = 1): bool
    {
        $timeSlice = floor(time() / 30);

        for ($i = -$window; $i <= $window; $i++) {
            $hash = self::totpHash($secret, $timeSlice + $i);
            if (hash_equals($hash, str_pad($code, 6, '0', STR_PAD_LEFT))) {
                return true;
            }
        }

        return false;
    }

    /**
     * Calculate TOTP hash
     */
    private static function totpHash(string $secret, int $timeSlice): string
    {
        $decoded = self::base32Decode($secret);
        $time = pack('N', $timeSlice);
        $hash = hash_hmac('sha1', $time, $decoded, true);

        return str_pad(
            (string) (
                (ord($hash[19]) & 0xf) << 24 |
                (ord($hash[18]) & 0xff) << 16 |
                (ord($hash[17]) & 0xff) << 8 |
                (ord($hash[16]) & 0xff)
            ),
            6,
            '0',
            STR_PAD_LEFT
        );
    }

    /**
     * Base32 decode
     */
    private static function base32Decode(string $encoded): string
    {
        $alphabet = 'ABCDEFGHIJKLMNOPQRSTUVWXYZ234567';
        $encoded = strtoupper($encoded);

        $decoded = '';
        $bits = 0;
        $value = 0;

        for ($i = 0; $i < strlen($encoded); $i++) {
            $index = strpos($alphabet, $encoded[$i]);
            if ($index === false) {
                continue;
            }

            $value = ($value << 5) | $index;
            $bits += 5;

            if ($bits >= 8) {
                $decoded .= chr(($value >> ($bits - 8)) & 0xff);
                $bits -= 8;
            }
        }

        return $decoded;
    }
}
```

## Best Practices

1. **Input Validation**: Always validate and sanitize user input
2. **Password Hashing**: Use bcrypt or Argon2 for password storage
3. **CSRF Protection**: Generate and validate CSRF tokens on all forms
4. **SQL Injection**: Use prepared statements exclusively
5. **XSS Prevention**: Escape output using htmlspecialchars
6. **Rate Limiting**: Implement rate limiting on sensitive endpoints
7. **Session Security**: Use secure session cookies
8. **Two-Factor Auth**: Enable 2FA for admin and sensitive operations
9. **IP Blocking**: Block suspicious IPs automatically
10. **Audit Logging**: Log all security-relevant events
