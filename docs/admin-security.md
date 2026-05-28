# Admin Area Security Guide

Securing the WHMCS admin area is paramount for protecting sensitive client data and system integrity. This guide covers comprehensive security measures for admin-facing functionality.

## Authentication Security

### Enhanced Admin Authentication

```php
<?php
/**
 * Enhanced admin authentication module
 */
class AdminAuthenticationManager
{
    private $sessionTimeout = 3600; // 1 hour
    private $maxLoginAttempts = 5;
    private $lockoutDuration = 900; // 15 minutes

    /**
     * Authenticate admin with enhanced security
     */
    public function authenticate(string $username, string $password): AuthResult
    {
        // Check for existing lockout
        if ($this->isLockedOut($username)) {
            return new AuthResult(false, 'Account is temporarily locked');
        }

        // Verify credentials
        $admin = Capsule::table('tbladminuser')
            ->where('username', $username)
            ->first();

        if (!$admin || !password_verify($password, $admin->password)) {
            $this->recordFailedAttempt($username);
            return new AuthResult(false, 'Invalid credentials');
        }

        // Check if password needs rehashing
        if (password_needs_rehash($admin->password, PASSWORD_DEFAULT)) {
            $this->updatePasswordHash($admin->id, $password);
        }

        // Clear failed attempts on success
        $this->clearFailedAttempts($username);

        // Create session
        return $this->createSession($admin);
    }

    /**
     * Create secure admin session
     */
    private function createSession(stdClass $admin): AuthResult
    {
        $sessionId = $this->generateSessionId();

        Capsule::table('tbladmin_session')->insert([
            'admin_id' => $admin->id,
            'session_id' => $sessionId,
            'ip_address' => $this->getClientIp(),
            'user_agent' => $_SERVER['HTTP_USER_AGENT'] ?? '',
            'created_at' => date('Y-m-d H:i:s'),
            'expires_at' => date('Y-m-d H:i:s', time() + $this->sessionTimeout),
        ]);

        // Set secure session cookies
        setcookie('admin_session', $sessionId, [
            'expires' => time() + $this->sessionTimeout,
            'path' => '/admin',
            'httponly' => true,
            'secure' => $this->isHttps(),
            'samesite' => 'Strict',
        ]);

        return new AuthResult(true, 'Authenticated', [
            'admin_id' => $admin->id,
            'permissions' => $this->getAdminPermissions($admin->id),
        ]);
    }

    private function generateSessionId(): string
    {
        return bin2hex(random_bytes(32));
    }

    private function isLockedOut(string $username): bool
    {
        $lockout = Capsule::table('mod_security_lockouts')
            ->where('username', $username)
            ->where('locked_until', '>', date('Y-m-d H:i:s'))
            ->first();

        return $lockout !== null;
    }

    private function recordFailedAttempt(string $username): void
    {
        $attempts = Capsule::table('mod_security_login_attempts')
            ->where('username', $username)
            ->first();

        if ($attempts) {
            Capsule::table('mod_security_login_attempts')
                ->where('username', $username)
                ->increment('attempts');
        } else {
            Capsule::table('mod_security_login_attempts')->insert([
                'username' => $username,
                'attempts' => 1,
                'ip_address' => $this->getClientIp(),
                'first_attempt' => date('Y-m-d H:i:s'),
            ]);
        }

        // Check if should lock out
        $totalAttempts = ($attempts->attempts ?? 0) + 1;
        if ($totalAttempts >= $this->maxLoginAttempts) {
            Capsule::table('mod_security_lockouts')->insert([
                'username' => $username,
                'ip_address' => $this->getClientIp(),
                'locked_until' => date('Y-m-d H:i:s', time() + $this->lockoutDuration),
                'reason' => 'Exceeded maximum login attempts',
            ]);
        }
    }

    private function clearFailedAttempts(string $username): void
    {
        Capsule::table('mod_security_login_attempts')
            ->where('username', $username)
            ->delete();
    }
}
```

### Multi-Factor Authentication

```php
<?php
/**
 * Two-Factor Authentication for admin
 */
class TwoFactorAuthenticator
{
    private $issuer = 'WHMCS';
    private $digits = 6;
    private $period = 30;

    /**
     * Generate secret for new enrollment
     */
    public function generateSecret(): string
    {
        return $this->base32Encode(random_bytes(20));
    }

    /**
     * Get provisioning URI for QR code
     */
    public function getProvisioningUri(string $secret, string $accountName): string
    {
        $label = rawurlencode($accountName);
        $params = http_build_query([
            'secret' => $secret,
            'issuer' => $this->issuer,
            'digits' => $this->digits,
            'period' => $this->period,
        ]);

        return "otpauth://totp/{$label}?{$params}";
    }

    /**
     * Verify TOTP code
     */
    public function verifyCode(string $secret, string $code, int $window = 1): bool
    {
        $timeSlice = floor(time() / $this->period);

        for ($i = -$window; $i <= $window; $i++) {
            $hash = $this->generateHash($secret, $timeSlice + $i);
            $offset = ord($hash[-1]) & 0x0F;
            $binary = (
                (ord($hash[$offset]) & 0x7F) << 24 |
                (ord($hash[$offset + 1]) & 0xFF) << 16 |
                (ord($hash[$offset + 2]) & 0xFF) << 8 |
                (ord($hash[$offset + 3]) & 0xFF)
            );
            $otp = $binary % pow(10, $this->digits);

            if (hash_equals(str_pad((string)$otp, $this->digits, '0', STR_PAD_LEFT), $code)) {
                return true;
            }
        }

        return false;
    }

    private function generateHash(string $secret, int $timeSlice): string
    {
        $secretKey = $this->base32Decode($secret);
        $timeSlice = pack('N', $timeSlice);
        $timeSlice = str_pad($timeSlice, 8, "\x00", STR_PAD_LEFT);

        $hash = hash_hmac('sha1', $timeSlice, $secretKey, true);

        return $hash;
    }

    private function base32Encode(string $data): string
    {
        $alphabet = 'ABCDEFGHIJKLMNOPQRSTUVWXYZ234567';
        $binary = '';

        foreach (str_split($data) as $char) {
            $binary .= str_pad(decbin(ord($char)), 8, '0', STR_PAD_LEFT);
        }

        $encoded = '';
        for ($i = 0; $i < strlen($binary); $i += 5) {
            $chunk = substr($binary, $i, 5);
            if (strlen($chunk) < 5) {
                $chunk = str_pad($chunk, 5, '0', STR_PAD_RIGHT);
            }
            $encoded .= $alphabet[bindec($chunk)];
        }

        return $encoded;
    }

    private function base32Decode(string $data): string
    {
        $alphabet = 'ABCDEFGHIJKLMNOPQRSTUVWXYZ234567';
        $binary = '';

        foreach (str_split(strtoupper($data)) as $char) {
            if ($char === '=') continue;
            $val = strpos($alphabet, $char);
            if ($val === false) continue;
            $binary .= str_pad(decbin($val), 5, '0', STR_PAD_LEFT);
        }

        $decoded = '';
        for ($i = 0; $i < strlen($binary); $i += 8) {
            $byte = substr($binary, $i, 8);
            if (strlen($byte) === 8) {
                $decoded .= chr(bindec($byte));
            }
        }

        return $decoded;
    }
}
```

## Role-Based Access Control

### Permission Manager

```php
<?php
/**
 * Admin permission manager
 */
class AdminPermissionManager
{
    private const PERMISSIONS = [
        'view_orders' => 'View orders and invoices',
        'create_orders' => 'Create new orders',
        'edit_orders' => 'Modify existing orders',
        'delete_orders' => 'Delete orders',
        'view_clients' => 'View client information',
        'edit_clients' => 'Modify client data',
        'delete_clients' => 'Delete clients',
        'manage_modules' => 'Configure modules',
        'view_reports' => 'Access reporting',
        'manage_settings' => 'Modify system settings',
        'view_logs' => 'Access system logs',
        'manage_admins' => 'Manage admin users',
    ];

    private array $rolePermissions = [
        'super_admin' => ['*'], // All permissions
        'support' => [
            'view_orders', 'view_clients', 'edit_clients', 'view_reports',
        ],
        'billing' => [
            'view_orders', 'create_orders', 'edit_orders', 'view_clients',
            'view_reports', 'view_logs',
        ],
        'developer' => [
            'manage_modules', 'view_logs', 'manage_settings',
        ],
    ];

    /**
     * Check if admin has specific permission
     */
    public function hasPermission(int $adminId, string $permission): bool
    {
        $admin = $this->getAdminRole($adminId);

        if (!$admin) {
            return false;
        }

        $permissions = $this->getRolePermissions($admin->role);

        // Super admin has all permissions
        if (in_array('*', $permissions)) {
            return true;
        }

        return in_array($permission, $permissions);
    }

    /**
     * Require permission or throw exception
     */
    public function requirePermission(int $adminId, string $permission): void
    {
        if (!$this->hasPermission($adminId, $permission)) {
            throw new PermissionDeniedException(
                "Admin does not have required permission: $permission"
            );
        }
    }

    /**
     * Get all permissions for an admin
     */
    public function getAdminPermissions(int $adminId): array
    {
        $admin = $this->getAdminRole($adminId);

        if (!$admin) {
            return [];
        }

        return $this->getRolePermissions($admin->role);
    }

    private function getAdminRole(int $adminId): ?stdClass
    {
        return Capsule::table('tbladminuser')
            ->where('id', $adminId)
            ->first();
    }

    private function getRolePermissions(string $role): array
    {
        return $this->rolePermissions[$role] ?? [];
    }
}

/**
 * Custom exception for permission denials
 */
class PermissionDeniedException extends Exception {}
```

## CSRF Protection

### CSRF Token Manager

```php
<?php
/**
 * CSRF token protection
 */
class CsrfTokenManager
{
    private const TOKEN_NAME = 'csrf_token';
    private const TOKEN_LENGTH = 32;

    /**
     * Generate new CSRF token
     */
    public static function generate(): string
    {
        $token = bin2hex(random_bytes(self::TOKEN_LENGTH));

        // Store in session
        $_SESSION[self::TOKEN_NAME] = [
            'token' => $token,
            'created' => time(),
            'used' => false,
        ];

        return $token;
    }

    /**
     * Validate CSRF token
     */
    public static function validate(string $token, bool $markUsed = true): bool
    {
        if (!isset($_SESSION[self::TOKEN_NAME])) {
            return false;
        }

        $stored = $_SESSION[self::TOKEN_NAME];

        // Check expiration (1 hour)
        if (time() - $stored['created'] > 3600) {
            self::invalidate();
            return false;
        }

        // Check if already used
        if ($stored['used'] && $markUsed) {
            return false;
        }

        // Timing-safe comparison
        if (!hash_equals($stored['token'], $token)) {
            return false;
        }

        if ($markUsed) {
            $stored['used'] = true;
            $_SESSION[self::TOKEN_NAME] = $stored;
        }

        return true;
    }

    /**
     * Get current token (regenerate if needed)
     */
    public static function getToken(): string
    {
        if (!isset($_SESSION[self::TOKEN_NAME]) ||
            time() - $_SESSION[self::TOKEN_NAME]['created'] > 3600) {
            return self::generate();
        }

        return $_SESSION[self::TOKEN_NAME]['token'];
    }

    /**
     * Invalidate current token
     */
    public static function invalidate(): void
    {
        unset($_SESSION[self::TOKEN_NAME]);
    }
}
```

### Form CSRF Helper

```php
<?php
/**
 * CSRF form helper for templates
 */
function csrfField(): string
{
    $token = CsrfTokenManager::getToken();
    return '<input type="hidden" name="csrf_token" value="' . htmlspecialchars($token) . '">';
}

function csrfMeta(): string
{
    $token = CsrfTokenManager::getToken();
    return '<meta name="csrf-token" content="' . htmlspecialchars($token) . '">';
}
```

### CSRF Validation in Hook

```php
<?php
/**
 * Hook function with CSRF protection
 */
add_hook('AdminAreaPage', 1, function ($vars) {
    if ($_SERVER['REQUEST_METHOD'] !== 'POST') {
        return;
    }

    $token = $_POST['csrf_token'] ?? '';

    if (!CsrfTokenManager::validate($token)) {
        return [
            'error' => 'Invalid or expired security token. Please refresh the page and try again.',
            'csrf_reload' => true,
        ];
    }
});
```

## Input Validation and Sanitization

### Admin Input Sanitizer

```php
<?php
/**
 * Input sanitization for admin operations
 */
class AdminInputSanitizer
{
    /**
     * Sanitize string input
     */
    public static function sanitizeString(string $input, int $maxLength = 255): string
    {
        $input = trim($input);
        $input = strip_tags($input);
        $input = htmlspecialchars($input, ENT_QUOTES | ENT_HTML5, 'UTF-8');
        $input = self::removeInvisibleChars($input);

        return substr($input, 0, $maxLength);
    }

    /**
     * Sanitize integer input
     */
    public static function sanitizeInt($input): int
    {
        return (int) filter_var($input, FILTER_SANITIZE_NUMBER_INT);
    }

    /**
     * Sanitize email input
     */
    public static function sanitizeEmail(string $email): ?string
    {
        $email = filter_var(trim($email), FILTER_SANITIZE_EMAIL);

        if (!filter_var($email, FILTER_VALIDATE_EMAIL)) {
            return null;
        }

        return $email;
    }

    /**
     * Sanitize URL input
     */
    public static function sanitizeUrl(string $url): ?string
    {
        $url = filter_var(trim($url), FILTER_SANITIZE_URL);

        if (!filter_var($url, FILTER_VALIDATE_URL)) {
            return null;
        }

        // Only allow safe protocols
        $parsed = parse_url($url);
        $allowed = ['http', 'https'];

        if (!in_array($parsed['scheme'] ?? '', $allowed)) {
            return null;
        }

        return $url;
    }

    /**
     * Sanitize array input
     */
    public static function sanitizeArray(array $input, string $type = 'string'): array
    {
        $sanitized = [];

        foreach ($input as $key => $value) {
            if (is_array($value)) {
                $sanitized[$key] = self::sanitizeArray($value, $type);
            } else {
                $sanitized[$key] = match ($type) {
                    'int' => self::sanitizeInt($value),
                    'email' => self::sanitizeEmail($value),
                    'url' => self::sanitizeUrl($value),
                    default => self::sanitizeString($value),
                };
            }
        }

        return $sanitized;
    }

    /**
     * Remove invisible/special characters
     */
    private static function removeInvisibleChars(string $input): string
    {
        return preg_replace('/[\x00-\x08\x0B\x0C\x0E-\x1F\x7F]/u', '', $input);
    }
}
```

## Audit Logging

### Admin Action Logger

```php
<?php
/**
 * Admin action audit logger
 */
class AdminAuditLogger
{
    /**
     * Log admin action
     */
    public static function log(
        int $adminId,
        string $action,
        array $context = [],
        ?string $ipAddress = null
    ): void {
        Capsule::table('mod_security_audit_log')->insert([
            'admin_id' => $adminId,
            'action' => $action,
            'context' => json_encode($context),
            'ip_address' => $ipAddress ?? self::getClientIp(),
            'user_agent' => $_SERVER['HTTP_USER_AGENT'] ?? '',
            'created_at' => date('Y-m-d H:i:s'),
        ]);
    }

    /**
     * Log sensitive action
     */
    public static function logSensitive(
        int $adminId,
        string $action,
        array $before,
        array $after,
        string $reason
    ): void {
        self::log($adminId, $action, [
            'before' => $before,
            'after' => $after,
            'reason' => $reason,
            'sensitive' => true,
        ]);
    }

    /**
     * Get audit trail for admin
     */
    public static function getAdminTrail(int $adminId, int $limit = 100): array
    {
        return Capsule::table('mod_security_audit_log')
            ->where('admin_id', $adminId)
            ->orderBy('created_at', 'desc')
            ->limit($limit)
            ->get();
    }

    private static function getClientIp(): string
    {
        $headers = ['HTTP_X_FORWARDED_FOR', 'HTTP_CLIENT_IP', 'REMOTE_ADDR'];

        foreach ($headers as $header) {
            if (!empty($_SERVER[$header])) {
                $ip = explode(',', $_SERVER[$header])[0];
                return trim($ip);
            }
        }

        return 'unknown';
    }
}

// Hook for automatic logging
add_hook('AdminAreaPage', 1, function ($vars) {
    // Log all POST requests to admin area
    if ($_SERVER['REQUEST_METHOD'] === 'POST' && isset($_SESSION['adminid'])) {
        AdminAuditLogger::log($_SESSION['adminid'], 'admin_action', [
            'uri' => $_SERVER['REQUEST_URI'],
            'action' => $_POST['action'] ?? 'unknown',
            'data_keys' => array_keys($_POST),
        ]);
    }
});
```

## Security Checklist

| Security Measure | Implementation | Status |
|-----------------|----------------|--------|
| Strong password policy | Minimum 12 chars, complexity requirements | [ ] |
| Two-factor authentication | TOTP-based 2FA for all admins | [ ] |
| Session timeout | Auto-logout after 30 minutes inactivity | [ ] |
| IP whitelist | Restrict admin access to specific IPs | [ ] |
| CSRF protection | Token validation on all POST requests | [ ] |
| Input sanitization | Validate and escape all user inputs | [ ] |
| Audit logging | Log all sensitive admin actions | [ ] |
| Rate limiting | Limit login attempts and API calls | [ ] |
| HTTPS enforcement | Force HTTPS for admin area | [ ] |
| Security headers | CSP, X-Frame-Options, etc. | [ ] |

## Related Patterns

- [Session Management](./session-management.md) - Session security
- [Client Data Privacy](./client-data-privacy.md) - Data protection
- [Admin Area Customization](./admin-area-customization.md) - Custom admin features