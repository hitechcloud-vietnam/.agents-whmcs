# WHMCS Security Implementation Workflow

## Purpose

Complete guide to implementing security best practices in WHMCS. Covers authentication, authorization, input validation, output escaping, secure coding practices, GDPR compliance, and security monitoring.

## Prerequisites

- WHMCS 7.0+ installation
- SSL/TLS certificate
- PHP 7.4+ with security extensions
- Basic understanding of OWASP guidelines
- Access to security configuration

## Workflow Steps

### Step 1: Authentication Security

Implement multi-layered authentication:

```php
<?php
/**
 * WHMCS Authentication Security
 *
 * Implements enhanced authentication with 2FA, brute force protection,
 * and secure session management.
 */

// ===========================================
// Two-Factor Authentication
// ===========================================

add_hook('Authentication', 1, function(array $vars) {
    $email = $vars['username'] ?? '';

    // Check if 2FA is enabled for this user
    $userId = Capsule::table('tblclients')
        ->where('email', $email)
        ->value('id');

    if (!$userId) {
        return true;
    }

    $twoFactorEnabled = Capsule::table('mod_user_2fa')
        ->where('user_id', $userId)
        ->where('enabled', 1)
        ->exists();

    if ($twoFactorEnabled && empty($_SESSION['2fa_verified'])) {
        // Generate TOTP code
        $secret = Capsule::table('mod_user_2fa')
            ->where('user_id', $userId)
            ->value('secret');

        // Require 2FA verification
        $_SESSION['2fa_required'] = true;
        $_SESSION['2fa_user_id'] = $userId;
        $_SESSION['2fa_secret'] = decrypt($secret);

        return [
            'success' => false,
            'require_2fa' => true,
            'redirect_uri' => '2fa-verify.php',
        ];
    }

    return true;
});

/**
 * TOTP Verification Helper
 */
class TOTPVerification
{
    private $secret;
    private $digits = 6;
    private $period = 30;
    private $algorithm = 'sha1';

    public function __construct(string $secret)
    {
        $this->secret = $secret;
    }

    /**
     * Generate current TOTP
     */
    public function generateCode(): string
    {
        $time = floor(time() / $this->period);
        $hash = hash_hmac($this->algorithm, pack('N*', 0, $time), $this->secret, true);

        $offset = ord($hash[19]) & 0xf;
        $code = (
            ((ord($hash[$offset++]) & 0x7f) << 24) |
            ((ord($hash[$offset++]) & 0xff) << 16) |
            ((ord($hash[$offset++]) & 0xff) << 8) |
            (ord($hash[$offset++]) & 0xff)
        ) % pow(10, $this->digits);

        return str_pad((string) $code, $this->digits, '0', STR_PAD_LEFT);
    }

    /**
     * Verify TOTP code
     */
    public function verifyCode(string $code, int $tolerance = 1): bool
    {
        $currentTime = time();

        // Allow codes within tolerance window
        for ($i = -$tolerance; $i <= $tolerance; $i++) {
            $checkTime = $currentTime + ($i * $this->period);
            $testCode = $this->generateCodeAtTime($checkTime);

            if (hash_equals($this->generateCodeAtTime($checkTime), $code)) {
                return true;
            }
        }

        return false;
    }

    private function generateCodeAtTime(int $time): string
    {
        $time = floor($time / $this->period);
        $hash = hash_hmac($this->algorithm, pack('N*', 0, $time), $this->secret, true);

        $offset = ord($hash[19]) & 0xf;
        $code = (
            ((ord($hash[$offset++]) & 0x7f) << 24) |
            ((ord($hash[$offset++]) & 0xff) << 16) |
            ((ord($hash[$offset++]) & 0xff) << 8) |
            (ord($hash[$offset++]) & 0xff)
        ) % pow(10, $this->digits);

        return str_pad((string) $code, $this->digits, '0', STR_PAD_LEFT);
    }
}

// ===========================================
// Brute Force Protection
// ===========================================

add_hook('Authentication', 1, function(array $vars) {
    $ip = $_SERVER['REMOTE_ADDR'] ?? 'unknown';
    $email = $vars['username'] ?? '';

    // Check login attempts
    $attempts = Capsule::table('mod_login_attempts')
        ->where('ip_address', $ip)
        ->where('created_at', '>', date('Y-m-d H:i:s', time() - 900))
        ->count();

    if ($attempts >= 5) {
        return [
            'success' => false,
            'error' => 'Too many login attempts. Please wait 15 minutes.',
            'blocked_until' => date('Y-m-d H:i:s', time() + 900),
        ];
    }

    return true;
});

add_hook('ClientLogin', 1, function(array $vars) {
    $ip = $_SERVER['REMOTE_ADDR'] ?? 'unknown';

    // Reset failed attempts on successful login
    Capsule::table('mod_login_attempts')
        ->where('ip_address', $ip)
        ->delete();
});

add_hook('FailedAuthentication', 1, function(array $vars) {
    $ip = $_SERVER['REMOTE_ADDR'] ?? 'unknown';
    $reason = $vars['reason'] ?? 'Unknown';

    // Log failed attempt
    Capsule::table('mod_login_attempts')->insert([
        'ip_address' => $ip,
        'email' => $vars['username'] ?? '',
        'reason' => $reason,
        'created_at' => date('Y-m-d H:i:s'),
    ]);
});

// ===========================================
// Secure Session Management
// ===========================================

add_hook('ClientAreaPageRun', 1, function() {
    if (!isset($_SESSION['last_activity'])) {
        $_SESSION['last_activity'] = time();
        return;
    }

    $sessionTimeout = 3600; // 1 hour
    $idleTimeout = 1800; // 30 minutes

    // Check session timeout
    if (time() - $_SESSION['last_activity'] > $sessionTimeout) {
        session_destroy();
        header('Location: login.php?expired=1');
        exit;
    }

    // Check idle timeout
    if (time() - $_SESSION['last_activity'] > $idleTimeout) {
        // Regenerate session ID but allow session continuation
        session_regenerate_id(true);
        $_SESSION['last_activity'] = time();
    }

    $_SESSION['last_activity'] = time();
});

// Generate new session ID periodically
add_hook('ClientAreaPageRun', 1, function() {
    if (!isset($_SESSION['session_generated'])) {
        $_SESSION['session_generated'] = time();
    }

    // Regenerate session ID every 30 minutes
    if (time() - $_SESSION['session_generated'] > 1800) {
        session_regenerate_id(true);
        $_SESSION['session_generated'] = time();
    }
});
```

### Step 2: Input Validation and Sanitization

```php
<?php
/**
 * WHMCS Input Validation Security
 */

class InputValidation
{
    /**
     * Validate and sanitize string input
     */
    public static function sanitizeString($input, array $options = []): string
    {
        $allowHtml = $options['allow_html'] ?? false;
        $maxLength = $options['max_length'] ?? 1000;

        if (is_array($input)) {
            return array_map(function($item) use ($options) {
                return self::sanitizeString($item, $options);
            }, $input);
        }

        $input = trim($input);
        $input = substr($input, 0, $maxLength);

        if (!$allowHtml) {
            $input = htmlspecialchars($input, ENT_QUOTES | ENT_HTML5, 'UTF-8');
        }

        return $input;
    }

    /**
     * Validate email address
     */
    public static function validateEmail(string $email): bool
    {
        return filter_var($email, FILTER_VALIDATE_EMAIL) !== false;
    }

    /**
     * Validate domain name
     */
    public static function validateDomain(string $domain): bool
    {
        // Check for valid domain format
        if (!preg_match('/^[a-zA-Z0-9]([a-zA-Z0-9-]*[a-zA-Z0-9])?(\.[a-zA-Z0-9]([a-zA-Z0-9-]*[a-zA-Z0-9])?)*\.[a-zA-Z]{2,}$/', $domain)) {
            return false;
        }

        // Check for valid TLD
        $tld = explode('.', $domain);
        $tld = end($tld);

        if (strlen($tld) < 2 || strlen($tld) > 10) {
            return false;
        }

        return true;
    }

    /**
     * Validate IP address
     */
    public static function validateIPAddress(string $ip): bool
    {
        return filter_var($ip, FILTER_VALIDATE_IP) !== false;
    }

    /**
     * Validate numeric input
     */
    public static function validateNumeric($input, float $min = null, float $max = null): bool
    {
        if (!is_numeric($input)) {
            return false;
        }

        $value = (float) $input;

        if ($min !== null && $value < $min) {
            return false;
        }

        if ($max !== null && $value > $max) {
            return false;
        }

        return true;
    }

    /**
     * Validate password strength
     */
    public static function validatePasswordStrength(string $password): array
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
     * Sanitize SQL input (additional to parameterized queries)
     */
    public static function sanitizeSQLString(string $input): string
    {
        // Additional sanitization beyond PDO
        $input = str_replace(['\'', '"', ';', '--', '/*', '*/', 'xp_', 'sp_'], '', $input);

        return trim($input);
    }

    /**
     * Validate file upload
     */
    public static function validateFileUpload(array $file, array $allowedTypes = [], int $maxSize = 10485760): array
    {
        $errors = [];

        // Check for upload errors
        if ($file['error'] !== UPLOAD_ERR_OK) {
            switch ($file['error']) {
                case UPLOAD_ERR_INI_SIZE:
                    $errors[] = 'File exceeds server upload limit';
                    break;
                case UPLOAD_ERR_FORM_SIZE:
                    $errors[] = 'File exceeds form upload limit';
                    break;
                case UPLOAD_ERR_PARTIAL:
                    $errors[] = 'File was only partially uploaded';
                    break;
                case UPLOAD_ERR_NO_FILE:
                    $errors[] = 'No file was uploaded';
                    break;
                default:
                    $errors[] = 'Unknown upload error';
            }
            return $errors;
        }

        // Check file size
        if ($file['size'] > $maxSize) {
            $errors[] = 'File size exceeds maximum allowed (' . ($maxSize / 1048576) . 'MB)';
        }

        // Check file type
        if (!empty($allowedTypes)) {
            $finfo = finfo_open(FILEINFO_MIME_TYPE);
            $mimeType = finfo_file($finfo, $file['tmp_name']);
            finfo_close($finfo);

            if (!in_array($mimeType, $allowedTypes)) {
                $errors[] = 'File type not allowed: ' . $mimeType;
            }
        }

        // Additional checks for executable files
        $forbiddenExtensions = ['php', 'phtml', 'php3', 'php4', 'php5', 'php7', 'phps', 'cgi', 'pl', 'py', 'sh', 'bash'];
        $extension = strtolower(pathinfo($file['name'], PATHINFO_EXTENSION));

        if (in_array($extension, $forbiddenExtensions)) {
            $errors[] = 'This file type is not allowed for security reasons';
        }

        return $errors;
    }
}

/**
 * Hook for validating module inputs
 */
add_hook('ServiceCustomFields', 1, function(array $vars) {
    $errors = [];

    foreach ($_POST as $key => $value) {
        if (strpos($key, 'customfield_') === 0) {
            $fieldId = str_replace('customfield_', '', $key);

            // Get field requirements
            $field = Capsule::table('tblcustomfields')
                ->where('id', $fieldId)
                ->first();

            if ($field) {
                switch ($field->fieldtype) {
                    case 'email':
                        if (!InputValidation::validateEmail($value)) {
                            $errors[] = 'Invalid email address for ' . $field->fieldname;
                        }
                        break;

                    case 'domain':
                        if (!InputValidation::validateDomain($value)) {
                            $errors[] = 'Invalid domain name for ' . $field->fieldname;
                        }
                        break;

                    default:
                        $value = InputValidation::sanitizeString($value);
                }
            }
        }
    }

    if (!empty($errors)) {
        return ['error' => implode(', ', $errors)];
    }

    return true;
});
```

### Step 3: Output Escaping and XSS Prevention

```php
<?php
/**
 * WHMCS Output Escaping Security
 *
 * Comprehensive output escaping for XSS prevention.
 */

class OutputEscaping
{
    /**
     * Escape for HTML context
     */
    public static function html($string): string
    {
        return htmlspecialchars($string, ENT_QUOTES | ENT_HTML5, 'UTF-8');
    }

    /**
     * Escape for JavaScript context
     */
    public static function js($string): string
    {
        return json_encode($string, JSON_HEX_TAG | JSON_HEX_APOS | JSON_HEX_QUOT | JSON_HEX_AMP);
    }

    /**
     * Escape for URL context
     */
    public static function url($string): string
    {
        return rawurlencode($string);
    }

    /**
     * Escape for CSS context
     */
    public static function css($string): string
    {
        // Remove anything that could be used for CSS injection
        return preg_replace('/[^a-zA-Z0-9_-]/', '', $string);
    }

    /**
     * Escape for attribute context
     */
    public static function attribute($string): string
    {
        return htmlspecialchars($string, ENT_QUOTES, 'UTF-8');
    }

    /**
     * Apply escaping based on context in templates
     */
    public static function escapeAll($variable, $escapeHtml = true)
    {
        if (is_array($variable)) {
            return array_map([self::class, 'escapeAll'], $variable, array_fill(0, count($variable), $escapeHtml));
        }

        if ($escapeHtml) {
            return self::html($variable);
        }

        return $variable;
    }
}

/**
 * Content Security Policy Header
 */
add_hook('ClientAreaHeaderOutput', 1, function() {
    $csp = [
        "default-src 'self'",
        "script-src 'self' 'unsafe-inline' https://cdn.example.com",
        "style-src 'self' 'unsafe-inline' https://fonts.googleapis.com",
        "font-src 'self' https://fonts.gstatic.com",
        "img-src 'self' data: https:",
        "connect-src 'self' https://api.example.com",
        "frame-src 'none'",
        "object-src 'none'",
        "base-uri 'self'",
        "form-action 'self'",
    ];

    return '<meta http-equiv="Content-Security-Policy" content="' . implode('; ', $csp) . '">' . "\n";
});

/**
 * HTTP Security Headers
 */
add_hook('ClientAreaPageRun', 1, function() {
    // Prevent clickjacking
    header('X-Frame-Options: DENY');

    // XSS Protection
    header('X-XSS-Protection: 1; mode=block');

    // Prevent MIME type sniffing
    header('X-Content-Type-Options: nosniff');

    // Referrer Policy
    header('Referrer-Policy: strict-origin-when-cross-origin');

    // Permissions Policy
    header('Permissions-Policy: camera=(), microphone=(), geolocation=()');
});
```

### Step 4: SQL Injection Prevention

```php
<?php
/**
 * WHMCS SQL Injection Prevention
 *
 * Secure database access practices.
 */

class SecureDatabase
{
    /**
     * Execute parameterized query safely
     */
    public static function query(string $sql, array $bindings = []): \PDOStatement
    {
        $pdo = Capsule::connection()->getPdo();

        $stmt = $pdo->prepare($sql);

        foreach ($bindings as $key => $value) {
            $param = is_int($key) ? $key + 1 : $key;
            $type = match (true) {
                is_int($value) => \PDO::PARAM_INT,
                is_bool($value) => \PDO::PARAM_BOOL,
                is_null($value) => \PDO::PARAM_NULL,
                default => \PDO::PARAM_STR,
            };

            $stmt->bindValue($param, $value, $type);
        }

        $stmt->execute();

        return $stmt;
    }

    /**
     * Insert with automatic parameter binding
     */
    public static function insert(string $table, array $data): int
    {
        $columns = implode(', ', array_keys($data));
        $placeholders = implode(', ', array_fill(0, count($data), '?'));

        $sql = "INSERT INTO {$table} ({$columns}) VALUES ({$placeholders})";

        self::query($sql, array_values($data));

        return (int) Capsule::connection()->getPdo()->lastInsertId();
    }

    /**
     * Update with automatic parameter binding
     */
    public static function update(string $table, array $data, array $where): int
    {
        $setParts = [];
        $values = [];

        foreach ($data as $column => $value) {
            $setParts[] = "{$column} = ?";
            $values[] = $value;
        }

        $whereParts = [];
        foreach ($where as $column => $value) {
            $whereParts[] = "{$column} = ?";
            $values[] = $value;
        }

        $sql = "UPDATE {$table} SET " . implode(', ', $setParts) . " WHERE " . implode(' AND ', $whereParts);

        $stmt = self::query($sql, $values);

        return $stmt->rowCount();
    }

    /**
     * Get with automatic parameter binding
     */
    public static function select(string $table, array $where, array $columns = ['*']): array
    {
        $columnList = implode(', ', $columns);

        $whereParts = [];
        $values = [];

        foreach ($where as $column => $value) {
            $whereParts[] = "{$column} = ?";
            $values[] = $value;
        }

        $sql = "SELECT {$columnList} FROM {$table} WHERE " . implode(' AND ', $whereParts);

        $stmt = self::query($sql, $values);

        return $stmt->fetchAll(\PDO::FETCH_ASSOC);
    }
}

/**
 * Usage examples - All queries use parameterized statements
 */

// UNSAFE - Never do this:
$email = $_POST['email'];
$sql = "SELECT * FROM tblclients WHERE email = '$email'"; // SQL INJECTION RISK!

// SAFE - Use parameterized queries:
$sql = "SELECT * FROM tblclients WHERE email = ?";
$result = Capsule::table('tblclients')
    ->where('email', $_POST['email'])
    ->first();

// Or with raw query:
$stmt = Capsule::connection()->prepare($sql);
$stmt->execute([$_POST['email']]);
$result = $stmt->fetch();
```

### Step 5: CSRF Protection

```php
<?php
/**
 * WHMCS CSRF Protection
 */

class CSRFProtection
{
    private static $tokenName = 'csrf_token';
    private static $sessionKey = 'csrf_tokens';

    /**
     * Generate CSRF token
     */
    public static function generate(): string
    {
        if (!isset($_SESSION[self::$sessionKey])) {
            $_SESSION[self::$sessionKey] = [];
        }

        $token = bin2hex(random_bytes(32));

        $_SESSION[self::$sessionKey][$token] = [
            'created' => time(),
            'used' => false,
        ];

        // Clean old tokens (older than 2 hours)
        foreach ($_SESSION[self::$sessionKey] as $key => $data) {
            if (time() - $data['created'] > 7200) {
                unset($_SESSION[self::$sessionKey][$key]);
            }
        }

        return $token;
    }

    /**
     * Validate CSRF token
     */
    public static function validate(string $token, bool $consume = true): bool
    {
        if (!isset($_SESSION[self::$sessionKey][$token])) {
            return false;
        }

        $data = $_SESSION[self::$sessionKey][$token];

        // Check if token is too old (15 minutes)
        if (time() - $data['created'] > 900) {
            unset($_SESSION[self::$sessionKey][$token]);
            return false;
        }

        // Mark as used if consume is true
        if ($consume) {
            $_SESSION[self::$sessionKey][$token]['used'] = true;
        }

        return true;
    }

    /**
     * Get token input HTML
     */
    public static function getInput(): string
    {
        $token = self::generate();

        return '<input type="hidden" name="' . self::$tokenName . '" value="' . OutputEscaping::html($token) . '">';
    }

    /**
     * Get token value
     */
    public static function getToken(): string
    {
        return self::generate();
    }
}

/**
 * Add CSRF token to admin forms
 */
add_hook('AdminAreaHeaderOutput', 1, function() {
    $token = CSRFProtection::getToken();

    return '<script>
        window.csrfToken = "' . OutputEscaping::js($token) . '";
    </script>';
});

/**
 * Hook for validating module CSRF tokens
 */
add_hook('AddonModuleOptionsUpdate', 1, function($vars) {
    if ($_SERVER['REQUEST_METHOD'] !== 'POST') {
        return true;
    }

    $token = $_POST['csrf_token'] ?? $_POST['token'] ?? '';

    if (!CSRFProtection::validate($token)) {
        return [
            'error' => 'Invalid or expired security token. Please refresh the page and try again.',
        ];
    }

    return true;
});
```

### Step 6: Security Logging and Monitoring

```php
<?php
/**
 * WHMCS Security Monitoring
 */

// ===========================================
// Security Event Logging
// ===========================================

class SecurityLogger
{
    private $table = 'mod_security_events';

    public function __construct()
    {
        if (!Capsule::schema()->hasTable($this->table)) {
            $this->createTable();
        }
    }

    private function createTable(): void
    {
        Capsule::schema()->create($this->table, function($table) {
            $table->increments('id');
            $table->string('event_type', 50);
            $table->string('severity', 20);
            $table->integer('user_id')->nullable();
            $table->string('ip_address', 50);
            $table->text('details');
            $table->string('user_agent', 500)->nullable();
            $table->timestamp('created_at')->useCurrent();
            $table->index('event_type');
            $table->index('created_at');
        });
    }

    /**
     * Log security event
     */
    public function log(
        string $eventType,
        string $severity,
        string $details,
        int $userId = null,
        string $ipAddress = null
    ): void {
        Capsule::table($this->table)->insert([
            'event_type' => $eventType,
            'severity' => $severity,
            'user_id' => $userId,
            'ip_address' => $ipAddress ?? ($_SERVER['REMOTE_ADDR'] ?? 'unknown'),
            'details' => $details,
            'user_agent' => $_SERVER['HTTP_USER_AGENT'] ?? null,
            'created_at' => date('Y-m-d H:i:s'),
        ]);

        // Alert on critical events
        if ($severity === 'critical') {
            $this->sendSecurityAlert($eventType, $details);
        }
    }

    /**
     * Get security events
     */
    public function getEvents(
        string $eventType = null,
        int $limit = 100,
        int $offset = 0
    ): array {
        $query = Capsule::table($this->table)
            ->orderBy('created_at', 'desc')
            ->limit($limit)
            ->offset($offset);

        if ($eventType) {
            $query->where('event_type', $eventType);
        }

        return $query->get()->toArray();
    }

    private function sendSecurityAlert(string $eventType, string $details): void
    {
        // Send email to security team
        $admins = Capsule::table('tbladmins')
            ->where('roleid', 1) // Admin role
            ->where('disabled', 0)
            ->get();

        foreach ($admins as $admin) {
            sendEmail(
                'Security Alert',
                $admin->email,
                [
                    'event_type' => $eventType,
                    'details' => $details,
                    'ip_address' => $_SERVER['REMOTE_ADDR'] ?? 'unknown',
                    'timestamp' => date('Y-m-d H:i:s'),
                ]
            );
        }
    }
}

/**
 * Hook into failed login attempts
 */
add_hook('FailedAuthentication', 1, function(array $vars) {
    $logger = new SecurityLogger();

    $logger->log(
        'failed_login',
        $vars['reason'] === 'invalid_password' ? 'warning' : 'high',
        'Failed login attempt: ' . ($vars['reason'] ?? 'Unknown reason'),
        null,
        $_SERVER['REMOTE_ADDR']
    );

    // Check for brute force patterns
    $recentAttempts = Capsule::table('mod_security_events')
        ->where('event_type', 'failed_login')
        ->where('ip_address', $_SERVER['REMOTE_ADDR'])
        ->where('created_at', '>', date('Y-m-d H:i:s', time() - 3600))
        ->count();

    if ($recentAttempts >= 10) {
        $logger->log(
            'brute_force_detected',
            'critical',
            "Possible brute force attack from IP: {$_SERVER['REMOTE_ADDR']} ({$recentAttempts} attempts)",
            null,
            $_SERVER['REMOTE_ADDR']
        );
    }
});

/**
 * Hook into successful login
 */
add_hook('ClientLogin', 1, function(array $vars) {
    $logger = new SecurityLogger();

    $logger->log(
        'successful_login',
        'info',
        'Successful login',
        $vars['user_id']
    );
});

/**
 * Hook into permission denied events
 */
add_hook('ApiAuthorization', 1, function(array $vars) {
    if (!($vars['authorized'] ?? true)) {
        $logger = new SecurityLogger();

        $logger->log(
            'api_access_denied',
            'medium',
            'API access denied: ' . ($vars['error'] ?? 'Unknown'),
            $vars['user_id'] ?? null,
            $_SERVER['REMOTE_ADDR']
        );
    }
});
```

## Security Verification Checklist

```
Authentication:
□ 2FA enforced for admin accounts
□ Password strength requirements enforced
□ Brute force protection enabled
□ Session timeout configured
□ Session regeneration enabled
□ Secure password storage (bcrypt/argon2)

Input Validation:
□ All user input validated before processing
□ SQL injection prevention (parameterized queries)
□ XSS prevention (output escaping)
□ CSRF tokens on all forms
□ File upload validation
□ Content type checking

Authorization:
□ Role-based access control
□ Principle of least privilege
□ API key permissions scope-limited
□ IP whitelisting configured

Monitoring:
□ Security event logging enabled
□ Failed login tracking
□ Suspicious activity alerts
□ Audit trail for sensitive operations
□ Log retention configured

Compliance:
□ GDPR data handling
□ Data retention policies
□ Right to erasure implemented
□ Data export capabilities
□ Cookie consent management
```

## Common Security Pitfalls to Avoid

1. **Not validating input** - Always validate at the entry point
2. **Direct SQL queries** - Use parameterized queries only
3. **Output without escaping** - Always escape output
4. **Missing CSRF protection** - Add tokens to all forms
5. **Weak passwords** - Enforce strong password policies
6. **No session timeout** - Implement session expiration
7. **Logging sensitive data** - Never log passwords/tokens
8. **Unprotected API endpoints** - Require authentication
9. **Missing HTTPS** - Force SSL/TLS everywhere
10. **Outdated software** - Keep WHMCS and modules updated

## WHMCS ClassDocs References

- [Security Guidelines](https://developers.whmcs.com/advanced/security-guidelines/)
- [Input Validation](https://developers.whmcs.com/advanced/input-handling/)
- [Authentication Hooks](https://developers.whmcs.com/advanced/hooks-reference/)
- [logActivity()](https://developers.whmcs.com/advanced/logging/)
- [encrypt()](https://developers.whmcs.com/advanced/encryption/)
