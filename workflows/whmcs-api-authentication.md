# WHMCS API Authentication Patterns

## Purpose

Comprehensive guide to implementing secure API authentication in WHMCS, covering API key management, OAuth2 flows, JWT tokens, and best practices for secure API integration.

## Prerequisites

- WHMCS installation with API access enabled
- Understanding of authentication protocols
- SSL certificate for production environments
- API credentials from target service

## Workflow Steps

### Step 1: WHMCS API Key Authentication

Configure WHMCS native API access:

```php
// Enable API access in WHMCS Admin
// Navigate to: Configuration > System Settings > API Credentials

// Generate API credentials programmatically
function createApiCredentials($adminId, $apiKey, $permissions)
{
    $result = localAPI('CreateApiCredential', [
        'admin_id' => $adminId,
        'api_key' => $apiKey,
        'permissions' => $permissions,
    ]);
    
    return $result;
}

// Usage example
$apiParams = [
    'identifier' => 'your_api_identifier',
    'secret' => 'your_api_secret',
    'url' => 'https://your-whmcs.com/includes/api.php',
];

$ch = curl_init();
curl_setopt_array($ch, [
    CURLOPT_URL => $apiParams['url'],
    CURLOPT_RETURNTRANSFER => true,
    CURLOPT_POST => true,
    CURLOPT_POSTFIELDS => http_build_query([
        'action' => 'GetClients',
        'api_key' => $apiParams['secret'],
        'identifier' => $apiParams['identifier'],
        'responsetype' => 'json',
    ]),
]);

$response = curl_exec($ch);
curl_close($ch);
```

### Step 2: API Key Authentication for Third-Party Services

Implement API key authentication for external service integration:

```php
// modules/servers/myprovider/myprovider.php

class MyProviderAPI
{
    private $apiKey;
    private $apiSecret;
    private $baseUrl;
    private $signatureMethod = 'HMAC-SHA256';
    
    public function __construct($apiKey, $apiSecret, $baseUrl)
    {
        $this->apiKey = $apiKey;
        $this->apiSecret = $apiSecret;
        $this->baseUrl = rtrim($baseUrl, '/');
    }
    
    /**
     * Generate authentication headers with signature
     */
    private function generateAuthHeaders(array $params = []): array
    {
        $timestamp = time();
        $nonce = bin2hex(random_bytes(16));
        
        // Build signature payload
        $payload = implode('|', [
            $this->apiKey,
            $timestamp,
            $nonce,
            http_build_query($params),
        ]);
        
        // Generate HMAC signature
        $signature = hash_hmac('sha256', $payload, $this->apiSecret);
        
        return [
            'X-API-Key: ' . $this->apiKey,
            'X-Timestamp: ' . $timestamp,
            'X-Nonce: ' . $nonce,
            'X-Signature: ' . $signature,
            'Content-Type: application/json',
        ];
    }
    
    /**
     * Make authenticated API request
     */
    public function request(string $method, string $endpoint, array $data = []): array
    {
        $url = $this->baseUrl . '/' . ltrim($endpoint, '/');
        $headers = $this->generateAuthHeaders($data);
        
        $ch = curl_init();
        curl_setopt_array($ch, [
            CURLOPT_URL => $url,
            CURLOPT_RETURNTRANSFER => true,
            CURLOPT_TIMEOUT => 30,
            CURLOPT_HTTPHEADER => $headers,
        ]);
        
        if ($method === 'POST') {
            curl_setopt($ch, CURLOPT_POST, true);
            curl_setopt($ch, CURLOPT_POSTFIELDS, json_encode($data));
        } elseif ($method !== 'GET') {
            curl_setopt($ch, CURLOPT_CUSTOMREQUEST, $method);
            curl_setopt($ch, CURLOPT_POSTFIELDS, json_encode($data));
        }
        
        $response = curl_exec($ch);
        $httpCode = curl_getinfo($ch, CURLINFO_HTTP_CODE);
        $error = curl_error($ch);
        curl_close($ch);
        
        if ($error) {
            throw new Exception("API request failed: {$error}");
        }
        
        $decoded = json_decode($response, true);
        
        if ($httpCode >= 400) {
            throw new Exception(
                "API error ({$httpCode}): " . ($decoded['message'] ?? 'Unknown error')
            );
        }
        
        return $decoded;
    }
    
    /**
     * Verify webhook signature
     */
    public function verifyWebhookSignature(string $payload, array $headers): bool
    {
        $signature = $headers['X-Signature'] ?? '';
        $timestamp = $headers['X-Timestamp'] ?? '';
        
        // Check timestamp to prevent replay attacks
        if (abs(time() - (int)$timestamp) > 300) {
            return false;
        }
        
        $expectedSignature = hash_hmac(
            'sha256',
            $payload . $timestamp,
            $this->apiSecret
        );
        
        return hash_equals($expectedSignature, $signature);
    }
}

// Usage in module
function myprovider_TestConnection(array $params): array
{
    try {
        $api = new MyProviderAPI(
            $params['serverusername'],
            $params['serverpassword'],
            $params['serverhostname']
        );
        
        $response = $api->request('GET', '/v1/account/verify');
        
        return ['success' => true];
    } catch (Exception $e) {
        return ['success' => false, 'error' => $e->getMessage()];
    }
}
```

### Step 3: OAuth2 Integration

Implement OAuth2 authentication flow:

```php
// modules/addons/oauth_integration/oauth_integration.php

add_hook('ClientAreaPrimaryNavbar', 1, function($vars) {
    return [
        'OAuthLogin' => [
            'uri' => 'oauth/login',
            'label' => 'Login with Provider',
            'icon' => 'fa-lock',
        ],
    ];
});

/**
 * OAuth2 Helper Class
 */
class OAuth2Helper
{
    private $clientId;
    private $clientSecret;
    private $redirectUri;
    private $authorizationUrl;
    private $tokenUrl;
    private $scopes;
    
    public function __construct(array $config)
    {
        $this->clientId = $config['client_id'];
        $this->clientSecret = $config['client_secret'];
        $this->redirectUri = $config['redirect_uri'];
        $this->authorizationUrl = $config['authorization_url'];
        $this->tokenUrl = $config['token_url'];
        $this->scopes = $config['scopes'] ?? [];
    }
    
    /**
     * Generate authorization URL
     */
    public function getAuthorizationUrl(string $state = null): string
    {
        $state = $state ?? bin2hex(random_bytes(16));
        
        $_SESSION['oauth2_state'] = $state;
        
        $params = [
            'client_id' => $this->clientId,
            'redirect_uri' => $this->redirectUri,
            'response_type' => 'code',
            'scope' => implode(' ', $this->scopes),
            'state' => $state,
        ];
        
        return $this->authorizationUrl . '?' . http_build_query($params);
    }
    
    /**
     * Exchange authorization code for access token
     */
    public function exchangeCodeForToken(string $code, string $state): array
    {
        // Verify state to prevent CSRF
        if (!isset($_SESSION['oauth2_state']) || $_SESSION['oauth2_state'] !== $state) {
            throw new Exception('Invalid state parameter');
        }
        unset($_SESSION['oauth2_state']);
        
        $ch = curl_init();
        curl_setopt_array($ch, [
            CURLOPT_URL => $this->tokenUrl,
            CURLOPT_RETURNTRANSFER => true,
            CURLOPT_POST => true,
            CURLOPT_POSTFIELDS => http_build_query([
                'grant_type' => 'authorization_code',
                'code' => $code,
                'redirect_uri' => $this->redirectUri,
                'client_id' => $this->clientId,
                'client_secret' => $this->clientSecret,
            ]),
        ]);
        
        $response = curl_exec($ch);
        curl_close($ch);
        
        $tokenData = json_decode($response, true);
        
        if (isset($tokenData['error'])) {
            throw new Exception($tokenData['error_description'] ?? $tokenData['error']);
        }
        
        // Store tokens securely
        return $this->storeTokens($tokenData);
    }
    
    /**
     * Refresh expired access token
     */
    public function refreshAccessToken(string $refreshToken): array
    {
        $ch = curl_init();
        curl_setopt_array($ch, [
            CURLOPT_URL => $this->tokenUrl,
            CURLOPT_RETURNTRANSFER => true,
            CURLOPT_POST => true,
            CURLOPT_POSTFIELDS => http_build_query([
                'grant_type' => 'refresh_token',
                'refresh_token' => $refreshToken,
                'client_id' => $this->clientId,
                'client_secret' => $this->clientSecret,
            ]),
        ]);
        
        $response = curl_exec($ch);
        curl_close($ch);
        
        return json_decode($response, true);
    }
    
    /**
     * Store tokens securely in database
     */
    private function storeTokens(array $tokenData): array
    {
        $encryptedAccessToken = encrypt($tokenData['access_token']);
        $encryptedRefreshToken = encrypt($tokenData['refresh_token']);
        
        Capsule::table('mod_oauth_tokens')->updateOrInsert(
            ['user_id' => $_SESSION['uid']],
            [
                'access_token' => $encryptedAccessToken,
                'refresh_token' => $encryptedRefreshToken,
                'expires_at' => date('Y-m-d H:i:s', time() + $tokenData['expires_in']),
                'created_at' => date('Y-m-d H:i:s'),
            ]
        );
        
        return $tokenData;
    }
}
```

### Step 4: JWT Token Authentication

Implement JWT-based authentication:

```php
// includes/jwt_handler.php

class JWTAuthHandler
{
    private $secretKey;
    private $algorithm = 'HS256';
    private $issuer = 'whmcs';
    private $ttl = 3600; // 1 hour
    
    public function __construct($secretKey = null)
    {
        $this->secretKey = $secretKey ?? getWHMCSConfig('API Secret');
    }
    
    /**
     * Generate JWT token
     */
    public function generateToken(array $payload, int $ttl = null): string
    {
        $ttl = $ttl ?? $this->ttl;
        
        $header = [
            'typ' => 'JWT',
            'alg' => $this->algorithm,
        ];
        
        $payload['iat'] = time();
        $payload['exp'] = time() + $ttl;
        $payload['iss'] = $this->issuer;
        
        $headerEncoded = $this->base64UrlEncode(json_encode($header));
        $payloadEncoded = $this->base64UrlEncode(json_encode($payload));
        
        $signature = hash_hmac(
            'sha256',
            "{$headerEncoded}.{$payloadEncoded}",
            $this->secretKey,
            true
        );
        $signatureEncoded = $this->base64UrlEncode($signature);
        
        return "{$headerEncoded}.{$payloadEncoded}.{$signatureEncoded}";
    }
    
    /**
     * Validate and decode JWT token
     */
    public function validateToken(string $token): ?array
    {
        $parts = explode('.', $token);
        
        if (count($parts) !== 3) {
            return null;
        }
        
        [$headerEncoded, $payloadEncoded, $signature] = $parts;
        
        // Verify signature
        $expectedSignature = $this->base64UrlEncode(
            hash_hmac(
                'sha256',
                "{$headerEncoded}.{$payloadEncoded}",
                $this->secretKey,
                true
            )
        );
        
        if (!hash_equals($expectedSignature, $signature)) {
            return null;
        }
        
        // Decode payload
        $payload = json_decode($this->base64UrlDecode($payloadEncoded), true);
        
        // Check expiration
        if (isset($payload['exp']) && $payload['exp'] < time()) {
            return null;
        }
        
        // Check issuer
        if (isset($payload['iss']) && $payload['iss'] !== $this->issuer) {
            return null;
        }
        
        return $payload;
    }
    
    private function base64UrlEncode(string $data): string
    {
        return rtrim(strtr(base64_encode($data), '+/', '-_'), '=');
    }
    
    private function base64UrlDecode(string $data): string
    {
        return base64_decode(strtr($data, '-_', '+/'));
    }
}

/**
 * JWT Authentication hook
 */
add_hook('ApiAuthorization', 1, function($vars) {
    $token = $_SERVER['HTTP_AUTHORIZATION'] ?? '';
    
    if (preg_match('/^Bearer\s+(.+)$/i', $token, $matches)) {
        $jwt = new JWTAuthHandler();
        $payload = $jwt->validateToken($matches[1]);
        
        if ($payload) {
            return [
                'authorized' => true,
                'user_id' => $payload['user_id'] ?? null,
                'permissions' => $payload['permissions'] ?? [],
            ];
        }
    }
    
    return ['authorized' => false, 'error' => 'Invalid or expired token'];
});
```

### Step 5: Secure Credential Storage

Implement secure API credential management:

```php
// Secure credential storage helper
class SecureCredentialStorage
{
    /**
     * Encrypt and store API credentials
     */
    public static function storeApiCredentials(int $userId, string $provider, array $credentials): void
    {
        $encrypted = encrypt(json_encode($credentials));
        
        Capsule::table('mod_secure_credentials')->updateOrInsert(
            [
                'user_id' => $userId,
                'provider' => $provider,
            ],
            [
                'credentials' => $encrypted,
                'updated_at' => date('Y-m-d H:i:s'),
            ]
        );
    }
    
    /**
     * Retrieve and decrypt API credentials
     */
    public static function getApiCredentials(int $userId, string $provider): ?array
    {
        $row = Capsule::table('mod_secure_credentials')
            ->where('user_id', $userId)
            ->where('provider', $provider)
            ->first();
        
        if (!$row) {
            return null;
        }
        
        return json_decode(decrypt($row->credentials), true);
    }
    
    /**
     * Delete stored credentials
     */
    public static function deleteApiCredentials(int $userId, string $provider): void
    {
        Capsule::table('mod_secure_credentials')
            ->where('user_id', $userId)
            ->where('provider', $provider)
            ->delete();
    }
}

/**
 * Module activation - create secure storage table
 */
function securecreds_activate(): array
{
    if (!Capsule::schema()->hasTable('mod_secure_credentials')) {
        Capsule::schema()->create('mod_secure_credentials', function($table) {
            $table->increments('id');
            $table->integer('user_id');
            $table->string('provider', 100);
            $table->text('credentials'); // Encrypted
            $table->timestamp('updated_at')->useCurrent();
            $table->unique(['user_id', 'provider']);
        });
    }
    
    return ['status' => 'success'];
}
```

## Best Practices

1. **Never expose secrets in URLs** - Use headers or request bodies for sensitive data
2. **Implement token expiration** - Short-lived tokens reduce risk of compromise
3. **Use secure storage** - Encrypt API keys and secrets at rest
4. **Validate signatures** - Always verify webhook/callback signatures
5. **Implement rate limiting** - Prevent brute force attacks
6. **Log authentication attempts** - Monitor for suspicious activity
7. **Use HTTPS only** - Never transmit credentials over unencrypted channels
8. **Rotate credentials** - Implement credential rotation policies

## Common Pitfalls to Avoid

1. **Storing plaintext credentials** - Always encrypt sensitive data
2. **Long-lived tokens** - Use short expiration times
3. **Not validating signatures** - Always verify webhook authenticity
4. **Logging sensitive data** - Never log passwords or tokens
5. **Hardcoding secrets** - Use environment variables or secure vaults
6. **Ignoring token refresh** - Handle token expiration gracefully
7. **Missing CSRF protection** - Validate state parameters in OAuth flows
