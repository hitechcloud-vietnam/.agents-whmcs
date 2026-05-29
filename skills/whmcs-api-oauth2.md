# WHMCS API OAuth2 Implementation

## Skill Description
Implement OAuth2 authentication for WHMCS modules including authorization code flow, refresh token handling, scope management, and secure token storage.

## Prerequisites
- WHMCS 8.0+ installation
- PHP 7.4+ with OpenSSL extension
- Database access for token storage
- SSL/HTTPS for production environment

## Step-by-Step Implementation

### 1. OAuth2 Server Class
```php
<?php
// includes/oauth2/OAuth2Server.php

namespace WHMCS\Module\YourModule\OAuth2;

class OAuth2Server
{
    private array $config;
    private array $supportedScopes = ['read', 'write', 'admin'];
    private array $supportedGrantTypes = ['authorization_code', 'refresh_token', 'client_credentials'];

    public function __construct(array $config = [])
    {
        $this->config = array_merge([
            'client_id' => '',
            'client_secret' => '',
            'redirect_uri' => '',
            'authorization_endpoint' => '/oauth/authorize',
            'token_endpoint' => '/oauth/token',
            'token_ttl' => 3600,
            'refresh_token_ttl' => 86400 * 30 // 30 days
        ], $config);
    }

    public function authorize(
        string $clientId,
        string $redirectUri,
        string $responseType,
        string $state,
        string $scope = ''
    ): array {
        // Validate client
        $client = $this->getClient($clientId);

        if (!$client) {
            return [
                'error' => 'invalid_client',
                'error_description' => 'Client application not found'
            ];
        }

        if ($client['redirect_uri'] !== $redirectUri) {
            return [
                'error' => 'invalid_request',
                'error_description' => 'Redirect URI mismatch'
            ];
        }

        // Validate requested scopes
        $requestedScopes = explode(' ', $scope);
        $allowedScopes = $this->validateScopes($requestedScopes);

        if (empty($allowedScopes)) {
            return [
                'error' => 'invalid_scope',
                'error_description' => 'Requested scopes are not allowed'
            ];
        }

        // Store authorization request
        $authCode = $this->createAuthorizationCode(
            $client['id'],
            $redirectUri,
            $allowedScopes
        );

        // Build redirect URL
        $params = [
            'code' => $authCode['code'],
            'state' => $state
        ];

        return [
            'redirect' => $redirectUri . '?' . http_build_query($params)
        ];
    }

    public function token(string $grantType, array $params): array
    {
        if (!in_array($grantType, $this->supportedGrantTypes)) {
            return [
                'error' => 'unsupported_grant_type',
                'error_description' => 'Grant type not supported'
            ];
        }

        return match ($grantType) {
            'authorization_code' => $this->handleAuthorizationCode($params),
            'refresh_token' => $this->handleRefreshToken($params),
            'client_credentials' => $this->handleClientCredentials($params),
            default => ['error' => 'unsupported_grant_type']
        };
    }

    private function handleAuthorizationCode(array $params): array
    {
        // Validate code
        $code = $params['code'] ?? '';
        $codeVerifier = $params['code_verifier'] ?? '';

        $authCode = $this->getAuthorizationCode($code);

        if (!$authCode) {
            return [
                'error' => 'invalid_grant',
                'error_description' => 'Invalid or expired authorization code'
            ];
        }

        // Verify code verifier if PKCE was used
        if ($authCode['code_challenge']) {
            if (!$this->verifyCodeChallenge($codeVerifier, $authCode['code_challenge'])) {
                return [
                    'error' => 'invalid_grant',
                    'error_description' => 'Code verifier does not match'
                ];
            }
        }

        // Delete the authorization code (single use)
        $this->deleteAuthorizationCode($code);

        // Generate tokens
        return $this->generateTokens($authCode['client_id'], $authCode['user_id'], $authCode['scopes']);
    }

    private function handleRefreshToken(array $params): array
    {
        $refreshToken = $params['refresh_token'] ?? '';

        $tokenData = $this->getRefreshToken($refreshToken);

        if (!$tokenData) {
            return [
                'error' => 'invalid_grant',
                'error_description' => 'Invalid or expired refresh token'
            ];
        }

        // Check if token is expired
        if (strtotime($tokenData['expires_at']) < time()) {
            $this->deleteRefreshToken($refreshToken);
            return [
                'error' => 'invalid_grant',
                'error_description' => 'Refresh token has expired'
            ];
        }

        // Delete old refresh token
        $this->deleteRefreshToken($refreshToken);

        // Generate new tokens
        return $this->generateTokens($tokenData['client_id'], $tokenData['user_id'], $tokenData['scopes']);
    }

    private function handleClientCredentials(array $params): array
    {
        $clientId = $params['client_id'] ?? '';
        $clientSecret = $params['client_secret'] ?? '';

        $client = $this->validateClientCredentials($clientId, $clientSecret);

        if (!$client) {
            return [
                'error' => 'invalid_client',
                'error_description' => 'Invalid client credentials'
            ];
        }

        return $this->generateTokens($client['id'], null, $client['scopes']);
    }

    private function generateTokens(
        int $clientId,
        ?int $userId,
        array $scopes
    ): array {
        $accessToken = $this->generateSecureToken(64);
        $refreshToken = $this->generateSecureToken(64);

        // Store access token
        $this->storeAccessToken($accessToken, $clientId, $userId, $scopes);

        // Store refresh token
        $this->storeRefreshToken($refreshToken, $clientId, $userId, $scopes);

        return [
            'access_token' => $accessToken,
            'token_type' => 'Bearer',
            'expires_in' => $this->config['token_ttl'],
            'refresh_token' => $refreshToken,
            'scope' => implode(' ', $scopes)
        ];
    }

    public function validateAccessToken(string $token): ?array
    {
        global $db;

        $result = $db->select(
            "SELECT * FROM mod_yourmodule_oauth_access_tokens
             WHERE token = ? AND expires_at > NOW()",
            [$token]
        );

        if (empty($result)) {
            return null;
        }

        $tokenData = $result[0];

        return [
            'client_id' => $tokenData['client_id'],
            'user_id' => $tokenData['user_id'],
            'scopes' => explode(' ', $tokenData['scopes']),
            'expires_at' => $tokenData['expires_at']
        ];
    }

    public function hasScope(array $tokenData, string $requiredScope): bool
    {
        return in_array($requiredScope, $tokenData['scopes']);
    }

    private function generateSecureToken(int $length): string
    {
        return bin2hex(random_bytes($length / 2));
    }

    private function createAuthorizationCode(
        int $clientId,
        string $redirectUri,
        array $scopes
    ): array {
        global $db;

        $code = $this->generateSecureToken(64);
        $expiresAt = date('Y-m-d H:i:s', time() + 600); // 10 minutes

        $db->insert('mod_yourmodule_oauth_authorization_codes', [
            'code' => $code,
            'client_id' => $clientId,
            'redirect_uri' => $redirectUri,
            'scopes' => implode(' ', $scopes),
            'expires_at' => $expiresAt,
            'created_at' => date('Y-m-d H:i:s')
        ]);

        return ['code' => $code];
    }

    private function getAuthorizationCode(string $code): ?array
    {
        global $db;

        $result = $db->select(
            "SELECT * FROM mod_yourmodule_oauth_authorization_codes
             WHERE code = ? AND expires_at > NOW()",
            [$code]
        );

        return $result[0] ?? null;
    }

    private function deleteAuthorizationCode(string $code): void
    {
        global $db;
        $db->delete('mod_yourmodule_oauth_authorization_codes', 'code = ?', [$code]);
    }

    private function storeAccessToken(
        string $token,
        int $clientId,
        ?int $userId,
        array $scopes
    ): void {
        global $db;

        $db->insert('mod_yourmodule_oauth_access_tokens', [
            'token' => $token,
            'client_id' => $clientId,
            'user_id' => $userId,
            'scopes' => implode(' ', $scopes),
            'expires_at' => date('Y-m-d H:i:s', time() + $this->config['token_ttl']),
            'created_at' => date('Y-m-d H:i:s')
        ]);
    }

    private function storeRefreshToken(
        string $token,
        int $clientId,
        ?int $userId,
        array $scopes
    ): void {
        global $db;

        $db->insert('mod_yourmodule_oauth_refresh_tokens', [
            'token' => password_hash($token, PASSWORD_ARGON2ID),
            'client_id' => $clientId,
            'user_id' => $userId,
            'scopes' => implode(' ', $scopes),
            'expires_at' => date('Y-m-d H:i:s', time() + $this->config['refresh_token_ttl']),
            'created_at' => date('Y-m-d H:i:s')
        ]);
    }

    private function getRefreshToken(string $token): ?array
    {
        global $db;

        // We need to check all refresh tokens as they are hashed
        $results = $db->select(
            "SELECT * FROM mod_yourmodule_oauth_refresh_tokens WHERE expires_at > NOW()"
        );

        foreach ($results as $result) {
            if (password_verify($token, $result['token'])) {
                return $result;
            }
        }

        return null;
    }

    private function deleteRefreshToken(string $token): void
    {
        global $db;
        $db->query("DELETE FROM mod_yourmodule_oauth_refresh_tokens");
    }

    private function validateScopes(array $scopes): array
    {
        return array_intersect($scopes, $this->supportedScopes);
    }

    private function getClient(string $clientId): ?array
    {
        global $db;

        $result = $db->select(
            "SELECT * FROM mod_yourmodule_oauth_clients WHERE client_id = ? AND is_active = 1",
            [$clientId]
        );

        return $result[0] ?? null;
    }

    private function validateClientCredentials(string $clientId, string $clientSecret): ?array
    {
        $client = $this->getClient($clientId);

        if (!$client) {
            return null;
        }

        if (!password_verify($clientSecret, $client['client_secret'])) {
            return null;
        }

        return $client;
    }

    private function verifyCodeChallenge(string $verifier, string $challenge): bool
    {
        $method = substr($challenge, 0, 7);
        $hash = substr($challenge, 9);

        $expectedHash = match ($method) {
            'S256:' => base64_encode(hash('sha256', $verifier, true)),
            'plain:' => $verifier,
            default => null
        };

        return $expectedHash !== null && hash_equals($hash, $expectedHash);
    }
}
```

### 2. OAuth2 Middleware
```php
<?php
// includes/oauth2/OAuth2Middleware.php

namespace WHMCS\Module\YourModule\OAuth2;

class OAuth2Middleware
{
    private OAuth2Server $server;
    private array $requiredScopes = [];

    public function __construct(?OAuth2Server $server = null)
    {
        $this->server = $server ?? new OAuth2Server();
    }

    public function setRequiredScopes(array $scopes): self
    {
        $this->requiredScopes = $scopes;
        return $this;
    }

    public function handle(): ?array
    {
        $token = $this->extractToken();

        if (!$token) {
            return null;
        }

        $tokenData = $this->server->validateAccessToken($token);

        if (!$tokenData) {
            return null;
        }

        // Check scopes
        foreach ($this->requiredScopes as $scope) {
            if (!$this->server->hasScope($tokenData, $scope)) {
                return null;
            }
        }

        return $tokenData;
    }

    public function requireScopes(array $scopes): self
    {
        return $this->setRequiredScopes($scopes);
    }

    private function extractToken(): ?string
    {
        // Try Authorization header
        $header = $_SERVER['HTTP_AUTHORIZATION'] ?? '';

        if (preg_match('/^Bearer\s+(.+)$/i', $header, $matches)) {
            return $matches[1];
        }

        // Try query parameter
        return $_GET['access_token'] ?? null;
    }
}
```

### 3. Database Tables
```sql
-- OAuth2 Clients
CREATE TABLE IF NOT EXISTS mod_yourmodule_oauth_clients (
    id INT AUTO_INCREMENT PRIMARY KEY,
    client_id VARCHAR(255) UNIQUE NOT NULL,
    client_secret VARCHAR(255) NOT NULL,
    name VARCHAR(255) NOT NULL,
    redirect_uri VARCHAR(500) NOT NULL,
    scopes VARCHAR(255) DEFAULT 'read',
    is_active TINYINT(1) DEFAULT 1,
    created_at DATETIME NOT NULL,
    updated_at DATETIME,
    INDEX idx_client_id (client_id)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;

-- OAuth2 Authorization Codes
CREATE TABLE IF NOT EXISTS mod_yourmodule_oauth_authorization_codes (
    id INT AUTO_INCREMENT PRIMARY KEY,
    code VARCHAR(255) UNIQUE NOT NULL,
    client_id INT NOT NULL,
    redirect_uri VARCHAR(500) NOT NULL,
    user_id INT,
    scopes VARCHAR(255),
    code_challenge VARCHAR(255),
    expires_at DATETIME NOT NULL,
    created_at DATETIME NOT NULL,
    INDEX idx_code (code),
    INDEX idx_expires (expires_at)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;

-- OAuth2 Access Tokens
CREATE TABLE IF NOT EXISTS mod_yourmodule_oauth_access_tokens (
    id INT AUTO_INCREMENT PRIMARY KEY,
    token VARCHAR(255) UNIQUE NOT NULL,
    client_id INT NOT NULL,
    user_id INT,
    scopes VARCHAR(255),
    expires_at DATETIME NOT NULL,
    created_at DATETIME NOT NULL,
    INDEX idx_token (token),
    INDEX idx_expires (expires_at)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;

-- OAuth2 Refresh Tokens
CREATE TABLE IF NOT EXISTS mod_yourmodule_oauth_refresh_tokens (
    id INT AUTO_INCREMENT PRIMARY KEY,
    token VARCHAR(255) NOT NULL,
    client_id INT NOT NULL,
    user_id INT,
    scopes VARCHAR(255),
    expires_at DATETIME NOT NULL,
    created_at DATETIME NOT NULL,
    INDEX idx_expires (expires_at)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
```

### 4. PKCE Support
```php
<?php
// includes/oauth2/PKCE.php

namespace WHMCS\Module\YourModule\OAuth2;

class PKCE
{
    public static function generateCodeVerifier(): string
    {
        return bin2hex(random_bytes(32));
    }

    public static function generateCodeChallenge(string $verifier, string $method = 'S256'): string
    {
        return match ($method) {
            'S256' => 'S256:' . base64_encode(hash('sha256', $verifier, true)),
            'plain' => 'plain:' . $verifier,
            default => throw new \InvalidArgumentException('Invalid method')
        };
    }

    public static function verifyChallenge(string $verifier, string $challenge): bool
    {
        $expectedChallenge = self::generateCodeChallenge($verifier);
        return hash_equals($expectedChallenge, $challenge);
    }

    public static function generateState(): string
    {
        return bin2hex(random_bytes(16));
    }
}
```

## Common Pitfalls and Solutions

| Pitfall | Solution |
|---------|----------|
| Token leakage in logs | Never log tokens or include in URLs |
| Token expiration issues | Implement refresh token rotation |
| Scope escalation | Always validate requested scopes against allowed scopes |
| PKCE bypass | Enforce PKCE for public clients |
| Token replay | Implement token rotation on refresh |

## Security Considerations

1. **Use HTTPS exclusively** - Never transmit tokens over HTTP
2. **Hash refresh tokens** - Never store plain text refresh tokens
3. **Implement PKCE for public clients** - Required for SPA and mobile apps
4. **Set short access token TTL** - Use refresh tokens for long-lived sessions
5. **Implement token revocation** - Allow users to revoke access
6. **Log token operations** - Track token issuance and usage

## Testing Checklist

- [ ] Test authorization code flow
- [ ] Test refresh token rotation
- [ ] Test client credentials flow
- [ ] Test scope validation
- [ ] Test PKCE verification
- [ ] Test token expiration
- [ ] Test token revocation
- [ ] Test invalid token handling
- [ ] Test redirect URI validation
- [ ] Test state parameter validation

## Reference Links

- [OAuth 2.0 RFC 6749](https://datatracker.ietf.org/doc/html/rfc6749)
- [OAuth 2.0 Security Best Current Practice](https://datatracker.ietf.org/doc/html/draft-ietf-oauth-security-topics)
- [PKCE RFC 7636](https://datatracker.ietf.org/doc/html/rfc7636)
- [OAuth 2.0 for Browser-Based Apps](https://datatracker.ietf.org/doc/html/draft-ietf-oauth-browser-based-apps)
