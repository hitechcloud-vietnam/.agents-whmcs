# WHMCS OAuth Integration

Complete guide for OAuth authentication with WHMCS.

## Overview

OAuth allows secure API access without sharing credentials.

## OAuth Flow

### OAuth Client Registration

```php
<?php
/**
 * OAuth client for WHMCS API
 */
class WHMCSOAuthClient
{
    private string $clientId;
    private string $clientSecret;
    private string $redirectUri;
    private string $authUrl;
    private string $tokenUrl;
    
    public function __construct(array $config)
    {
        $this->clientId = $config['client_id'];
        $this->clientSecret = $config['client_secret'];
        $this->redirectUri = $config['redirect_uri'];
        $this->authUrl = rtrim($config['base_url'], '/') . '/oauth/authorize';
        $this->tokenUrl = rtrim($config['base_url'], '/') . '/oauth/token';
    }
    
    /**
     * Generate authorization URL
     */
    public function getAuthorizationUrl(string $state): string
    {
        $params = [
            'client_id' => $this->clientId,
            'redirect_uri' => $this->redirectUri,
            'response_type' => 'code',
            'scope' => 'clientarea:read clientarea:write',
            'state' => $state,
        ];
        
        return $this->authUrl . '?' . http_build_query($params);
    }
    
    /**
     * Exchange authorization code for tokens
     */
    public function exchangeCode(string $code): array
    {
        $ch = curl_init($this->tokenUrl);
        curl_setopt_array($ch, [
            CURLOPT_POST => true,
            CURLOPT_POSTFIELDS => http_build_query([
                'grant_type' => 'authorization_code',
                'client_id' => $this->clientId,
                'client_secret' => $this->clientSecret,
                'code' => $code,
                'redirect_uri' => $this->redirectUri,
            ]),
            CURLOPT_RETURNTRANSFER => true,
        ]);
        
        $response = curl_exec($ch);
        curl_close($ch);
        
        return json_decode($response, true);
    }
    
    /**
     * Refresh access token
     */
    public function refreshToken(string $refreshToken): array
    {
        $ch = curl_init($this->tokenUrl);
        curl_setopt_array($ch, [
            CURLOPT_POST => true,
            CURLOPT_POSTFIELDS => http_build_query([
                'grant_type' => 'refresh_token',
                'client_id' => $this->clientId,
                'client_secret' => $this->clientSecret,
                'refresh_token' => $refreshToken,
            ]),
            CURLOPT_RETURNTRANSFER => true,
        ]);
        
        $response = curl_exec($ch);
        curl_close($ch);
        
        return json_decode($response, true);
    }
}
```

### OAuth Authorization Flow

```php
<?php
/**
 * OAuth callback handler
 */
function handleOAuthCallback(): void
{
    $error = $_GET['error'] ?? null;
    if ($error) {
        throw new Exception("OAuth error: {$error}");
    }
    
    $code = $_GET['code'] ?? null;
    $state = $_GET['state'] ?? null;
    
    // Verify state
    $savedState = $_SESSION['oauth_state'];
    if ($state !== $savedState) {
        throw new Exception('Invalid state parameter');
    }
    
    // Exchange code for tokens
    $oauthClient = new WHMCSOAuthClient([
        'client_id' => OAUTH_CLIENT_ID,
        'client_secret' => OAUTH_CLIENT_SECRET,
        'redirect_uri' => OAUTH_REDIRECT_URI,
        'base_url' => WHMCS_URL,
    ]);
    
    $tokens = $oauthClient->exchangeCode($code);
    
    if (isset($tokens['error'])) {
        throw new Exception($tokens['error_description'] ?? 'Token exchange failed');
    }
    
    // Store tokens securely
    $_SESSION['access_token'] = $tokens['access_token'];
    $_SESSION['refresh_token'] = $tokens['refresh_token'];
    $_SESSION['token_expires'] = time() + $tokens['expires_in'];
    
    // Redirect to dashboard
    header('Location: /dashboard');
    exit;
}
```

## OAuth API Calls

### Authenticated Requests

```php
<?php
/**
 * Make authenticated API call
 */
class WHMCSOAuthApiClient
{
    private string $baseUrl;
    private string $accessToken;
    private int $tokenExpires;
    private string $refreshToken;
    private string $clientId;
    private string $clientSecret;
    private string $redirectUri;
    
    public function __construct(array $config)
    {
        $this->baseUrl = rtrim($config['base_url'], '/');
        $this->accessToken = $config['access_token'];
        $this->tokenExpires = $config['token_expires'];
        $this->refreshToken = $config['refresh_token'];
        $this->clientId = $config['client_id'];
        $this->clientSecret = $config['client_secret'];
        $this->redirectUri = $config['redirect_uri'];
    }
    
    /**
     * Ensure valid access token
     */
    private function ensureValidToken(): void
    {
        if (time() >= ($this->tokenExpires - 60)) {
            $this->refreshAccessToken();
        }
    }
    
    /**
     * Refresh access token
     */
    private function refreshAccessToken(): void
    {
        $oauthClient = new WHMCSOAuthClient([
            'client_id' => $this->clientId,
            'client_secret' => $this->clientSecret,
            'redirect_uri' => $this->redirectUri,
            'base_url' => $this->baseUrl,
        ]);
        
        $tokens = $oauthClient->refreshToken($this->refreshToken);
        
        if (isset($tokens['error'])) {
            throw new Exception('Token refresh failed: ' . $tokens['error']);
        }
        
        $this->accessToken = $tokens['access_token'];
        $this->refreshToken = $tokens['refresh_token'];
        $this->tokenExpires = time() + $tokens['expires_in'];
        
        // Persist updated tokens
        $this->saveTokens();
    }
    
    /**
     * Make API call
     */
    public function call(string $action, array $params = []): array
    {
        $this->ensureValidToken();
        
        $params['action'] = $action;
        $params['responsetype'] = 'json';
        
        $ch = curl_init($this->baseUrl . '/includes/api.php');
        curl_setopt_array($ch, [
            CURLOPT_POST => true,
            CURLOPT_POSTFIELDS => http_build_query($params),
            CURLOPT_RETURNTRANSFER => true,
            CURLOPT_HTTPHEADER => [
                'Authorization: Bearer ' . $this->accessToken,
            ],
        ]);
        
        $response = curl_exec($ch);
        curl_close($ch);
        
        return json_decode($response, true);
    }
    
    /**
     * Save tokens to storage
     */
    private function saveTokens(): void
    {
        // Save to secure session or database
    }
}
```

## OAuth Scopes

### Available Scopes

```php
<?php
/**
 * OAuth scopes
 */
const OAUTH_SCOPES = [
    'clientarea:read' => 'Read client area data',
    'clientarea:write' => 'Modify client area data',
    'profile:read' => 'Read user profile',
    'profile:write' => 'Modify user profile',
    'invoices:read' => 'Read invoices',
    'invoices:write' => 'Create and modify invoices',
    'services:read' => 'Read services',
    'services:write' => 'Create and modify services',
    'domains:read' => 'Read domains',
    'domains:write' => 'Register and modify domains',
    'tickets:read' => 'Read support tickets',
    'tickets:write' => 'Create and modify tickets',
    'notifications:write' => 'Send notifications',
];

/**
 * Request specific scopes
 */
function getAuthorizationUrlWithScopes(array $scopes, string $state): string
{
    $oauthClient = new WHMCSOAuthClient($config);
    
    $scopeString = implode(' ', $scopes);
    
    $params = [
        'client_id' => $oauthClient->getClientId(),
        'redirect_uri' => $oauthClient->getRedirectUri(),
        'response_type' => 'code',
        'scope' => $scopeString,
        'state' => $state,
    ];
    
    return $oauthClient->getAuthUrl() . '?' . http_build_query($params);
}
```

## Token Storage

### Secure Token Storage

```php
<?php
/**
 * Store tokens securely
 */
class SecureTokenStorage
{
    private PDO $db;
    
    public function __construct()
    {
        $this->db = Capsule::connection()->getPdo();
    }
    
    /**
     * Store tokens for user
     */
    public function storeTokens(int $userId, array $tokens): void
    {
        $stmt = $this->db->prepare('
            INSERT INTO oauth_tokens (user_id, access_token, refresh_token, expires_at)
            VALUES (?, ?, ?, ?)
            ON DUPLICATE KEY UPDATE
                access_token = VALUES(access_token),
                refresh_token = VALUES(refresh_token),
                expires_at = VALUES(expires_at)
        ');
        
        $stmt->execute([
            $userId,
            $tokens['access_token'],
            $tokens['refresh_token'],
            date('Y-m-d H:i:s', time() + $tokens['expires_in']),
        ]);
    }
    
    /**
     * Get valid access token
     */
    public function getValidToken(int $userId): ?array
    {
        $stmt = $this->db->prepare('
            SELECT * FROM oauth_tokens
            WHERE user_id = ? AND expires_at > NOW()
        ');
        
        $stmt->execute([$userId]);
        $token = $stmt->fetch(PDO::FETCH_ASSOC);
        
        return $token ?: null;
    }
    
    /**
     * Delete tokens
     */
    public function deleteTokens(int $userId): void
    {
        $stmt = $this->db->prepare('DELETE FROM oauth_tokens WHERE user_id = ?');
        $stmt->execute([$userId]);
    }
}
```

## PKCE Extension

### PKCE Flow

```php
<?php
/**
 * OAuth with PKCE extension
 */
class PKCEOAuthClient
{
    private string $codeVerifier;
    private string $codeChallenge;
    
    public function __construct()
    {
        $this->codeVerifier = $this->generateCodeVerifier();
        $this->codeChallenge = $this->generateCodeChallenge();
    }
    
    /**
     * Generate code verifier
     */
    private function generateCodeVerifier(): string
    {
        return rtrim(strtr(base64_encode(random_bytes(32)), '+/', '-_'), '=');
    }
    
    /**
     * Generate code challenge from verifier
     */
    private function generateCodeChallenge(): string
    {
        return rtrim(strtr(base64_encode(hash('sha256', $this->codeVerifier, true)), '+/', '-_'), '=');
    }
    
    /**
     * Get authorization URL with PKCE
     */
    public function getAuthorizationUrl(string $clientId, string $redirectUri, string $state): string
    {
        $params = [
            'client_id' => $clientId,
            'redirect_uri' => $redirectUri,
            'response_type' => 'code',
            'scope' => 'clientarea:read',
            'state' => $state,
            'code_challenge' => $this->codeChallenge,
            'code_challenge_method' => 'S256',
        ];
        
        return 'https://whmcs.example.com/oauth/authorize?' . http_build_query($params);
    }
    
    /**
     * Exchange code with verifier
     */
    public function exchangeCode(string $code, string $clientId, string $clientSecret, string $redirectUri): array
    {
        $ch = curl_init('https://whmcs.example.com/oauth/token');
        curl_setopt_array($ch, [
            CURLOPT_POST => true,
            CURLOPT_POSTFIELDS => http_build_query([
                'grant_type' => 'authorization_code',
                'client_id' => $clientId,
                'client_secret' => $clientSecret,
                'code' => $code,
                'redirect_uri' => $redirectUri,
                'code_verifier' => $this->codeVerifier,
            ]),
            CURLOPT_RETURNTRANSFER => true,
        ]);
        
        $response = curl_exec($ch);
        curl_close($ch);
        
        return json_decode($response, true);
    }
    
    public function getCodeVerifier(): string
    {
        return $this->codeVerifier;
    }
}
```

## Best Practices

1. **Use PKCE for public clients** - Add extra security
2. **Store tokens securely** - Encrypt at rest
3. **Implement refresh tokens** - Maintain long sessions
4. **Validate all tokens** - Check expiration
5. **Handle token revocation** - Support logout
6. **Use HTTPS only** - Never send tokens over HTTP

## Related Documentation

- [whmcs-integration-api.md](whmcs-integration-api.md)
- [whmcs-integration-authentication.md](whmcs-integration-authentication.md)
