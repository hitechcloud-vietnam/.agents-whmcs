# WHMCS Authentication Setup Workflow
# Version: 1.0 | Created: 2026-05-28

## Purpose

Comprehensive guide to setting up and configuring authentication in WHMCS including admin login security, two-factor authentication, SSO integration, password policies, and session management.

## Prerequisites

- WHMCS installation with admin access
- SSL certificate configured
- Understanding of authentication protocols
- Optional: LDAP directory or SSO provider

## Workflow Steps

### Step 1: Configure WHMCS Admin Authentication

Secure admin area access:

```php
// includes/hooks/auth_security.php

/**
 * Force strong passwords for admin users
 */
add_hook('AdminLogin', 1, function(array $vars) {
    $admin = Capsule::table('tbladmins')
        ->where('id', $vars['admin_id'])
        ->first();

    // Check password strength
    if (!$this->isPasswordStrong($admin->password, 12)) {
        logActivity("Weak password detected for admin: " . $admin->username);
    }
});

private function isPasswordStrong(string $password, int $minLength): bool
{
    if (strlen($password) < $minLength) {
        return false;
    }

    $patterns = [
        '/[A-Z]/',      // uppercase
        '/[a-z]/',      // lowercase
        '/[0-9]/',      // digits
        '/[^A-Za-z0-9]/', // special chars
    ];

    $matches = 0;
    foreach ($patterns as $pattern) {
        if (preg_match($pattern, $password)) {
            $matches++;
        }
    }

    return $matches >= 3;
}

/**
 * Monitor failed login attempts
 */
add_hook('AdminLogin', 1, function(array $vars) {
    $ip = $_SERVER['REMOTE_ADDR'];

    if ($vars['success'] === false) {
        $attempts = Capsule::table('mod_login_attempts')
            ->where('ip_address', $ip)
            ->where('created_at', '>', date('Y-m-d H:i:s', strtotime('-15 minutes')))
            ->count();

        Capsule::table('mod_login_attempts')->insert([
            'ip_address'  => $ip,
            'username'    => $_POST['username'],
            'created_at'   => date('Y-m-d H:i:s'),
        ]);

        if ($attempts >= 5) {
            // Block IP temporarily
            Capsule::table('mod_ip_blacklist')->insert([
                'ip_address'  => $ip,
                'blocked_until'=> date('Y-m-d H:i:s', strtotime('+30 minutes')),
                'reason'       => 'Too many failed login attempts',
            ]);
        }
    }
});
```

### Step 2: Enable Two-Factor Authentication

Configure 2FA for enhanced security:

```php
// includes/hooks/two_factor_auth.php

use WHMCS\User\User;
use WHMCS\Utility\Environment\WebSystem;

/**
 * Enforce 2FA for admin users
 */
add_hook('AdminAreaPageHook', 1, function() {
    if (!WebSystem::isAdminArea()) {
        return;
    }

    $adminId = $_SESSION['adminid'] ?? null;

    if (!$adminId) {
        return;
    }

    $admin = Capsule::table('tbladmins')->where('id', $adminId)->first();

    // Check if 2FA is enabled for this admin
    $twoFactorEnabled = Capsule::table('tbladmin_security')
        ->where('admin_id', $adminId)
        ->where('two_factor_enabled', 1)
        ->exists();

    if (!$twoFactorEnabled && !$this->isSetupPage()) {
        // Show 2FA setup prompt
        $this->showTwoFactorSetupPrompt($admin);
    }
});

/**
 * Generate 2FA secret and QR code
 */
public function generateTwoFactorSecret(int $adminId): array
{
    $secret = $this->generateRandomSecret(32);
    $admin = Capsule::table('tbladmins')->where('id', $adminId)->first();

    // Generate TOTP URI for authenticator apps
    $totpUri = sprintf(
        'otpauth://totp/%s:%s?secret=%s&issuer=%s',
        rawurlencode($_SERVER['SERVER_NAME']),
        rawurlencode($admin->username),
        $secret,
        rawurlencode('WHMCS Admin')
    );

    // Generate QR code URL
    $qrCodeUrl = 'https://api.qrserver.com/v1/create-qr-code/?' . http_build_query([
        'size'   => '200x200',
        'data'   => $totpUri,
        'ecc'    => 'M',
    ]);

    return [
        'secret'    => $secret,
        'totp_uri'  => $totpUri,
        'qr_code'   => $qrCodeUrl,
    ];
}

private function generateRandomSecret(int $length): string
{
    $chars = 'ABCDEFGHIJKLMNOPQRSTUVWXYZ234567';
    $secret = '';

    for ($i = 0; $i < $length; $i++) {
        $secret .= $chars[random_int(0, strlen($chars) - 1)];
    }

    return $secret;
}

/**
 * Verify 2FA code
 */
public function verifyTwoFactorCode(string $secret, string $code): bool
{
    $timeSlice = (int)(time() / 30);

    // Check current and adjacent time slices for clock skew tolerance
    for ($i = -1; $i <= 1; $i++) {
        $expectedCode = $this->generateTOTP($secret, $timeSlice + $i);

        if (hash_equals($expectedCode, $code)) {
            return true;
        }
    }

    return false;
}

private function generateTOTP(string $secret, int $timeSlice): string
{
    $secretKey = base32_decode($secret);
    $timePack = pack('N*', 0) . pack('N*', $timeSlice);
    $hash = hash_hmac('sha1', $timePack, $secretKey, true);

    $offset = ord($hash[19]) & 0xf;
    $binary = (
        (ord($hash[$offset + 0]) & 0x7f) << 24 |
        (ord($hash[$offset + 1]) & 0xff) << 16 |
        (ord($hash[$offset + 2]) & 0xff) << 8 |
        (ord($hash[$offset + 3]) & 0xff)
    ) % 100000000;

    return str_pad($binary, 8, '0', STR_PAD_LEFT);
}

private function base32_decode(string $encoded): string
{
    $base32Chars = 'ABCDEFGHIJKLMNOPQRSTUVWXYZ234567';
    $encoded = strtoupper($encoded);
    $encoded = str_replace(['=', ' '], '', $encoded);

    $binaryString = '';
    for ($i = 0; $i < strlen($encoded); $i++) {
        $val = strpos($base32Chars, $encoded[$i]);
        $binaryString .= str_pad(decbin($val), 5, '0', STR_PAD_LEFT);
    }

    $binaryString = rtrim($binaryString, '0');
    $binary = '';
    for ($i = 0; $i < strlen($binaryString); $i += 8) {
        $binary .= chr(bindec(substr($binaryString, $i, 8)));
    }

    return $binary;
}

/**
 * Generate backup codes
 */
public function generateBackupCodes(int $adminId): array
{
    $codes = [];

    for ($i = 0; $i < 10; $i++) {
        $code = bin2hex(random_bytes(4)) . '-' . bin2hex(random_bytes(4));
        $codes[] = $code;
    }

    // Store hashed backup codes
    foreach ($codes as $code) {
        Capsule::table('mod_backup_codes')->insert([
            'admin_id'    => $adminId,
            'code_hash'   => password_hash($code, PASSWORD_DEFAULT),
            'used'        => 0,
            'created_at'  => date('Y-m-d H:i:s'),
        ]);
    }

    return $codes; // Return plain codes (only shown once)
}
```

### Step 3: Implement SSO Integration

Set up Single Sign-On for seamless authentication:

```php
// includes/hooks/sso_integration.php

/**
 * SAML SSO Integration Hook
 */
add_hook('SamlLogin', 1, function(array $vars) {
    $samlResponse = $vars['saml_response'];

    // Verify SAML signature
    if (!$this->verifySamlResponse($samlResponse)) {
        logActivity('Invalid SAML response signature');
        return ['error' => 'Invalid SAML response'];
    }

    // Extract user attributes
    $attributes = $this->parseSamlAttributes($samlResponse);

    // Find or create WHMCS user
    $client = Capsule::table('tblclients')
        ->where('email', $attributes['email'])
        ->first();

    if (!$client) {
        // Auto-create client based on SSO attributes
        $clientId = $this->createClientFromSSO($attributes);
        $client = Capsule::table('tblclients')->where('id', $clientId)->first();
    }

    // Log SSO login
    logActivity("SSO login for client: " . $client->email);

    return [
        'user_id' => $client->id,
        'email'   => $client->email,
    ];
});

private function verifySamlResponse(string $samlResponse): bool
{
    $dom = new DOMDocument();
    $dom->loadXML(base64_decode($samlResponse));

    $xpath = new DOMXPath($dom);
    $xpath->registerNamespace('ds', 'http://www.w3.org/2000/09/xmldsig#');
    $xpath->registerNamespace('saml', 'urn:oasis:names:tc:SAML:2.0:assertion');

    // Get signature element
    $signatureNode = $xpath->query('//ds:Signature')->item(0);

    if (!$signatureNode) {
        return false;
    }

    // Verify signature using IdP certificate
    $idpCert = Capsule::table('tblconfiguration')
        ->where('setting', 'SSO_IdpCertificate')
        ->first()->value ?? '';

    // ... signature verification logic ...
    return true;
}

private function parseSamlAttributes(DOMDocument $dom): array
{
    $xpath = new DOMXPath($dom);
    $xpath->registerNamespace('saml', 'urn:oasis:names:tc:SAML:2.0:assertion');

    $attributes = [];

    // Extract standard attributes
    $nameId = $xpath->query('//saml:NameID')->item(0);
    if ($nameId) {
        $attributes['email'] = $nameId->textContent;
    }

    // Extract attribute statement
    $attrs = $xpath->query('//saml:Attribute');
    foreach ($attrs as $attr) {
        $name = $attr->getAttribute('Name');
        $value = $xpath->query('saml:AttributeValue', $attr)->item(0);
        $attributes[$name] = $value ? $value->textContent : '';
    }

    return $attributes;
}

private function createClientFromSSO(array $attributes): int
{
    $firstname = $attributes['firstName'] ?? 'SSO';
    $lastname = $attributes['lastName'] ?? 'User';

    return Capsule::table('tblclients')->insertGetId([
        'email'       => $attributes['email'],
        'firstname'   => $firstname,
        'lastname'    => $lastname,
        'companyname' => $attributes['company'] ?? '',
        'datecreated' => date('Y-m-d H:i:s'),
        'status'      => 'Active',
    ]);
}

/**
 * OAuth2 Client Authentication
 */
class OAuth2ClientAuthentication
{
    public function authenticateClient(string $clientId, string $clientSecret): array
    {
        // Validate client credentials
        $client = Capsule::table('mod_oauth_clients')
            ->where('client_id', $clientId)
            ->first();

        if (!$client) {
            return ['error' => 'invalid_client', 'error_description' => 'Unknown client'];
        }

        if (!password_verify($clientSecret, $client->client_secret_hash)) {
            return ['error' => 'invalid_client', 'error_description' => 'Invalid credentials'];
        }

        // Generate access token
        $accessToken = bin2hex(random_bytes(32));
        $refreshToken = bin2hex(random_bytes(32));
        $expiresIn = 3600;

        // Store token
        Capsule::table('mod_oauth_access_tokens')->insert([
            'client_id'     => $clientId,
            'access_token'  => hash('sha256', $accessToken),
            'refresh_token' => hash('sha256', $refreshToken),
            'expires_at'    => date('Y-m-d H:i:s', time() + $expiresIn),
            'created_at'    => date('Y-m-d H:i:s'),
        ]);

        return [
            'access_token'  => $accessToken,
            'token_type'    => 'Bearer',
            'expires_in'    => $expiresIn,
            'refresh_token' => $refreshToken,
        ];
    }

    public function validateAccessToken(string $accessToken): ?array
    {
        $tokenHash = hash('sha256', $accessToken);

        $token = Capsule::table('mod_oauth_access_tokens')
            ->where('access_token', $tokenHash)
            ->where('expires_at', '>', date('Y-m-d H:i:s'))
            ->first();

        if (!$token) {
            return null;
        }

        return [
            'client_id' => $token->client_id,
            'user_id'   => $token->user_id,
        ];
    }
}
```

### Step 4: Configure Password Policies

Implement strong password enforcement:

```php
// includes/hooks/password_policy.php

/**
 * Enforce password policy on client registration
 */
add_hook('ClientRegister', 1, function(array $vars) {
    $password = $_POST['password'] ?? '';

    $validation = $this->validatePasswordPolicy($password);

    if (!$validation['valid']) {
        return [
            'error' => implode(', ', $validation['errors']),
            'errorcode' => 'WEAK_PASSWORD',
        ];
    }
});

/**
 * Enforce password policy on admin password change
 */
add_hook('AdminPasswordChange', 1, function(array $vars) {
    $adminId = $_SESSION['adminid'];
    $newPassword = $vars['new_password'];

    $validation = $this->validatePasswordPolicy($newPassword, true);

    if (!$validation['valid']) {
        return ['error' => implode(', ', $validation['errors'])];
    }
});

public function validatePasswordPolicy(string $password, bool $isAdmin = false): array
{
    $errors = [];
    $config = $this->getPasswordPolicyConfig($isAdmin);

    // Minimum length
    if (strlen($password) < $config['min_length']) {
        $errors[] = "Password must be at least {$config['min_length']} characters";
    }

    // Maximum length
    if (strlen($password) > $config['max_length']) {
        $errors[] = "Password must not exceed {$config['max_length']} characters";
    }

    // Uppercase requirement
    if ($config['require_uppercase'] && !preg_match('/[A-Z]/', $password)) {
        $errors[] = 'Password must contain at least one uppercase letter';
    }

    // Lowercase requirement
    if ($config['require_lowercase'] && !preg_match('/[a-z]/', $password)) {
        $errors[] = 'Password must contain at least one lowercase letter';
    }

    // Number requirement
    if ($config['require_number'] && !preg_match('/[0-9]/', $password)) {
        $errors[] = 'Password must contain at least one number';
    }

    // Special character requirement
    if ($config['require_special'] && !preg_match('/[^A-Za-z0-9]/', $password)) {
        $errors[] = 'Password must contain at least one special character';
    }

    // Check against common passwords list
    if ($this->isCommonPassword($password)) {
        $errors[] = 'This password is too common. Please choose a stronger password.';
    }

    // Check password history
    if ($this->isPasswordReused($password, $_SESSION['uid'])) {
        $errors[] = 'You cannot reuse your last ' . $config['history_count'] . ' passwords';
    }

    return [
        'valid' => empty($errors),
        'errors' => $errors,
    ];
}

private function getPasswordPolicyConfig(bool $isAdmin): array
{
    // Default policies
    $adminDefaults = [
        'min_length'      => 12,
        'max_length'      => 128,
        'require_uppercase' => true,
        'require_lowercase' => true,
        'require_number'  => true,
        'require_special' => true,
        'history_count'   => 5,
    ];

    $clientDefaults = [
        'min_length'      => 8,
        'max_length'      => 72,
        'require_uppercase' => false,
'require_lowercase'  => true,
        'require_number'  => true,
        'require_special' => false,
        'history_count'   => 3,
    ];

    return $isAdmin ? $adminDefaults : $clientDefaults;
}
```

### Step 5: Session Security Configuration

Configure secure session handling:

```php
// includes/classes/SessionSecurity.php

class SessionSecurity
{
    private int $sessionLifetime = 7200; // 2 hours
    private int $absoluteTimeout = 28800; // 8 hours
    private array $trustedIps = [];

    /**
     * Initialize secure session
     */
    public function initSecureSession(): void
    {
        // Set secure session parameters
        ini_set('session.cookie_secure', true);
        ini_set('session.cookie_httponly', true);
        ini_set('session.cookie_samesite', 'Strict');
        ini_set('session.use_strict_mode', true);
        ini_set('session.cookie_lifetime', 0);

        // Regenerate session ID periodically
        if (!isset($_SESSION['last_regeneration'])) {
            $_SESSION['last_regeneration'] = time();
        }

        if (time() - $_SESSION['last_regeneration'] > 300) {
            session_regenerate_id(true);
            $_SESSION['last_regeneration'] = time();
        }

        // Check for session hijacking
        $this->validateSessionFingerprint();
    }

    /**
     * Validate session fingerprint
     */
    private function validateSessionFingerprint(): void
    {
        if (!isset($_SESSION['fingerprint'])) {
            $_SESSION['fingerprint'] = $this->generateFingerprint();
            return;
        }

        $currentFingerprint = $this->generateFingerprint();

        if (!hash_equals($_SESSION['fingerprint'], $currentFingerprint)) {
            logActivity('Session fingerprint mismatch - possible hijacking attempt');
            $this->destroySession();
            header('Location: /login.php');
            exit;
        }
    }

    /**
     * Generate browser fingerprint
     */
    private function generateFingerprint(): string
    {
        $components = [
            $_SERVER['HTTP_USER_AGENT'] ?? '',
            $_SERVER['HTTP_ACCEPT_LANGUAGE'] ?? '',
            $_SERVER['HTTP_ACCEPT_ENCODING'] ?? '',
        ];

        return hash('sha256', implode('|', $components));
    }

    /**
     * Check if IP is trusted
     */
    public function isTrustedIp(string $ip): bool
    {
        if (empty($this->trustedIps)) {
            return false;
        }

        return in_array($ip, $this->trustedIps);
    }

    /**
     * Destroy session securely
     */
    public function destroySession(): void
    {
        $_SESSION = [];

        if (ini_get('session.use_cookies')) {
            $params = session_get_cookie_params();
            setcookie(
                session_name(),
                '',
                time() - 42000,
                $params['path'],
                $params['domain'],
                $params['secure'],
                $params['httponly']
            );
        }

        session_destroy();
    }

    /**
     * Track concurrent sessions
     */
    public function checkConcurrentSession(int $userId): bool
    {
        $currentSessionId = session_id();

        // Find existing active sessions for this user
        $existingSession = Capsule::table('mod_active_sessions')
            ->where('user_id', $userId)
            ->where('session_id', '!=', $currentSessionId)
            ->where('last_activity', '>', date('Y-m-d H- i:s', time() - $this->sessionLifetime))
            ->first();

        if ($existingSession) {
            // Option 1: Deny new session
            // Option 2: Invalidate old session
            $this->invalidateSession($existingSession->session_id);
            return true;
        }

        return false;
    }
}

// Initialize session security
add_hook('SessionStarted', 1, function() {
    $sessionSecurity = new SessionSecurity();
    $sessionSecurity->initSecureSession();
});
```

---

## Best Practices

1. **Use HTTPS everywhere** - Encrypt all authentication traffic
2. **Enforce strong passwords** - Implement comprehensive password policies
3. **Enable 2FA** - Require two-factor authentication for admin accounts
4. **Implement SSO carefully** - Validate all SSO tokens properly
5. **Monitor login attempts** - Track and block suspicious activity
6. **Use secure session handling** - Set secure cookie flags
7. **Limit session lifetime** - Auto-expire inactive sessions
8. **Log all auth events** - Maintain audit trail of logins
9. **Provide backup authentication** - Have fallback options for 2FA failures
10. **Regular security audits** - Review authentication logs regularly

---

## Verification Checklist

- [ ] Admin authentication uses HTTPS
- [ ] Password policy enforced on registration
- [ ] Password policy enforced on admin changes
- [ ] 2FA setup flow complete
- [ ] 2FA verification working correctly
- [ ] Backup codes generated and stored securely
- [ ] SSO integration functional
- [ ] Session hijacking detection active
- [ ] Concurrent session management working
- [ ] Failed login attempt blocking active
