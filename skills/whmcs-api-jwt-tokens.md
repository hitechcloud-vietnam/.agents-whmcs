# WHMCS API JWT Tokens

## Skill Description
Implement JSON Web Token (JWT) authentication for WHMCS modules including token generation, validation, refresh handling, and secure storage.

## Prerequisites
- WHMCS 7.0+ installation
- PHP 7.4+ with OpenSSL extension
- PHP-JWT library or custom implementation
- Understanding of JWT structure

## Step-by-Step Implementation

### 1. JWT Token Handler
```php
<?php
// includes/jwt/JwtHandler.php

namespace WHMCS\Module\YourModule\JWT;

class JwtHandler
{
    private string $secretKey;
    private string $algorithm;
    private int $accessTokenTtl;
    private int $refreshTokenTtl;

    public function __construct(array $config = [])
    {
        $this->secretKey = $config['secret_key'] ?? $this->getSecretFromConfig();
        $this->algorithm = $config['algorithm'] ?? 'HS256';
        $this->accessTokenTtl = $config['access_token_ttl'] ?? 3600;
        $this->refreshTokenTtl = $config['refresh_token_ttl'] ?? 86400 * 30;
    }

    private function getSecretFromConfig(): string
    {
        $settings = getWHMCSModuleConfig('yourmodule');
        return $settings['jwt_secret'] ?? '';
    }

    public function createAccessToken(array $payload): string
    {
        $payload['iat'] = time();
        $payload['exp'] = time() + $this->accessTokenTtl;
        $payload['type'] = 'access';

        return $this->encode($payload);
    }

    public function createRefreshToken(array $payload): string
    {
        $payload['iat'] = time();
        $payload['exp'] = time() + $this->refreshTokenTtl;
        $payload['type'] = 'refresh';
        $payload['jti'] = $this->generateJti();

        return $this->encode($payload);
    }

    public function createTokenPair(int $userId, array $scopes = [], array $additionalClaims = []): array
    {
        $accessPayload = array_merge([
            'sub' => $userId,
            'scopes' => $scopes,
            'token_type' => 'access'
        ], $additionalClaims);

        $refreshPayload = [
            'sub' => $userId,
            'scopes' => $scopes,
            'token_type' => 'refresh'
        ];

        return [
            'access_token' => $this->createAccessToken($accessPayload),
            'refresh_token' => $this->createRefreshToken($refreshPayload),
            'token_type' => 'Bearer',
            'expires_in' => $this->accessTokenTtl
        ];
    }

    public function validateToken(string $token): ?array
    {
        try {
            $payload = $this->decode($token);

            // Verify expiration
            if (isset($payload['exp']) && $payload['exp'] < time()) {
                return null;
            }

            // Verify not before
            if (isset($payload['nbf']) && $payload['nbf'] > time()) {
                return null;
            }

            return $payload;

        } catch (\Exception $e) {
            logActivity('JWT validation failed: ' . $e->getMessage());
            return null;
        }
    }

    public function validateAccessToken(string $token): ?array
    {
        $payload = $this->validateToken($token);

        if (!$payload || ($payload['type'] ?? '') !== 'access') {
            return null;
        }

        return $payload;
    }

    public function validateRefreshToken(string $token): ?array
    {
        $payload = $this->validateToken($token);

        if (!$payload || ($payload['type'] ?? '') !== 'refresh') {
            return null;
        }

        // Check if refresh token has been revoked
        if (isset($payload['jti']) && $this->isRefreshTokenRevoked($payload['jti'])) {
            return null;
        }

        return $payload;
    }

    public function refreshTokens(string $refreshToken): ?array
    {
        $payload = $this->validateRefreshToken($refreshToken);

        if (!$payload) {
            return null;
        }

        // Revoke old refresh token
        if (isset($payload['jti'])) {
            $this->revokeRefreshToken($payload['jti']);
        }

        // Create new token pair
        return $this->createTokenPair(
            $payload['sub'],
            $payload['scopes'] ?? []
        );
    }

    public function revokeRefreshToken(string $jti): void
    {
        global $db;

        $db->insert('mod_yourmodule_revoked_tokens', [
            'jti' => $jti,
            'revoked_at' => date('Y-m-d H:i:s')
        ]);
    }

    public function isRefreshTokenRevoked(string $jti): bool
    {
        global $db;

        $result = $db->select(
            "SELECT id FROM mod_yourmodule_revoked_tokens WHERE jti = ?",
            [$jti]
        );

        return !empty($result);
    }

    private function encode(array $payload): string
    {
        $header = $this->getHeader();

        $headerEncoded = $this->base64UrlEncode(json_encode($header));
        $payloadEncoded = $this->base64UrlEncode(json_encode($payload));

        $signature = hash_hmac(
            'sha256',
            $headerEncoded . '.' . $payloadEncoded,
            $this->secretKey,
            true
        );
        $signatureEncoded = $this->base64UrlEncode($signature);

        return $headerEncoded . '.' . $payloadEncoded . '.' . $signatureEncoded;
    }

    private function decode(string $token): array
    {
        $parts = explode('.', $token);

        if (count($parts) !== 3) {
            throw new \InvalidArgumentException('Invalid token structure');
        }

        [$headerEncoded, $payloadEncoded, $signatureEncoded] = $parts;

        // Verify signature
        $expectedSignature = hash_hmac(
            'sha256',
            $headerEncoded . '.' . $payloadEncoded,
            $this->secretKey,
            true
        );

        if (!hash_equals($this->base64UrlEncode($expectedSignature), $signatureEncoded)) {
            throw new \InvalidArgumentException('Invalid signature');
        }

        // Decode header
        $header = json_decode($this->base64UrlDecode($headerEncoded), true);

        if (!$header || !isset($header['alg'])) {
            throw new \InvalidArgumentException('Invalid token header');
        }

        // Verify algorithm
        if ($header['alg'] !== $this->algorithm) {
            throw new \InvalidArgumentException('Algorithm mismatch');
        }

        // Decode payload
        $payload = json_decode($this->base64UrlDecode($payloadEncoded), true);

        if (!$payload) {
            throw new \InvalidArgumentException('Invalid token payload');
        }

        return $payload;
    }

    private function getHeader(): array
    {
        return [
            'typ' => 'JWT',
            'alg' => $this->algorithm
        ];
    }

    private function base64UrlEncode(string $data): string
    {
        return rtrim(strtr(base64_encode($data), '+/', '-_'), '=');
    }

    private function base64UrlDecode(string $data): string
    {
        return base64_decode(strtr($data, '-_', '+/'));
    }

    private function generateJti(): string
    {
        return bin2hex(random_bytes(16));
    }

    public function getClaims(string $token): ?array
    {
        $payload = $this->validateToken($token);
        return $payload;
    }

    public function getSubject(string $token): ?string
    {
        $payload = $this->validateToken($token);
        return $payload['sub'] ?? null;
    }
}
```

### 2. JWT Authentication Middleware
```php
<?php
// includes/jwt/JwtMiddleware.php

namespace WHMCS\Module\YourModule\JWT;

use WHMCS\Module\YourModule\Api\ApiResponse;

class JwtMiddleware
{
    private JwtHandler $jwt;
    private array $requiredScopes = [];

    public function __construct(?JwtHandler $jwt = null)
    {
        $this->jwt = $jwt ?? new JwtHandler();
    }

    public function requireScopes(array $scopes): self
    {
        $this->requiredScopes = $scopes;
        return $this;
    }

    public function handle(): ?array
    {
        $token = $this->extractToken();

        if (!$token) {
            $this->sendUnauthorizedResponse('No token provided');
            return null;
        }

        $payload = $this->jwt->validateAccessToken($token);

        if (!$payload) {
            $this->sendUnauthorizedResponse('Invalid or expired token');
            return null;
        }

        // Check scopes
        foreach ($this->requiredScopes as $scope) {
            if (!in_array($scope, $payload['scopes'] ?? [])) {
                $this->sendForbiddenResponse('Insufficient scope');
                return null;
            }
        }

        return $payload;
    }

    private function extractToken(): ?string
    {
        $header = $_SERVER['HTTP_AUTHORIZATION'] ?? '';

        if (preg_match('/^Bearer\s+(.+)$/i', $header, $matches)) {
            return $matches[1];
        }

        return $_GET['access_token'] ?? null;
    }

    private function sendUnauthorizedResponse(string $message): void
    {
        ApiResponse::unauthorized($message)->send();
        exit;
    }

    private function sendForbiddenResponse(string $message): void
    {
        ApiResponse::forbidden($message)->send();
        exit;
    }
}
```

### 3. Database Tables
```sql
-- Revoked tokens for refresh token blacklist
CREATE TABLE IF NOT EXISTS mod_yourmodule_revoked_tokens (
    id INT AUTO_INCREMENT PRIMARY KEY,
    jti VARCHAR(255) UNIQUE NOT NULL,
    revoked_at DATETIME NOT NULL,
    expires_at DATETIME,
    INDEX idx_jti (jti),
    INDEX idx_revoked_at (revoked_at)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;

-- Token usage audit log
CREATE TABLE IF NOT EXISTS mod_yourmodule_token_audit (
    id INT AUTO_INCREMENT PRIMARY KEY,
    user_id INT NOT NULL,
    token_jti VARCHAR(255),
    action ENUM('issued', 'refreshed', 'revoked') NOT NULL,
    ip_address VARCHAR(45),
    user_agent TEXT,
    created_at DATETIME NOT NULL,
    INDEX idx_user_id (user_id),
    INDEX idx_created_at (created_at)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
```

### 4. Token Storage Trait
```php
<?php
// includes/jwt/StoresJwtTokens.php

namespace WHMCS\Module\YourModule\Traits;

trait StoresJwtTokens
{
    public function storeTokenAudit(
        int $userId,
        ?string $jti,
        string $action,
        ?string $ipAddress = null,
        ?string $userAgent = null
    ): void {
        global $db;

        $db->insert('mod_yourmodule_token_audit', [
            'user_id' => $userId,
            'token_jti' => $jti,
            'action' => $action,
            'ip_address' => $ipAddress ?? ($_SERVER['REMOTE_ADDR'] ?? ''),
            'user_agent' => $userAgent ?? ($_SERVER['HTTP_USER_AGENT'] ?? ''),
            'created_at' => date('Y-m-d H:i:s')
        ]);
    }

    public function getUserTokenHistory(int $userId, int $limit = 50): array
    {
        global $db;

        return $db->select(
            "SELECT * FROM mod_yourmodule_token_audit
             WHERE user_id = ?
             ORDER BY created_at DESC
             LIMIT ?",
            [$userId, $limit]
        );
    }

    public function cleanupExpiredTokens(int $daysOld = 90): int
    {
        global $db;

        $cutoffDate = date('Y-m-d H:i:s', strtotime("-{$daysOld} days"));

        $db->delete(
            'mod_yourmodule_revoked_tokens',
            'revoked_at < ?',
            [$cutoffDate]
        );

        return $db->affectedRows();
    }

    public function revokeAllUserTokens(int $userId): int
    {
        global $db;

        // Get all active refresh tokens for user and revoke them
        // Note: For full implementation, you'd need to track active tokens

        $db->insert('mod_yourmodule_token_audit', [
            'user_id' => $userId,
            'token_jti' => null,
            'action' => 'revoked',
            'created_at' => date('Y-m-d H:i:s')
        ]);

        return 1;
    }
}
```

### 5. Usage Example
```php
<?php
// Example JWT-protected controller

namespace WHMCS\Module\YourModule\Api\Controllers;

use WHMCS\Module\YourModule\Api\ApiController;
use WHMCS\Module\YourModule\Api\ApiResponse;
use WHMCS\Module\YourModule\JWT\JwtHandler;
use WHMCS\Module\YourModule\JWT\JwtMiddleware;

class AuthController extends ApiController
{
    private JwtHandler $jwt;
    private JwtMiddleware $middleware;

    public function __construct()
    {
        parent::__construct();
        $this->jwt = new JwtHandler();
        $this->middleware = new JwtMiddleware($this->jwt);
    }

    public function login(): void
    {
        $email = $this->getRequiredParam('email');
        $password = $this->getRequiredParam('password');

        // Authenticate user
        $user = $this->authenticateUser($email, $password);

        if (!$user) {
            ApiResponse::error('Invalid credentials', 401)->send();
            return;
        }

        // Generate tokens
        $tokens = $this->jwt->createTokenPair(
            $user['id'],
            ['read', 'write'],
            ['email' => $user['email']]
        );

        // Store audit log
        $this->storeTokenAudit($user['id'], null, 'issued');

        ApiResponse::success($tokens)->send();
    }

    public function refresh(): void
    {
        $refreshToken = $this->getRequiredParam('refresh_token');

        $tokens = $this->jwt->refreshTokens($refreshToken);

        if (!$tokens) {
            ApiResponse::error('Invalid or expired refresh token', 401)->send();
            return;
        }

        ApiResponse::success($tokens)->send();
    }

    public function revoke(): void
    {
        $payload = $this->middleware->handle();

        if (!$payload) {
            return;
        }

        $jti = $payload['jti'] ?? null;

        if ($jti) {
            $this->jwt->revokeRefreshToken($jti);
            $this->storeTokenAudit($payload['sub'], $jti, 'revoked');
        }

        ApiResponse::success(['message' => 'Token revoked'])->send();
    }

    private function authenticateUser(string $email, string $password): ?array
    {
        global $db;

        $result = $db->select(
            "SELECT * FROM tblusers WHERE email = ?",
            [$email]
        );

        if (empty($result)) {
            return null;
        }

        $user = $result[0];

        if (!password_verify($password, $user['password'])) {
            return null;
        }

        return $user;
    }

    private function storeTokenAudit(int $userId, ?string $jti, string $action): void
    {
        global $db;

        $db->insert('mod_yourmodule_token_audit', [
            'user_id' => $userId,
            'token_jti' => $jti,
            'action' => $action,
            'ip_address' => $_SERVER['REMOTE_ADDR'] ?? '',
            'user_agent' => $_SERVER['HTTP_USER_AGENT'] ?? '',
            'created_at' => date('Y-m-d H:i:s')
        ]);
    }
}
```

## Common Pitfalls and Solutions

| Pitfall | Solution |
|---------|----------|
| Token stored in URL | Always use Authorization header, not query params |
| Weak secret key | Use cryptographically secure secret, min 256 bits |
| Tokens not validated properly | Always check signature, expiration, and claims |
| Refresh token not rotated | Always rotate refresh tokens on use |
| No token revocation | Implement token blacklist for immediate revocation |

## Security Considerations

1. **Use HTTPS only** - Never transmit tokens over HTTP
2. **Short token lifetime** - Keep access tokens short, use refresh tokens
3. **Rotate refresh tokens** - Issue new refresh token on each use
4. **Validate algorithm** - Prevent algorithm confusion attacks
5. **Store secrets securely** - Use environment variables or secure vault
6. **Implement token blacklist** - Allow immediate token revocation
7. **Audit token usage** - Log all token operations

## Testing Checklist

- [ ] Test token generation with all claims
- [ ] Test token validation with valid token
- [ ] Test token validation with expired token
- [ ] Test token validation with invalid signature
- [ ] Test token refresh with valid refresh token
- [ ] Test token refresh with revoked refresh token
- [ ] Test scope validation
- [ ] Test token revocation
- [ ] Test blacklist checking
- [ ] Test audit logging

## Reference Links

- [JWT RFC 7519](https://datatracker.ietf.org/doc/html/rfc7519)
- [JWS RFC 7515](https://datatracker.ietf.org/doc/html/rfc7515)
- [OAuth 2.0 Security Best Current Practice](https://datatracker.ietf.org/doc/html/draft-ietf-oauth-security-topics)
- [OWASP JWT Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/JSON_Web_Token_Cheat_Sheet.html)
