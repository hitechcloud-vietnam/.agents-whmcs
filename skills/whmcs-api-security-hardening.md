# WHMCS API Security Hardening

## Skill Description
Implement comprehensive security hardening for WHMCS API modules including input validation, output encoding, secure headers, and protection against common vulnerabilities.

## Prerequisites
- WHMCS 7.0+ installation
- PHP 7.4+
- Understanding of web security vulnerabilities
- SSL/HTTPS configuration

## Step-by-Step Implementation

### 1. Security Headers
```php
<?php
// includes/security/SecurityHeaders.php

namespace WHMCS\Module\YourModule\Security;

class SecurityHeaders
{
    public static function apply(): void
    {
        // Prevent MIME type sniffing
        header('X-Content-Type-Options: nosniff');

        // Prevent clickjacking
        header('X-Frame-Options: SAMEORIGIN');

        // XSS protection
        header('X-XSS-Protection: 1; mode=block');

        // Referrer policy
        header('Referrer-Policy: strict-origin-when-cross-origin');

        // Content Security Policy
        header('Content-Security-Policy: default-src \'self\'; script-src \'self\'; style-src \'self\'; img-src \'self\' data:; font-src \'self\'; connect-src \'self\';');

        // Permissions Policy
        header('Permissions-Policy: geolocation=(), microphone=(), camera=()');

        // Strict Transport Security (requires HTTPS)
        if (isset($_SERVER['HTTPS']) && $_SERVER['HTTPS'] === 'on') {
            header('Strict-Transport-Security: max-age=31536000; includeSubDomains');
        }

        // Cache control for sensitive data
        header('Cache-Control: no-store, no-cache, must-revalidate, private');
        header('Pragma: no-cache');
    }

    public static function applyCORS(array $allowedOrigins = []): void
    {
        if (empty($allowedOrigins)) {
            return;
        }

        $origin = $_SERVER['HTTP_ORIGIN'] ?? '';

        if (in_array($origin, $allowedOrigins)) {
            header("Access-Control-Allow-Origin: {$origin}");
            header('Access-Control-Allow-Methods: GET, POST, PUT, DELETE, OPTIONS');
            header('Access-Control-Allow-Headers: Content-Type, Authorization, X-API-Key');
            header('Access-Control-Max-Age: 86400');
        }

        // Handle preflight
        if ($_SERVER['REQUEST_METHOD'] === 'OPTIONS') {
            http_response_code(204);
            exit;
        }
    }
}
```

### 2. Input Sanitizer
```php
<?php
// includes/security/InputSanitizer.php

namespace WHMCS\Module\YourModule\Security;

class InputSanitizer
{
    public static function sanitizeString(string $input, int $maxLength = 1000): string
    {
        $input = trim($input);
        $input = stripslashes($input);
        $input = strip_tags($input);
        $input = htmlspecialchars($input, ENT_QUOTES | ENT_HTML5, 'UTF-8');

        if ($maxLength > 0) {
            $input = mb_substr($input, 0, $maxLength, 'UTF-8');
        }

        return $input;
    }

    public static function sanitizeEmail(string $email): string
    {
        $email = trim($email);
        $email = filter_var($email, FILTER_SANITIZE_EMAIL);

        if (!filter_var($email, FILTER_VALIDATE_EMAIL)) {
            throw new \InvalidArgumentException('Invalid email address');
        }

        return $email;
    }

    public static function sanitizeInteger(mixed $value): int
    {
        return (int) filter_var($value, FILTER_SANITIZE_NUMBER_INT);
    }

    public static function sanitizeFloat(mixed $value): float
    {
        return (float) filter_var($value, FILTER_SANITIZE_NUMBER_FLOAT, FILTER_FLAG_ALLOW_FRACTION);
    }

    public static function sanitizeArray(array $input, array $allowedKeys): array
    {
        return array_intersect_key($input, array_flip($allowedKeys));
    }

    public static function sanitizeHtml(string $input, array $allowedTags = []): string
    {
        if (empty($allowedTags)) {
            return strip_tags($input);
        }

        $allowedTags = '<' . implode('><', $allowedTags) . '>';
        return strip_tags($input, $allowedTags);
    }

    public static function sanitizeFilename(string $filename): string
    {
        // Remove any directory components
        $filename = basename($filename);
        // Remove special characters
        $filename = preg_replace('/[^a-zA-Z0-9._-]/', '_', $filename);
        // Limit length
        $filename = mb_substr($filename, 0, 255, 'UTF-8');

        return $filename;
    }

    public static function sanitizeSqlWildcards(string $input): string
    {
        return str_replace(['%', '_'], ['\%', '\_'], $input);
    }
}
```

### 3. CSRF Protection
```php
<?php
// includes/security/CsrfProtection.php

namespace WHMCS\Module\YourModule\Security;

class CsrfProtection
{
    private const TOKEN_NAME = 'csrf_token';
    private const TOKEN_LENGTH = 32;

    public static function generate(): string
    {
        if (!isset($_SESSION)) {
            session_start();
        }

        $token = bin2hex(random_bytes(self::TOKEN_LENGTH));
        $_SESSION[self::TOKEN_NAME] = [
            'token' => hash('sha256', $token),
            'expires' => time() + 3600 // 1 hour
        ];

        return $token;
    }

    public static function validate(string $token): bool
    {
        if (!isset($_SESSION)) {
            session_start();
        }

        if (!isset($_SESSION[self::TOKEN_NAME])) {
            return false;
        }

        $stored = $_SESSION[self::TOKEN_NAME];

        // Check expiration
        if ($stored['expires'] < time()) {
            unset($_SESSION[self::TOKEN_NAME]);
            return false;
        }

        // Constant-time comparison
        if (!hash_equals($stored['token'], hash('sha256', $token))) {
            return false;
        }

        // Regenerate token (single use)
        unset($_SESSION[self::TOKEN_NAME]);

        return true;
    }

    public static function getField(): string
    {
        $token = self::generate();
        return '<input type="hidden" name="' . self::TOKEN_NAME . '" value="' . htmlspecialchars($token) . '">';
    }

    public static function getTokenFromRequest(): ?string
    {
        return $_POST[self::TOKEN_NAME]
            ?? $_SERVER['HTTP_X_CSRF_TOKEN']
            ?? $_GET[self::TOKEN_NAME]
            ?? null;
    }

    public static function requireValidation(): void
    {
        $token = self::getTokenFromRequest();

        if (!$token || !self::validate($token)) {
            http_response_code(403);
            header('Content-Type: application/json');
            echo json_encode([
                'success' => false,
                'error' => 'CSRF validation failed'
            ]);
            exit;
        }
    }
}
```

### 4. SQL Injection Prevention
```php
<?php
// includes/security/DatabaseSecurity.php

namespace WHMCS\Module\YourModule\Security;

class DatabaseSecurity
{
    public static function prepareQuery(string $query, array $params = []): array
    {
        // Validate query is SELECT, INSERT, UPDATE, or DELETE
        $validTypes = ['SELECT', 'INSERT', 'UPDATE', 'DELETE', 'REPLACE'];

        $firstWord = strtoupper(strtok($query, " \t\n\r\0\x0B"));

        if (!in_array($firstWord, $validTypes)) {
            throw new \InvalidArgumentException('Invalid SQL query type');
        }

        // Check for dangerous patterns
        $dangerousPatterns = [
            '/\bDROP\s+TABLE\b/i',
            '/\bDROP\s+DATABASE\b/i',
            '/\bTRUNCATE\b/i',
            '/\bLOAD_FILE\b/i',
            '/\bINTO\s+OUTFILE\b/i',
            '/\bINTO\s+DUMPFILE\b/i',
            '/\;.*\bSELECT\b/i'
        ];

        foreach ($dangerousPatterns as $pattern) {
            if (preg_match($pattern, $query)) {
                logActivity('Potential SQL injection attempt detected');
                throw new \InvalidArgumentException('Query contains forbidden patterns');
            }
        }

        return [$query, $params];
    }

    public static function escapeValue(mixed $value): string
    {
        if ($value === null) {
            return 'NULL';
        }

        if (is_int($value)) {
            return (int) $value;
        }

        if (is_bool($value)) {
            return $value ? '1' : '0';
        }

        if (is_float($value)) {
            return (float) $value;
        }

        global $db;

        return "'" . $db->escape($value) . "'";
    }

    public static function buildWhereClause(array $conditions): string
    {
        $parts = [];

        foreach ($conditions as $column => $value) {
            if (is_array($value)) {
                // IN clause
                $values = array_map([self::class, 'escapeValue'], $value);
                $parts[] = $column . ' IN (' . implode(', ', $values) . ')';
            } elseif ($value === null) {
                $parts[] = $column . ' IS NULL';
            } else {
                $parts[] = $column . ' = ' . self::escapeValue($value);
            }
        }

        return implode(' AND ', $parts);
    }
}
```

### 5. Rate Limiting Security
```php
<?php
// includes/security/RateLimitSecurity.php

namespace WHMCS\Module\YourModule\Security;

class RateLimitSecurity
{
    private static array $limits = [
        'login' => ['attempts' => 5, 'window' => 300], // 5 attempts per 5 minutes
        'api' => ['attempts' => 100, 'window' => 60], // 100 per minute
        'password_reset' => ['attempts' => 3, 'window' => 3600] // 3 per hour
    ];

    public static function checkLimit(string $action, string $identifier): bool
    {
        $limit = self::$limits[$action] ?? self::$limits['api'];
        $key = "rate_limit:{$action}:{$identifier}";

        $redis = self::getRedis();

        if ($redis) {
            return self::checkRedisLimit($redis, $key, $limit);
        }

        return self::checkFileLimit($key, $limit);
    }

    private static function getRedis(): ?\Redis
    {
        try {
            $redis = new \Redis();
            $redis->connect('127.0.0.1', 6379);
            return $redis;
        } catch (\Exception $e) {
            return null;
        }
    }

    private static function checkRedisLimit(\Redis $redis, string $key, array $limit): bool
    {
        $count = (int) $redis->incr($key);

        if ($count === 1) {
            $redis->expire($key, $limit['window']);
        }

        return $count <= $limit['attempts'];
    }

    private static function checkFileLimit(string $key, array $limit): bool
    {
        $file = sys_get_temp_dir() . '/' . md5($key) . '.json';
        $now = time();

        $data = file_exists($file) ? json_decode(file_get_contents($file), true) : null;

        if (!$data || $data['reset_at'] < $now) {
            $data = [
                'attempts' => 0,
                'reset_at' => $now + $limit['window']
            ];
        }

        $data['attempts']++;

        file_put_contents($file, json_encode($data));

        return $data['attempts'] <= $limit['attempts'];
    }

    public static function recordFailure(string $action, string $identifier): void
    {
        $key = "rate_limit_failure:{$action}:{$identifier}";

        $redis = self::getRedis();

        if ($redis) {
            $redis->incr($key);
            $redis->expire($key, 86400); // Track for 24 hours
        }

        logActivity("Rate limit exceeded: {$action} for {$identifier}");
    }

    public static function isBlocked(string $identifier): bool
    {
        $key = "blocked:{$identifier}";

        $redis = self::getRedis();

        if ($redis) {
            return $redis->exists($key) > 0;
        }

        $file = sys_get_temp_dir() . '/blocked_' . md5($identifier);
        return file_exists($file);
    }

    public static function block(string $identifier, int $duration = 3600): void
    {
        $key = "blocked:{$identifier}";

        $redis = self::getRedis();

        if ($redis) {
            $redis->setex($key, $duration, '1');
        } else {
            $file = sys_get_temp_dir() . '/blocked_' . md5($identifier);
            file_put_contents($file, time() + $duration);
        }

        logActivity("IP blocked: {$identifier} for {$duration} seconds");
    }
}
```

### 6. Security Middleware
```php
<?php
// includes/security/SecurityMiddleware.php

namespace WHMCS\Module\YourModule\Security;

class SecurityMiddleware
{
    public static function apply(): void
    {
        // Apply security headers
        SecurityHeaders::apply();

        // Check for blocked IPs
        $ip = $_SERVER['REMOTE_ADDR'] ?? '';
        if (RateLimitSecurity::isBlocked($ip)) {
            http_response_code(403);
            header('Content-Type: application/json');
            echo json_encode(['error' => 'Access denied']);
            exit;
        }
    }

    public static function validateRequest(): void
    {
        // Block suspicious requests
        self::checkSuspiciousPatterns();

        // Validate content type for POST/PUT/PATCH
        if (in_array($_SERVER['REQUEST_METHOD'], ['POST', 'PUT', 'PATCH'])) {
            self::validateContentType();
        }
    }

    private static function checkSuspiciousPatterns(): void
    {
        $uri = $_SERVER['REQUEST_URI'] ?? '';
        $query = $_SERVER['QUERY_STRING'] ?? '';

        $patterns = [
            '/\.\.\//', // Path traversal
            '/<script/i', // XSS
            '/union\s+select/i', // SQL injection
            '/eval\s*\(/i', // Code injection
            '/base64_decode\s*\(/i' // Obfuscation
        ];

        foreach ($patterns as $pattern) {
            if (preg_match($pattern, $uri) || preg_match($pattern, $query)) {
                logActivity('Suspicious request pattern detected: ' . $uri);
                http_response_code(400);
                header('Content-Type: application/json');
                echo json_encode(['error' => 'Bad request']);
                exit;
            }
        }
    }

    private static function validateContentType(): void
    {
        $contentType = $_SERVER['CONTENT_TYPE'] ?? '';

        if (stripos($contentType, 'application/json') !== false) {
            // Validate JSON
            $input = file_get_contents('php://input');
            json_decode($input);

            if (json_last_error() !== JSON_ERROR_NONE) {
                http_response_code(400);
                header('Content-Type: application/json');
                echo json_encode(['error' => 'Invalid JSON']);
                exit;
            }
        }
    }
}
```

## Common Pitfalls and Solutions

| Pitfall | Solution |
|---------|----------|
| Missing security headers | Apply headers to all responses, not just API |
| Unvalidated user input | Sanitize all input at entry points |
| SQL injection via ORM bypass | Use prepared statements exclusively |
| CSRF on API endpoints | Implement token validation for state-changing operations |
| Weak rate limits | Use Redis for distributed rate limiting |

## Security Considerations

1. **Always use HTTPS** - Never transmit sensitive data over HTTP
2. **Validate all input** - Assume all user input is malicious
3. **Use prepared statements** - Never concatenate user input into SQL
4. **Implement CSRF protection** - Protect all state-changing endpoints
5. **Log security events** - Track authentication failures and attacks
6. **Use secure password hashing** - Use Argon2id or bcrypt
7. **Implement proper authentication** - Don't rely on IP alone

## Testing Checklist

- [ ] Test all security headers are present
- [ ] Test XSS protection with malicious input
- [ ] Test SQL injection prevention
- [ ] Test CSRF validation
- [ ] Test rate limiting enforcement
- [ ] Test path traversal prevention
- [ ] Test blocked IP handling
- [ ] Test content type validation
- [ ] Test CORS configuration
- [ ] Test security audit logging

## Reference Links

- [OWASP Top 10](https://owasp.org/www-project-top-ten/)
- [OWASP API Security Top 10](https://owasp.org/www-project-api-security/)
- [PHP Security Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/PHP_Security_Cheat_Sheet.html)
- [WHMCS Security Guidelines](https://developers.whmcs.com/security/)
