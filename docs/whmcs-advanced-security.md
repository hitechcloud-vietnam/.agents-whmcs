# WHMCS Advanced Security

Complete guide to security hardening.

## Overview

Implement comprehensive security measures for WHMCS.

## Input Validation

### Input Sanitizer

```php
<?php
/**
 * Input sanitization
 */
class InputSanitizer
{
    /**
     * Sanitize string input
     */
    public static function sanitizeString(string $input, int $maxLength = 255): string
    {
        $input = trim($input);
        $input = strip_tags($input);
        $input = htmlspecialchars($input, ENT_QUOTES | ENT_HTML5, 'UTF-8');
        
        return mb_substr($input, 0, $maxLength);
    }
    
    /**
     * Sanitize email
     */
    public static function sanitizeEmail(string $email): string
    {
        $email = trim(strtolower($email));
        $email = filter_var($email, FILTER_SANITIZE_EMAIL);
        
        return filter_var($email, FILTER_VALIDATE_EMAIL) ? $email : '';
    }
    
    /**
     * Sanitize integer
     */
    public static function sanitizeInteger($input): int
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
        if (!preg_match('/^https?:\/\//', $url)) {
            return '';
        }
        
        return $url;
    }
    
    /**
     * Sanitize HTML (allow limited tags)
     */
    public static function sanitizeHtml(string $html, array $allowedTags = ['p', 'br', 'b', 'i', 'u']): string
    {
        $html = strip_tags($html, '<' . implode('><', $allowedTags) . '>');
        return $html;
    }
    
    /**
     * Validate domain name
     */
    public static function validateDomain(string $domain): bool
    {
        return filter_var($domain, FILTER_VALIDATE_DOMAIN) !== false;
    }
    
    /**
     * Validate IP address
     */
    public static function validateIp(string $ip): bool
    {
        return filter_var($ip, FILTER_VALIDATE_IP) !== false;
    }
}
```

## Password Security

### Password Manager

```php
<?php
/**
 * Password security utilities
 */
class PasswordManager
{
    /**
     * Hash password
     */
    public static function hash(string $password): string
    {
        return password_hash($password, PASSWORD_ARGON2ID, [
            'memory_cost' => 65536,
            'time_cost' => 4,
            'threads' => 3,
        ]);
    }
    
    /**
     * Verify password
     */
    public static function verify(string $password, string $hash): bool
    {
        return password_verify($password, $hash);
    }
    
    /**
     * Check password strength
     */
    public static function checkStrength(string $password): array
    {
        $score = 0;
        $feedback = [];
        
        // Length checks
        if (strlen($password) >= 8) {
            $score += 1;
        }
        if (strlen($password) >= 12) {
            $score += 1;
        }
        if (strlen($password) >= 16) {
            $score += 1;
        }
        
        // Character variety
        if (preg_match('/[a-z]/', $password)) {
            $score += 1;
        }
        if (preg_match('/[A-Z]/', $password)) {
            $score += 1;
        }
        if (preg_match('/[0-9]/', $password)) {
            $score += 1;
        }
        if (preg_match('/[^a-zA-Z0-9]/', $password)) {
            $score += 1;
        }
        
        // Feedback
        if (strlen($password) < 8) {
            $feedback[] = 'Password must be at least 8 characters';
        }
        if (!preg_match('/[A-Z]/', $password)) {
            $feedback[] = 'Add uppercase letters';
        }
        if (!preg_match('/[0-9]/', $password)) {
            $feedback[] = 'Add numbers';
        }
        if (!preg_match('/[^a-zA-Z0-9]/', $password)) {
            $feedback[] = 'Add special characters';
        }
        
        $strength = match(true) {
            $score >= 6 => 'strong',
            $score >= 4 => 'medium',
            default => 'weak',
        };
        
        return [
            'score' => $score,
            'strength' => $strength,
            'feedback' => $feedback,
        ];
    }
    
    /**
     * Generate secure random password
     */
    public static function generate(int $length = 16): string
    {
        $chars = 'abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ0123456789!@#$%^&*';
        $password = '';
        
        for ($i = 0; $i < $length; $i++) {
            $password .= $chars[random_int(0, strlen($chars) - 1)];
        }
        
        return $password;
    }
}
```

## Encryption

### Data Encryption

```php
<?php
/**
 * Encryption utilities
 */
class EncryptionManager
{
    private string $key;
    
    public function __construct(string $key = null)
    {
        $this->key = $key ?? App::getApplication()->getConfig()->encryption_key;
    }
    
    /**
     * Encrypt data
     */
    public function encrypt(string $data): string
    {
        $iv = random_bytes(16);
        $cipher = 'aes-256-gcm';
        
        $tag = '';
        $encrypted = openssl_encrypt(
            $data,
            $cipher,
            $this->key,
            OPENSSL_RAW_DATA,
            $iv,
            $tag,
            '',
            16
        );
        
        // Combine IV + ciphertext + tag
        return base64_encode($iv . $encrypted . $tag);
    }
    
    /**
     * Decrypt data
     */
    public function decrypt(string $encryptedData): string
    {
        $data = base64_decode($encryptedData);
        
        $iv = substr($data, 0, 16);
        $tag = substr($data, -16);
        $ciphertext = substr($data, 16, -16);
        
        return openssl_decrypt(
            $ciphertext,
            'aes-256-gcm',
            $this->key,
            OPENSSL_RAW_DATA,
            $iv,
            $tag
        );
    }
    
    /**
     * Encrypt sensitive module data
     */
    public function encryptApiKey(string $apiKey): string
    {
        return $this->encrypt($apiKey);
    }
    
    /**
     * Decrypt sensitive module data
     */
    public function decryptApiKey(string $encryptedKey): string
    {
        return $this->decrypt($encryptedKey);
    }
}
```

## Rate Limiting

### Rate Limiter

```php
<?php
/**
 * Rate limiting
 */
class RateLimiter
{
    private RedisCache $cache;
    private int $maxAttempts;
    private int $decayMinutes;
    
    public function __construct(RedisCache $cache, int $maxAttempts = 60, int $decayMinutes = 1)
    {
        $this->cache = $cache;
        $this->maxAttempts = $maxAttempts;
        $this->decayMinutes = $decayMinutes;
    }
    
    /**
     * Check if request is allowed
     */
    public function attempt(string $key): bool
    {
        $cacheKey = "rate_limit:{$key}";
        $attempts = $this->cache->get($cacheKey) ?? 0;
        
        if ($attempts >= $this->maxAttempts) {
            return false;
        }
        
        $this->cache->increment($cacheKey);
        
        if ($attempts === 0) {
            $this->cache->set($cacheKey, 1, $this->decayMinutes * 60);
        }
        
        return true;
    }
    
    /**
     * Get remaining attempts
     */
    public function remaining(string $key): int
    {
        $attempts = $this->cache->get("rate_limit:{$key}") ?? 0;
        return max(0, $this->maxAttempts - $attempts);
    }
    
    /**
     * Reset rate limit
     */
    public function reset(string $key): void
    {
        $this->cache->delete("rate_limit:{$key}");
    }
    
    /**
     * Get retry after seconds
     */
    public function retryAfter(string $key): int
    {
        // Implement based on cache TTL
        return $this->decayMinutes * 60;
    }
}
```

## Security Headers

```php
<?php
/**
 * Security headers middleware
 */
function setSecurityHeaders(): void
{
    // Prevent XSS
    header('X-XSS-Protection: 1; mode=block');
    
    // Prevent clickjacking
    header('X-Frame-Options: DENY');
    
    // Content type sniffing
    header('X-Content-Type-Options: nosniff');
    
    // HTTPS only
    header('Strict-Transport-Security: max-age=31536000; includeSubDomains');
    
    // CSP
    header('Content-Security-Policy: default-src \'self\'; script-src \'self\'; style-src \'self\' \'unsafe-inline\'');
    
    // Referrer policy
    header('Referrer-Policy: strict-origin-when-cross-origin');
    
    // Permissions policy
    header('Permissions-Policy: geolocation=(), microphone=(), camera=()');
}
```

## Audit Trail

### Security Audit

```php
<?php
/**
 * Security audit logging
 */
class SecurityAudit
{
    /**
     * Log authentication attempt
     */
    public static function logAuthAttempt(string $email, bool $success, string $ip): void
    {
        Capsule::table('mod_security_log')->insert([
            'event_type' => 'auth_attempt',
            'email' => $success ? $email : null,
            'success' => $success ? 1 : 0,
            'ip_address' => $ip,
            'user_agent' => $_SERVER['HTTP_USER_AGENT'] ?? '',
            'created_at' => date('Y-m-d H:i:s'),
        ]);
        
        // Alert on failed attempts
        if (!$success) {
            self::checkForBruteForce($ip);
        }
    }
    
    /**
     * Check for brute force attacks
     */
    private static function checkForBruteForce(string $ip): void
    {
        $recentFailures = Capsule::table('mod_security_log')
            ->where('event_type', 'auth_attempt')
            ->where('success', 0)
            ->where('ip_address', $ip)
            ->where('created_at', '>', date('Y-m-d H:i:s', strtotime('-15 minutes')))
            ->count();
        
        if ($recentFailures >= 10) {
            self::alertBruteForce($ip, $recentFailures);
        }
    }
    
    /**
     * Alert admin of brute force
     */
    private static function alertBruteForce(string $ip, int $attempts): void
    {
        logActivity("Potential brute force attack from {$ip}: {$attempts} failed attempts in 15 minutes");
        
        // Send alert email
        $adminEmail = Capsule::table('tblconfiguration')
            ->where('setting', 'SupportEmail')
            ->first()->value ?? '';
        
        if ($adminEmail) {
            mail($adminEmail,
                'Security Alert: Brute Force Attempt',
                "IP: {$ip}\nFailed attempts: {$attempts}"
            );
        }
    }
    
    /**
     * Log data access
     */
    public static function logDataAccess(string $resource, int $resourceId, string $action): void
    {
        Capsule::table('mod_security_log')->insert([
            'event_type' => 'data_access',
            'resource' => $resource,
            'resource_id' => $resourceId,
            'action' => $action,
            'user_id' => $_SESSION['uid'] ?? 0,
            'ip_address' => $_SERVER['REMOTE_ADDR'] ?? '',
            'created_at' => date('Y-m-d H:i:s'),
        ]);
    }
}
```

## Best Practices

1. **Validate all input** - Never trust user data
2. **Hash passwords** - Use Argon2id
3. **Encrypt sensitive data** - Use AES-256-GCM
4. **Implement rate limiting** - Prevent abuse
5. **Log security events** - Track suspicious activity
6. **Use security headers** - Layered defense

## Related Documentation

- [whmcs-advanced-security.md](whmcs-advanced-security.md)
- [whmcs-integration-logging.md](whmcs-integration-logging.md)
