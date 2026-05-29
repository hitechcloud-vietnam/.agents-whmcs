# WHMCS Authentication Integration

Complete guide for implementing authentication with WHMCS.

## Overview

Integrate custom authentication systems with WHMCS client accounts.

## SSO Integration

### Single Sign-On Implementation

```php
<?php
/**
 * Custom SSO provider
 */
class WHMCSSSOProvider
{
    private string $whmcsUrl;
    private string $apiKey;
    
    public function __construct(string $whmcsUrl, string $apiKey)
    {
        $this->whmcsUrl = rtrim($whmcsUrl, '/');
        $this->apiKey = $apiKey;
    }
    
    /**
     * Generate SSO token for client
     */
    public function generateSSOToken(int $clientId): string
    {
        $expiry = time() + 3600; // 1 hour
        $data = [
            'client_id' => $clientId,
            'expiry' => $expiry,
            'timestamp' => time(),
        ];
        
        $token = base64_encode(json_encode($data));
        $signature = $this->sign($token);
        
        return $token . '.' . $signature;
    }
    
    /**
     * Sign token data
     */
    private function sign(string $data): string
    {
        $secret = $this->getSigningSecret();
        return hash_hmac('sha256', $data, $secret);
    }
    
    /**
     * Get signing secret from WHMCS
     */
    private function getSigningSecret(): string
    {
        return Capsule::table('tblconfiguration')
            ->where('setting', 'SSOSecret')
            ->first()->value ?? '';
    }
    
    /**
     * Build SSO redirect URL
     */
    public function buildSSOUrl(string $token, string $redirectUrl = ''): string
    {
        $params = [
            'sso' => $token,
            'redirect' => base64_encode($redirectUrl),
        ];
        
        return $this->whmcsUrl . '/sso.php?' . http_build_query($params);
    }
}
```

### SSO Callback Handler

```php
<?php
/**
 * Handle SSO authentication
 */
function handleSSOAuthentication(): void
{
    $token = $_GET['sso'] ?? '';
    $signature = $_GET['sig'] ?? '';
    
    if (!$token || !$signature) {
        throw new Exception('Missing SSO parameters');
    }
    
    // Verify signature
    $expectedSig = hash_hmac('sha256', $token, getSSOSecret());
    if (!hash_equals($expectedSig, $signature)) {
        throw new Exception('Invalid SSO signature');
    }
    
    // Decode token
    $data = json_decode(base64_decode($token), true);
    
    if (!$data || $data['expiry'] < time()) {
        throw new Exception('Invalid or expired SSO token');
    }
    
    // Get or create session for client
    $clientId = $data['client_id'];
    $_SESSION['uid'] = $clientId;
    $_SESSION['upw'] = ''; // No password check needed
    
    // Redirect to intended page
    $redirectUrl = base64_decode($_GET['redirect'] ?? '');
    header('Location: ' . ($redirectUrl ?: $whmcsUrl . '/clientarea.php'));
    exit;
}
```

### SSO Link Generation

```php
<?php
/**
 * Generate SSO link for WHMCS login
 */
function generateWHMCSLoginLink(int $clientId, string $returnUrl = ''): string
{
    $ssoProvider = new WHMCSSSOProvider(WHMCS_URL, API_KEY);
    $token = $ssoProvider->generateSSOToken($clientId);
    
    return $ssoProvider->buildSSOUrl($token, $returnUrl);
}
```

## External Authentication

### LDAP Integration

```php
<?php
/**
 * LDAP authentication provider
 */
class LDAPAuthProvider
{
    private string $server;
    private int $port;
    private string $baseDn;
    
    public function __construct(array $config)
    {
        $this->server = $config['server'];
        $this->port = $config['port'] ?? 389;
        $this->baseDn = $config['base_dn'];
    }
    
    /**
     * Authenticate user against LDAP
     */
    public function authenticate(string $username, string $password): ?array
    {
        $connection = ldap_connect($this->server, $this->port);
        
        ldap_set_option($connection, LDAP_OPT_PROTOCOL_VERSION, 3);
        ldap_set_option($connection, LDAP_OPT_REFERRALS, 0);
        
        // Bind with user credentials
        $userDn = "uid={$username},{$this->baseDn}";
        
        if (!@ldap_bind($connection, $userDn, $password)) {
            return null;
        }
        
        // Get user attributes
        $search = ldap_search($connection, $this->baseDn, "uid={$username}");
        $entries = ldap_get_entries($connection, $search);
        
        if ($entries['count'] === 0) {
            return null;
        }
        
        $entry = $entries[0];
        
        return [
            'username' => $username,
            'email' => $entry['mail'][0] ?? '',
            'firstname' => $entry['givenname'][0] ?? '',
            'lastname' => $entry['sn'][0] ?? '',
            'company' => $entry['o'][0] ?? '',
        ];
    }
}
```

### External Auth Hook

```php
<?php
/**
 * Custom authentication hook
 */
add_hook('UserAuthPreValidation', 1, function($vars) {
    $username = $vars['username'];
    $password = $vars['password'];
    
    // Try external authentication first
    $ldapAuth = new LDAPAuthProvider([
        'server' => 'ldap.example.com',
        'base_dn' => 'ou=users,dc=example,dc=com',
    ]);
    
    $user = $ldapAuth->authenticate($username, $password);
    
    if ($user) {
        // Check if user exists in WHMCS
        $existingClient = Capsule::table('tblclients')
            ->where('email', $user['email'])
            ->first();
        
        if ($existingClient) {
            return ['auth_user_id' => $existingClient->id];
        }
        
        // Auto-create client
        return createClientFromExternalAuth($user);
    }
    
    // Return null to fall back to standard WHMCS auth
    return null;
});

/**
 * Create client from external auth
 */
function createClientFromExternalAuth(array $userData): array
{
    $result = localAPI('AddClient', [
        'firstname' => $userData['firstname'],
        'lastname' => $userData['lastname'],
        'email' => $userData['email'],
        'companyname' => $userData['company'] ?? '',
        'password2' => bin2hex(random_bytes(16)), // Random password
    ]);
    
    if ($result['result'] === 'success') {
        return ['auth_user_id' => $result['clientid']];
    }
    
    return null;
}
```

## Two-Factor Authentication

### Custom 2FA Provider

```php
<?php
/**
 * Custom 2FA implementation
 */
class CustomTwoFactorProvider
{
    /**
     * Generate secret
     */
    public function generateSecret(): string
    {
        return base64_encode(random_bytes(20));
    }
    
    /**
     * Generate QR code URL
     */
    public function getQrCodeUrl(string $secret, string $email): string
    {
        $otpauthUrl = 'otpauth://totp/' . rawurlencode($email) . '?secret=' . $secret;
        return 'https://api.qrserver.com/v1/create-qr-code/?data=' . rawurlencode($otpauthUrl);
    }
    
    /**
     * Verify TOTP code
     */
    public function verify(string $secret, string $code): bool
    {
        $timeSlice = floor(time() / 30);
        
        for ($i = -1; $i <= 1; $i++) {
            $checkCode = $this->generateTOTP($secret, $timeSlice + $i);
            if (hash_equals($checkCode, $code)) {
                return true;
            }
        }
        
        return false;
    }
    
    /**
     * Generate TOTP code
     */
    private function generateTOTP(string $secret, int $timeSlice): string
    {
        $binary = pack('H*', hash_hmac('sha1', pack('N*', 0) . pack('N*', $timeSlice), base64_decode($secret), true));
        
        $hash = '';
        for ($i = 0; $i < strlen($binary); $i++) {
            $hash .= chr(ord($binary[$i]) & 0xFA);
        }
        
        $offset = ord($hash[strlen($hash) - 1]) & 0x0F;
        $code = (
            ((ord($hash[$offset]) & 0x7F) << 24) |
            ((ord($hash[$offset + 1]) & 0xFF) << 16) |
            ((ord($hash[$offset + 2]) & 0xFF) << 8) |
            (ord($hash[$offset + 3]) & 0xFF)
        ) % 1000000;
        
        return str_pad((string)$code, 6, '0', STR_PAD_LEFT);
    }
}
```

### Enable 2FA for User

```php
<?php
/**
 * Enable 2FA for client
 */
function enableClientTwoFactor(int $clientId): array
{
    $provider = new CustomTwoFactorProvider();
    $secret = $provider->generateSecret();
    
    // Store secret
    Capsule::table('tblclients')
        ->where('id', $clientId)
        ->update([
            'authdata' => json_encode([
                '2fa_secret' => $secret,
                '2fa_enabled' => true,
                '2fa_enabled_at' => date('Y-m-d H:i:s'),
            ]),
        ]);
    
    // Get client email for QR code
    $client = Capsule::table('tblclients')
        ->where('id', $clientId)
        ->first();
    
    return [
        'secret' => $secret,
        'qr_code_url' => $provider->getQrCodeUrl($secret, $client->email),
    ];
}
```

## Session Management

### Custom Session Handler

```php
<?php
/**
 * Database session handler
 */
class DatabaseSessionHandler implements SessionHandlerInterface
{
    private PDO $pdo;
    
    public function __construct()
    {
        $this->pdo = Capsule::connection()->getPdo();
    }
    
    public function open(string $path, string $name): bool
    {
        return true;
    }
    
    public function close(): bool
    {
        return true;
    }
    
    public function read(string $id): string
    {
        $stmt = $this->pdo->prepare(
            'SELECT data FROM sessions WHERE id = ? AND expires > ?'
        );
        $stmt->execute([$id, time()]);
        $result = $stmt->fetch();
        
        return $result ? $result['data'] : '';
    }
    
    public function write(string $id, string $data): bool
    {
        $expires = time() + ini_get('session.gc_maxlifetime');
        
        $stmt = $this->pdo->prepare(
            'INSERT INTO sessions (id, data, expires) VALUES (?, ?, ?)
             ON DUPLICATE KEY UPDATE data = VALUES(data), expires = VALUES(expires)'
        );
        
        return $stmt->execute([$id, $data, $expires]);
    }
    
    public function destroy(string $id): bool
    {
        $stmt = $this->pdo->prepare('DELETE FROM sessions WHERE id = ?');
        return $stmt->execute([$id]);
    }
    
    public function gc(int $maxlifetime): bool
    {
        $stmt = $this->pdo->prepare('DELETE FROM sessions WHERE expires < ?');
        return $stmt->execute([time()]);
    }
}
```

### Initialize Custom Sessions

```php
<?php
/**
 * Initialize database session handler
 */
function initializeDatabaseSessions(): void
{
    $handler = new DatabaseSessionHandler();
    session_set_save_handler($handler, true);
}
```

## Best Practices

1. **Use HTTPS** - Always encrypt authentication traffic
2. **Hash passwords** - Never store plain text passwords
3. **Use secure tokens** - Sign and verify all tokens
4. **Implement 2FA** - Add extra security layer
5. **Log authentication** - Track login attempts
6. **Session timeout** - Implement appropriate timeouts

## Related Documentation

- [whmcs-integration-oauth.md](whmcs-integration-oauth.md)
- [whmcs-integration-api.md](whmcs-integration-api.md)
