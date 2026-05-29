# WHMCS API Authentication

## Skill Description
Implement secure API authentication for WHMCS modules including API key-based authentication, OAuth 2.0, and integration with WHMCS native authentication system.

## Prerequisites
- WHMCS 8.0+ installation
- PHP 7.4+ with OpenSSL extension
- Understanding of HTTP authentication headers
- Access to WHMCS database

## Step-by-Step Implementation

### 1. API Key Authentication Class
```php
<?php
// includes/auth/ApiKeyAuthenticator.php

namespace WHMCS\Module\YourModule\Auth;

class ApiKeyAuthenticator
{
    private $db;
    private $validApiKeys = [];

    public function __construct()
    {
        global $db;
        $this->db = $db;
    }

    public function authenticate(string $apiKey): ?array
    {
        if (empty($apiKey)) {
            return null;
        }

        // Check cache first
        if (isset($this->validApiKeys[$apiKey])) {
            return $this->validApiKeys[$apiKey];
        }

        $result = $this->db->select(
            "SELECT * FROM mod_yourmodule_api_keys WHERE api_key = ? AND is_active = 1",
            [$apiKey]
        );

        if (empty($result)) {
            logActivity('Invalid API key attempted: ' . substr($apiKey, 0, 8) . '...');
            return null;
        }

        $keyData = $result[0];

        // Check expiration
        if ($keyData['expires_at'] && strtotime($keyData['expires_at']) < time()) {
            logActivity('Expired API key used: ' . substr($apiKey, 0, 8) . '...');
            return null;
        }

        // Cache the valid key
        $this->validApiKeys[$apiKey] = $keyData;

        return $keyData;
    }

    public function createApiKey(int $userId, string $description, int $daysValid = 365): array
    {
        $apiKey = $this->generateSecureApiKey();
        $hashedKey = password_hash($apiKey, PASSWORD_ARGON2ID);

        $expiresAt = date('Y-m-d H:i:s', strtotime("+{$daysValid} days"));

        $this->db->insert('mod_yourmodule_api_keys', [
            'user_id' => $userId,
            'api_key_hash' => $hashedKey,
            'api_key_prefix' => substr($apiKey, 0, 8),
            'description' => $description,
            'created_at' => date('Y-m-d H:i:s'),
            'expires_at' => $expiresAt,
            'is_active' => 1
        ]);

        return [
            'api_key' => $apiKey,
            'expires_at' => $expiresAt
        ];
    }

    private function generateSecureApiKey(): string
    {
        return bin2hex(random_bytes(32));
    }

    public function revokeApiKey(string $apiKey): bool
    {
        $result = $this->db->update('mod_yourmodule_api_keys', [
            'is_active' => 0,
            'revoked_at' => date('Y-m-d H:i:s')
        ], 'api_key_hash = ?', [password_hash($apiKey, PASSWORD_ARGON2ID)]);

        if (isset($this->validApiKeys[$apiKey])) {
            unset($this->validApiKeys[$apiKey]);
        }

        return $result > 0;
    }

    public function validateRequest(): ?array
    {
        $headers = getallheaders();

        // Try X-API-Key header first
        if (isset($headers['X-API-Key'])) {
            return $this->authenticate($headers['X-API-Key']);
        }

        // Try Authorization Bearer token
        if (isset($headers['Authorization'])) {
            if (preg_match('/^Bearer\s+(.+)$/i', $headers['Authorization'], $matches)) {
                return $this->authenticate($matches[1]);
            }
        }

        // Try query parameter (less secure, for testing only)
        if (isset($_GET['api_key'])) {
            return $this->authenticate($_GET['api_key']);
        }

        return null;
    }
}
```

### 2. Two-Factor Authentication Support
```php
<?php
// includes/auth/TwoFactorAuthenticator.php

namespace WHMCS\Module\YourModule\Auth;

class TwoFactorAuthenticator
{
    private $db;

    public function __construct()
    {
        global $db;
        $this->db = $db;
    }

    public function requires2FA(int $userId): bool
    {
        $result = $this->db->select(
            "SELECT force_2fa FROM tblusers WHERE id = ?",
            [$userId]
        );

        return !empty($result) && $result[0]['force_2fa'] === 1;
    }

    public function verify2FAToken(int $userId, string $token): bool
    {
        // Get user's 2FA secret
        $result = $this->db->select(
            "SELECT secret FROM tblusers_2fa WHERE user_id = ?",
            [$userId]
        );

        if (empty($result)) {
            return false;
        }

        $secret = $result[0]['secret'];

        // Verify TOTP
        return $this->verifyTOTP($secret, $token);
    }

    private function verifyTOTP(string $secret, string $token): bool
    {
        $timeSlice = floor(time() / 30);

        // Check current and adjacent time slices
        for ($i = -1; $i <= 1; $i++) {
            $calculatedToken = $this->generateTOTP($secret, $timeSlice + $i);
            if (hash_equals($calculatedToken, $token)) {
                return true;
            }
        }

        return false;
    }

    private function generateTOTP(string $secret, int $timeSlice): string
    {
        $secretBinary = base32_decode($secret);

        $timeBinary = pack('N*', 0) . pack('N*', $timeSlice);
        $hash = hash_hmac('sha1', $timeBinary, $secretBinary, true);

        $offset = ord($hash[19]) & 0xf;
        $binary = (
            ((ord($hash[$offset]) & 0x7f) << 24) |
            ((ord($hash[$offset + 1]) & 0xff) << 16) |
            ((ord($hash[$offset + 2]) & 0xff) << 8) |
            (ord($hash[$offset + 3]) & 0xff)
        );

        $otp = $binary % 1000000;
        return str_pad((string)$otp, 6, '0', STR_PAD_LEFT);
    }
}
```

### 3. Create Database Table Migration
```sql
-- Create API keys table
CREATE TABLE IF NOT EXISTS mod_yourmodule_api_keys (
    id INT AUTO_INCREMENT PRIMARY KEY,
    user_id INT NOT NULL,
    api_key_hash VARCHAR(255) NOT NULL,
    api_key_prefix VARCHAR(8) NOT NULL,
    description VARCHAR(255),
    created_at DATETIME NOT NULL,
    expires_at DATETIME,
    revoked_at DATETIME,
    is_active TINYINT(1) DEFAULT 1,
    last_used_at DATETIME,
    rate_limit INT DEFAULT 100,
    INDEX idx_api_key_prefix (api_key_prefix),
    INDEX idx_user_id (user_id),
    INDEX idx_is_active (is_active)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
```

### 4. Authentication Middleware
```php
<?php
// includes/auth/AuthenticationMiddleware.php

namespace WHMCS\Module\YourModule\Auth;

use Illuminate\Http\Request;

class AuthenticationMiddleware
{
    private $apiKeyAuth;
    private $twoFactorAuth;

    public function __construct()
    {
        $this->apiKeyAuth = new ApiKeyAuthenticator();
        $this->twoFactorAuth = new TwoFactorAuthenticator();
    }

    public function handle(Request $request, \Closure $next, bool $require2FA = false)
    {
        $authResult = $this->apiKeyAuth->validateRequest();

        if ($authResult === null) {
            return response()->json([
                'status' => 'error',
                'message' => 'Authentication required'
            ], 401, [
                'WWW-Authenticate' => 'API-Key realm="WHMCS API"'
            ]);
        }

        // Update last used timestamp
        $this->updateLastUsed($authResult['id']);

        // Check 2FA if required
        if ($require2FA && $this->twoFactorAuth->requires2FA($authResult['user_id'])) {
            $twoFactorToken = $request->header('X-2FA-Token');

            if (empty($twoFactorToken)) {
                return response()->json([
                    'status' => 'error',
                    'message' => '2FA verification required',
                    'requires2fa' => true
                ], 403);
            }

            if (!$this->twoFactorAuth->verify2FAToken($authResult['user_id'], $twoFactorToken)) {
                return response()->json([
                    'status' => 'error',
                    'message' => 'Invalid 2FA token'
                ], 403);
            }
        }

        // Attach authenticated user to request
        $request->attributes->set('authenticated_user', $authResult);
        $request->attributes->set('api_key_id', $authResult['id']);

        return $next($request);
    }

    private function updateLastUsed(int $keyId): void
    {
        global $db;
        $db->update('mod_yourmodule_api_keys', [
            'last_used_at' => date('Y-m-d H:i:s')
        ], 'id = ?', [$keyId]);
    }
}
```

## Common Pitfalls and Solutions

| Pitfall | Solution |
|---------|----------|
| API keys stored in plain text | Always hash API keys using Argon2id or bcrypt |
| No expiration on API keys | Implement automatic expiration with cron job |
| Rate limiting bypassed | Apply rate limiting per API key, not just per IP |
| Token replay attacks | Implement one-time tokens for sensitive operations |
| Missing audit logging | Log all authentication attempts with timestamps |

## Security Considerations

1. **Hash API keys before storage** - Never store plain text API keys
2. **Implement key rotation** - Allow users to rotate keys without downtime
3. **Use HTTPS exclusively** - Never transmit API keys over HTTP
4. **Implement scope-based access** - Limit what each API key can access
5. **Monitor for brute force** - Track failed authentication attempts
6. **Set up IP allowlisting** - Restrict keys to specific IP addresses

## Testing Checklist

- [ ] Test authentication with valid API key
- [ ] Test authentication with invalid API key
- [ ] Test authentication with expired API key
- [ ] Test authentication with revoked API key
- [ ] Verify 2FA enforcement when required
- [ ] Test API key generation
- [ ] Test API key revocation
- [ ] Verify rate limiting per API key
- [ ] Test concurrent authentication requests
- [ ] Verify audit logging captures all attempts

## Reference Links

- [WHMCS API Authentication](https://developers.whmcs.com/api/authentication/)
- [PHP Password Hashing](https://www.php.net/manual/en/function.password-hash.php)
- [RFC 7235 - HTTP Authentication](https://tools.ietf.org/html/rfc7235)
- [TOTP Algorithm (RFC 6238)](https://tools.ietf.org/html/rfc6238)
