# WHMCS Authentication Implementation Skill
# Version: 1.0 | Updated: 2026-05-29

## Purpose

Guide for implementing secure authentication in WHMCS custom modules and integrations.

## When to Use

- Building custom login systems
- Implementing SSO authentication
- Adding API authentication
- Creating secure admin interfaces

## Authentication Patterns

### 1. Session-Based Authentication

```php
<?php
namespace WHMCS\Auth;

class SessionAuthenticator {
    private string $sessionName = 'WHMPASS';
    private int $sessionLifetime = 7200; // 2 hours
    private string $sessionPath = '/';
    private bool $sessionSecure = true;

    public function __construct() {
        $this->configureSession();
    }

    private function configureSession(): void {
        if (session_status() === PHP_SESSION_NONE) {
            session_name($this->sessionName);
            session_set_cookie_params([
                'lifetime' => $this->sessionLifetime,
                'path' => $this->sessionPath,
                'secure' => $this->sessionSecure,
                'httponly' => true,
                'samesite' => 'Strict',
            ]);
            session_start();
        }
    }

    public function authenticate(string $username, string $password): AuthResult {
        // Sanitize input
        $username = $this->sanitizeUsername($username);

        // Find user
        $user = Capsule::table('tblclients')
            ->where('email', $username)
            ->orWhere('username', $username)
            ->first();

        if (!$user) {
            $this->logFailedAttempt($username, 'user_not_found');
            return new AuthResult(false, 'Invalid credentials');
        }

        // Verify password
        if (!$this->verifyPassword($password, $user->password)) {
            $this->logFailedAttempt($username, 'invalid_password', $user->id);
            return new AuthResult(false, 'Invalid credentials');
        }

        // Check if account is active
        if ($user->status !== 'Active') {
            $this->logFailedAttempt($username, 'account_inactive', $user->id);
            return new AuthResult(false, 'Account is not active');
        }

        // Create session
        return $this->createAuthenticatedSession($user);
    }

    private function verifyPassword(string $plain, string $hashed): bool {
        // WHMCS uses different hash types depending on version
        if (str_starts_with($hashed, '$2a$') || str_starts_with($hashed, '$2y$')) {
            return password_verify($plain, $hashed);
        }

        if (str_starts_with($hashed, '$argon')) {
            return password_verify($plain, $hashed);
        }

        // Legacy DES hash
        return $this->verifyLegacyHash($plain, $hashed);
    }

    private function verifyLegacyHash(string $plain, string $hash): bool {
        $parts = explode(':', $hash);
        if (count($parts) !== 2) {
            return false;
        }

        [$storedHash, $salt] = $parts;
        $calcHash = md5($salt . $plain);

        return hash_equals($storedHash, $calcHash);
    }

    private function createAuthenticatedSession(object $user): AuthResult {
        // Regenerate session ID to prevent fixation
        session_regenerate_id(true);

        $_SESSION['uid'] = $user->id;
        $_SESSION['upw'] = $this->hashSessionToken();
        $_SESSION['adminid'] = null;
        $_SESSION['clientid'] = $user->id;
        $_SESSION['authuser'] = [
            'id' => $user->id,
            'email' => $user->email,
            'name' => trim($user->firstname . ' ' . $user->lastname),
            'logged_in_at' => time(),
        ];

        // Set remember me cookie if requested
        $this->setRememberMeCookie($user->id);

        // Log successful login
        logActivity("User {$user->email} logged in successfully");

        return new AuthResult(true, 'Login successful', [
            'user_id' => $user->id,
            'email' => $user->email,
        ]);
    }

    private function hashSessionToken(): string {
        return hash('sha256', session_id() . $_SERVER['HTTP_USER_AGENT']);
    }

    public function isAuthenticated(): bool {
        if (empty($_SESSION['uid']) || empty($_SESSION['upw'])) {
            return false;
        }

        $expectedToken = $this->hashSessionToken();

        if (!hash_equals($expectedToken, $_SESSION['upw'])) {
            $this->destroySession();
            return false;
        }

        // Check session expiry
        $lastActivity = $_SESSION['authuser']['logged_in_at'] ?? 0;
        if (time() - $lastActivity > $this->sessionLifetime) {
            $this->logout();
            return false;
        }

        return true;
    }

    public function logout(): void {
        $userId = $_SESSION['uid'] ?? null;

        if ($userId) {
            logActivity("User ID $userId logged out");
        }

        $this->destroySession();
        $this->clearRememberMeCookie();
    }

    private function destroySession(): void {
        $_SESSION = [];

        if (ini_get('session.use_cookies')) {
            $params = session_get_cookie_params();
            setcookie(session_name(), '', time() - 42000,
                $params['path'], $params['domain'],
                $params['secure'], $params['httponly']
            );
        }

        session_destroy();
    }

    private function setRememberMeCookie(int $userId): void {
        $token = $this->generateRememberToken();
        $expiry = time() + (86400 * 30); // 30 days

        setcookie('whmcs_remember', $token, $expiry, '/', '', $this->sessionSecure, true);

        // Store token hash in database
        $tokenHash = hash('sha256', $token);
        Capsule::table('mod_auth_tokens')->insert([
            'user_id' => $userId,
            'token_hash' => $tokenHash,
            'expires_at' => date('Y-m-d H:i:s', $expiry),
            'created_at' => date('Y-m-d H:i:s'),
        ]);
    }

    private function generateRememberToken(): string {
        return bin2hex(random_bytes(32));
    }

    private function sanitizeUsername(string $username): string {
        return filter_var($username, FILTER_SANITIZE_EMAIL);
    }

    private function logFailedAttempt(string $username, string $reason, ?int $userId = null): void {
        logActivity("Failed login attempt for $username: $reason");

        Capsule::table('mod_auth_failures')->insert([
            'username' => $username,
            'user_id' => $userId,
            'reason' => $reason,
            'ip_address' => $_SERVER['REMOTE_ADDR'] ?? 'unknown',
            'user_agent' => $_SERVER['HTTP_USER_AGENT'] ?? '',
            'created_at' => date('Y-m-d H:i:s'),
        ]);
    }
}

class AuthResult {
    private bool $success;
    private string $message;
    private array $data;

    public function __construct(bool $success, string $message, array $data = []) {
        $this->success = $success;
        $this->message = $message;
        $this->data = $data;
    }

    public function isSuccess(): bool { return $this->success; }
    public function getMessage(): string { return $this->message; }
    public function getData(): array { return $this->data; }
}
```

### 2. API Key Authentication

```php
<?php
namespace WHMCS\Auth;

class ApiKeyAuthenticator {
    private string $headerName = 'X-API-Key';
    private array $validKeys = [];

    public function __construct() {
        $this->loadApiKeys();
    }

    public function authenticate(): APIAuthResult {
        $apiKey = $this->getApiKey();

        if (!$apiKey) {
            return new APIAuthResult(false, 'API key required', 401);
        }

        if (!in_array($apiKey, $this->validKeys)) {
            $this->logFailedApiAttempt($apiKey, 'invalid_key');
            return new APIAuthResult(false, 'Invalid API key', 401);
        }

        $keyData = $this->getKeyData($apiKey);

        if ($keyData['expires_at'] && strtotime($keyData['expires_at']) < time()) {
            return new APIAuthResult(false, 'API key expired', 401);
        }

        if (!$keyData['active']) {
            return new APIAuthResult(false, 'API key disabled', 401);
        }

        // Log API access
        $this->logApiAccess($keyData['id']);

        return new APIAuthResult(true, 'Authenticated', 200, [
            'key_id' => $keyData['id'],
            'user_id' => $keyData['user_id'],
            'permissions' => $keyData['permissions'],
        ]);
    }

    private function getApiKey(): ?string {
        $headers = getallheaders();

        // Check header
        if (isset($headers[$this->headerName])) {
            return $headers[$this->headerName];
        }

        // Check $_SERVER for prefixed headers
        $serverKey = 'HTTP_' . str_replace('-', '_', strtoupper($this->headerName));
        if (isset($_SERVER[$serverKey])) {
            return $_SERVER[$serverKey];
        }

        // Check query parameter as fallback
        if (isset($_GET['api_key'])) {
            return $_GET['api_key'];
        }

        return null;
    }

    private function loadApiKeys(): void {
        $keys = Capsule::table('mod_api_keys')
            ->where('active', 1)
            ->where(function($q) {
                $q->whereNull('expires_at')
                  ->orWhere('expires_at', '>', date('Y-m-d H:i:s'));
            })
            ->get();

        foreach ($keys as $key) {
            $this->validKeys[] = $key->api_key;
        }
    }

    private function getKeyData(string $apiKey): array {
        return Capsule::table('mod_api_keys')
            ->where('api_key', $apiKey)
            ->first()
            ->toArray();
    }

    private function logApiAccess(int $keyId): void {
        Capsule::table('mod_api_access_log')->insert([
            'key_id' => $keyId,
            'endpoint' => $_SERVER['REQUEST_URI'],
            'method' => $_SERVER['REQUEST_METHOD'],
            'ip_address' => $_SERVER['REMOTE_ADDR'] ?? 'unknown',
            'user_agent' => $_SERVER['HTTP_USER_AGENT'] ?? '',
            'created_at' => date('Y-m-d H:i:s'),
        ]);
    }

    private function logFailedApiAttempt(string $apiKey, string $reason): void {
        logActivity("Failed API authentication: $reason for key " . substr($apiKey, 0, 8) . "...");

        Capsule::table('mod_api_failures')->insert([
            'api_key_prefix' => substr($apiKey, 0, 8),
            'reason' => $reason,
            'ip_address' => $_SERVER['REMOTE_ADDR'] ?? 'unknown',
            'created_at' => date('Y-m-d H:i:s'),
        ]);
    }

    public function generateApiKey(string $name, ?int $userId = null, array $permissions = []): string {
        $apiKey = bin2hex(random_bytes(32));

        Capsule::table('mod_api_keys')->insert([
            'name' => $name,
            'api_key' => $apiKey,
            'user_id' => $userId,
            'permissions' => json_encode($permissions),
            'active' => 1,
            'created_at' => date('Y-m-d H:i:s'),
        ]);

        return $apiKey;
    }

    public function revokeApiKey(string $apiKey): void {
        Capsule::table('mod_api_keys')
            ->where('api_key', $apiKey)
            ->update(['active' => 0]);
    }
}

class APIAuthResult {
    private bool $success;
    private string $message;
    private int $statusCode;
    private array $data;

    public function __construct(bool $success, string $message, int $statusCode, array $data = []) {
        $this->success = $success;
        $this->message = $message;
        $this->statusCode = $statusCode;
        $this->data = $data;
    }

    public function isSuccess(): bool { return $this->success; }
    public function getMessage(): string { return $this->message; }
    public function getStatusCode(): int { return $this->statusCode; }
    public function getData(): array { return $this->data; }
}
```

### 3. JWT Authentication

```php
<?php
namespace WHMCS\Auth;

class JWTAuthenticator {
    private string $secret;
    private string $algorithm = 'HS256';
    private int $expiry = 3600;

    public function __construct() {
        $this->secret = Capsule::table('tblconfiguration')
            ->where('setting', 'jwt_secret')
            ->first()->value ?? '';

        if (empty($this->secret)) {
            $this->secret = bin2hex(random_bytes(32));
        }
    }

    public function generateToken(array $claims): string {
        $payload = array_merge([
            'iat' => time(),
            'exp' => time() + $this->expiry,
            'jti' => bin2hex(random_bytes(16)),
        ], $claims);

        $header = $this->base64UrlEncode(json_encode([
            'alg' => $this->algorithm,
            'typ' => 'JWT',
        ]));

        $payloadEncoded = $this->base64UrlEncode(json_encode($payload));
        $signature = $this->base64UrlEncode(
            hash_hmac($this->getHmacAlgorithm(), "$header.$payloadEncoded", $this->secret, true)
        );

        return "$header.$payloadEncoded.$signature";
    }

    public function validateToken(string $token): JWTResult {
        $parts = explode('.', $token);

        if (count($parts) !== 3) {
            return new JWTResult(false, 'Invalid token format');
        }

        [$header, $payload, $signature] = $parts;

        // Verify signature
        $expectedSignature = $this->base64UrlEncode(
            hash_hmac($this->getHmacAlgorithm(), "$header.$payload", $this->secret, true)
        );

        if (!hash_equals($expectedSignature, $signature)) {
            return new JWTResult(false, 'Invalid signature');
        }

        // Decode payload
        $payloadData = json_decode($this->base64UrlDecode($payload), true);

        if (!$payloadData) {
            return new JWTResult(false, 'Invalid payload encoding');
        }

        // Check expiry
        if (isset($payloadData['exp']) && $payloadData['exp'] < time()) {
            return new JWTResult(false, 'Token expired');
        }

        // Check issuer
        $expectedIssuer = Capsule::table('tblconfiguration')
            ->where('setting', 'SystemURL')
            ->first()->value ?? '';

        if (isset($payloadData['iss']) && $payloadData['iss'] !== $expectedIssuer) {
            return new JWTResult(false, 'Invalid issuer');
        }

        return new JWTResult(true, 'Valid', $payloadData);
    }

    private function base64UrlEncode(string $data): string {
        return rtrim(strtr(base64_encode($data), '+/', '-_'), '=');
    }

    private function base64UrlDecode(string $data): string {
        $padding = strlen($data) % 4;
        if ($padding) {
            $data .= str_repeat('=', 4 - $padding);
        }
        return base64_decode(strtr($data, '-_', '+/'));
    }

    private function getHmacAlgorithm(): string {
        return match($this->algorithm) {
            'HS256' => 'sha256',
            'HS384' => 'sha384',
            'HS512' => 'sha512',
            default => 'sha256',
        };
    }
}

class JWTResult {
    private bool $success;
    private string $message;
    private array $claims;

    public function __construct(bool $success, string $message, array $claims = []) {
        $this->success = $success;
        $this->message = $message;
        $this->claims = $claims;
    }

    public function isSuccess(): bool { return $this->success; }
    public function getMessage(): string { return $this->message; }
    public function getClaims(): array { return $this->claims; }
}
```

### 4. Two-Factor Authentication

```php
<?php
namespace WHMCS\Auth;

class TwoFactorAuth {
    private string $issuer = 'WHMCS';
    private int $digits = 6;
    private int $period = 30;

    public function generateSecret(): string {
        return bin2hex(random_bytes(20));
    }

    public function getQRCodeUrl(string $secret, string $accountName): string {
        $label = rawurlencode("$this->issuer:$accountName");
        $secretBase32 = $this->base32Encode($secret);
        $secretEncoded = rawurlencode($secretBase32);

        return "otpauth://totp/$label?secret=$secretEncoded&issuer=$this->issuer&digits=$this->digits&period=$this->period";
    }

    public function verifyCode(string $secret, string $code): bool {
        $timeSlice = floor(time() / $this->period);
        $codes = [];

        // Check current and adjacent time slices for tolerance
        for ($i = -1; $i <= 1; $i++) {
            $codes[] = $this->generateCode($secret, $timeSlice + $i);
        }

        return in_array($code, $codes);
    }

    private function generateCode(string $secret, int $timeSlice): string {
        $secretBase32 = $this->base32Encode($secret);
        $timeHex = str_pad(dechex($timeSlice), 16, '0', STR_PAD_LEFT);

        $timeBinary = pack('H*', $timeHex);
        $hash = hash_hmac('sha1', $timeBinary, $this->base32Decode($secretBase32), true);

        $offset = ord($hash[19]) & 0xf;
        $binary = (
            ((ord($hash[$offset]) & 0x7f) << 24) |
            ((ord($hash[$offset + 1]) & 0xff) << 16) |
            ((ord($hash[$offset + 2]) & 0xff) << 8) |
            (ord($hash[$offset + 3]) & 0xff)
        );

        $otp = $binary % pow(10, $this->digits);

        return str_pad((string)$otp, $this->digits, '0', STR_PAD_LEFT);
    }

    private function base32Encode(string $data): string {
        $alphabet = 'ABCDEFGHIJKLMNOPQRSTUVWXYZ234567';
        $output = '';

        foreach (str_split($data) as $char) {
            $output .= str_pad(decbin(ord($char)), 8, '0', STR_PAD_LEFT);
        }

        $output = str_pad($output, ceil(strlen($output) / 5) * 5, '0');

        foreach (str_split($output, 5) as $chunk) {
            $output .= $alphabet[bindec($chunk)];
        }

        return $output;
    }

    private function base32Decode(string $data): string {
        $alphabet = 'ABCDEFGHIJKLMNOPQRSTUVWXYZ234567';
        $data = strtoupper(trim($data));
        $output = '';

        foreach (str_split($data) as $char) {
            $pos = strpos($alphabet, $char);
            if ($pos !== false) {
                $output .= str_pad(decbin($pos), 5, '0', STR_PAD_LEFT);
            }
        }

        foreach (str_split($output, 8) as $byte) {
            $output .= chr(bindec($byte));
        }

        return $output;
    }

    public function enableForUser(int $userId, string $secret): void {
        $code = $_POST['code'] ?? '';

        if (!$this->verifyCode($secret, $code)) {
            throw new \Exception('Invalid verification code');
        }

        Capsule::table('tblclients')
            ->where('id', $userId)
            ->update([
                'secret_2fa' => encrypt($secret),
                'twofaenabled' => 1,
            ]);
    }
}
```

### 5. Authentication Middleware

```php
<?php
namespace WHMCS\Auth;

class AuthMiddleware {
    public static function requireAuth(): void {
        $auth = new SessionAuthenticator();

        if (!$auth->isAuthenticated()) {
            header('Location: ' . WHMCS\Config\Setting::getValue('SystemURL') . '/login.php');
            exit;
        }
    }

    public static function requireApiAuth(): void {
        $auth = new ApiKeyAuthenticator();
        $result = $auth->authenticate();

        if (!$result->isSuccess()) {
            http_response_code($result->getStatusCode());
            header('Content-Type: application/json');
            echo json_encode(['status' => 'error', 'message' => $result->getMessage()]);
            exit;
        }
    }

    public static function requirePermission(string $permission): callable {
        return function() use ($permission) {
            if (!self::hasPermission($permission)) {
                http_response_code(403);
                header('Content-Type: application/json');
                echo json_encode(['status' => 'error', 'message' => 'Permission denied']);
                exit;
            }
        };
    }

    public static function hasPermission(string $permission): bool {
        $userId = $_SESSION['uid'] ?? null;

        if (!$userId) {
            return false;
        }

        $permissions = Capsule::table('mod_user_permissions')
            ->where('user_id', $userId)
            ->pluck('permission')
            ->toArray();

        return in_array($permission, $permissions);
    }
}
```

### 6. Database Schema

```php
<?php
function createAuthTables(): void {
    Capsule::schema()->create('mod_api_keys', function($t) {
        $t->increments('id');
        $t->string('name');
        $t->string('api_key', 64)->unique();
        $t->integer('user_id')->unsigned()->nullable();
        $t->text('permissions')->nullable();
        $t->boolean('active')->default(1);
        $t->timestamp('expires_at')->nullable();
        $t->timestamps();
    });

    Capsule::schema()->create('mod_api_access_log', function($t) {
        $t->increments('id');
        $t->integer('key_id')->unsigned();
        $t->string('endpoint');
        $t->string('method', 10);
        $t->string('ip_address', 45);
        $t->string('user_agent')->nullable();
        $t->timestamp('created_at');
    });

    Capsule::schema()->create('mod_auth_tokens', function($t) {
        $t->increments('id');
        $t->integer('user_id')->unsigned();
        $t->string('token_hash', 64);
        $t->timestamp('expires_at');
        $t->timestamp('created_at');
    });

    Capsule::schema()->create('mod_auth_failures', function($t) {
        $t->increments('id');
        $t->string('username');
        $t->integer('user_id')->unsigned()->nullable();
        $t->string('reason');
        $t->string('ip_address', 45);
        $t->string('user_agent')->nullable();
        $t->timestamp('created_at');
    });
}
```

## Checklist

- [ ] Session authentication
- [ ] Password verification (legacy + modern)
- [ ] Remember me functionality
- [ ] API key authentication
- [ ] JWT token generation/validation
- [ ] Two-factor authentication (TOTP)
- [ ] Authentication middleware
- [ ] Failed login tracking
- [ ] Session regeneration on login
- [ ] Secure cookie settings

---

**Related Skills:**
- whmcs-two-factor-auth
- whmcs-sso-integration
- whmcs-security-hardening
- whmcs-security-checklist
