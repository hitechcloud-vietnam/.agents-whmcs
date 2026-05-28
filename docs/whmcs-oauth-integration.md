# WHMCS OAuth Integration Guide

**Version:** 8.x | **Updated:** 2026-05-29
**Related Skills:** `whmcs-api-integration`, `whmcs-sso-integration`, `whmcs-security-hardening`

---

## Overview

OAuth 2.0 integration enables third-party applications to securely authenticate WHMCS users and access WHMCS functionality on their behalf. This guide covers implementing OAuth 2.0 authorization code flow for WHMCS integrations.

---

## OAuth 2.0 Flow

### Authorization Code Flow

```
Client App                    WHMCS                      User
    |                           |                         |
    |--- Authorization Request ->|                         |
    |                           |--- Login Page ---------->|
    |                           |<-- Credentials ---------|
    |                           |                         |
    |<-- Auth Code -------------|                         |
    |                           |                         |
    |--- Token Exchange ------->|                         |
    |<-- Access Token ---------|                         |
    |                           |                         |
    |--- API Request --------->|                         |
    |<-- Response -------------|                         |
```

### Supported Grant Types

- **Authorization Code** - Standard web application flow
- **Refresh Token** - Token renewal without re-authentication
- **Client Credentials** - Machine-to-machine authentication

---

## Configuration

### Registering an OAuth Application

```php
<?php
use WHMCS\User\Application\Application;

// Create OAuth application
$application = Application::create([
    'name' => 'My Integration',
    'description' => 'Third-party integration',
    'client_id' => generateSecureToken(32),
    'client_secret' => password_hash(generateSecureToken(64), PASSWORD_ARGON2ID),
    'redirect_uri' => 'https://myapp.example.com/callback',
    'grant_types' => ['authorization_code', 'refresh_token'],
    'scopes' => ['openid', 'profile', 'invoices:read', 'services:read'],
    'user_id' => 1, // Admin user creating the app
    'is_confidential' => true,
]);
```

### Application Scopes

| Scope | Description |
|-------|-------------|
| `openid` | OpenID Connect identity |
| `profile` | User profile information |
| `invoices:read` | Read invoice data |
| `invoices:write` | Create/modify invoices |
| `services:read` | Read service data |
| `services:write` | Create/modify services |
| `clients:read` | Read client data |
| `clients:write` | Create/modify clients |
| `tickets:read` | Read support tickets |
| `tickets:write` | Create/modify tickets |

---

## Authorization Endpoint

### Building Authorization URL

```php
<?php
/**
 * Generate OAuth authorization URL
 */
function buildAuthorizationUrl(
    string $clientId,
    string $redirectUri,
    array $scopes,
    string $state = null,
    string $codeChallenge = null
): string {

    $params = [
        'client_id' => $clientId,
        'redirect_uri' => urlencode($redirectUri),
        'response_type' => 'code',
        'scope' => implode(' ', $scopes),
        'state' => $state ?? bin2hex(random_bytes(16)),
    ];

    if ($codeChallenge) {
        $params['code_challenge'] = $codeChallenge;
        $params['code_challenge_method'] = 'S256';
    }

    return WHMCS\Config\Setting::getValue('SystemURL')
        . 'oauth/authorize.php?'
        . http_build_query($params);
}

// PKCE Code Challenge
function generateCodeChallenge(string $codeVerifier): string
{
    return rtrim(strtr(base64_encode(
        hash('sha256', $codeVerifier, true)
    ), '+/', '-_'), '=');
}

// Usage
$codeVerifier = bin2hex(random_bytes(32));
$codeChallenge = generateCodeChallenge($codeVerifier);

$authUrl = buildAuthorizationUrl(
    'your_client_id',
    'https://myapp.example.com/callback',
    ['openid', 'profile', 'invoices:read'],
    'random_state_string',
    $codeChallenge
);
```

### Authorization Handler

```php
<?php
// oauth/authorize.php (WHMCS endpoint)

// Validate request
$clientId = $_GET['client_id'];
$redirectUri = $_GET['redirect_uri'];
$state = $_GET['state'];

// Verify client_id and redirect_uri match registration
$application = Application::where('client_id', $clientId)->first();

if (!$application || $application->redirect_uri !== $redirectUri) {
    logActivity("OAuth auth failed: Invalid client");
    header('Location: ' . $redirectUri . '?error=invalid_client');
    exit;
}

// Display login form (or use existing session)
if (!$_SESSION['adminid']) {
    // Show WHMCS login
    include __DIR__ . '/../login.php';
    exit;
}

// User approved - generate code
$authCode = bin2hex(random_bytes(32));

// Store code with expiry (10 minutes)
Cache::put('oauth_code_' . $authCode, [
    'client_id' => $clientId,
    'user_id' => $_SESSION['adminid'],
    'scopes' => explode(' ', $_GET['scope']),
    'redirect_uri' => $redirectUri,
    'code_challenge' => $_GET['code_challenge'] ?? null,
    'created_at' => time(),
], 600);

header('Location: ' . $redirectUri
    . '?code=' . $authCode
    . '&state=' . $state);
```

---

## Token Exchange

### Exchanging Code for Tokens

```php
<?php
/**
 * Exchange authorization code for tokens
 */
function exchangeCodeForTokens(
    string $authCode,
    string $clientId,
    string $clientSecret,
    string $redirectUri,
    string $codeVerifier = null
): array {

    $whmcsUrl = WHMCS\Config\Setting::getValue('SystemURL');

    $tokenEndpoint = $whmcsUrl . 'oauth/token.php';

    $params = [
        'grant_type' => 'authorization_code',
        'code' => $authCode,
        'client_id' => $clientId,
        'client_secret' => $clientSecret,
        'redirect_uri' => $redirectUri,
    ];

    if ($codeVerifier) {
        $params['code_verifier'] = $codeVerifier;
    }

    $ch = curl_init($tokenEndpoint);
    curl_setopt_array($ch, [
        CURLOPT_RETURNTRANSFER => true,
        CURLOPT_POST => true,
        CURLOPT_POSTFIELDS => http_build_query($params),
        CURLOPT_HTTPHEADER => ['Accept: application/json'],
    ]);

    $response = curl_exec($ch);
    $httpCode = curl_getinfo($ch, CURLINFO_HTTP_CODE);
    curl_close($ch);

    $data = json_decode($response, true);

    if ($httpCode !== 200) {
        throw new \Exception(
            $data['error'] ?? 'Token exchange failed'
        );
    }

    return $data;
}

/**
 * Token response structure
 */
$tokens = [
    'access_token' => 'eyJhbGciOiJSUzI1NiIs...',
    'token_type' => 'Bearer',
    'expires_in' => 3600,
    'refresh_token' => 'dGhpcyBpcyBhIHJl...',
    'scope' => 'openid profile invoices:read',
];
```

### Refresh Token Flow

```php
<?php
/**
 * Refresh access token
 */
function refreshAccessToken(
    string $refreshToken,
    string $clientId,
    string $clientSecret
): array {

    $whmcsUrl = WHMCS\Config\Setting::getValue('SystemURL');

    $ch = curl_init($whmcsUrl . 'oauth/token.php');
    curl_setopt_array($ch, [
        CURLOPT_RETURNTRANSFER => true,
        CURLOPT_POST => true,
        CURLOPT_POSTFIELDS => http_build_query([
            'grant_type' => 'refresh_token',
            'refresh_token' => $refreshToken,
            'client_id' => $clientId,
            'client_secret' => $clientSecret,
        ]),
        CURLOPT_HTTPHEADER => ['Accept: application/json'],
    ]);

    $response = curl_exec($ch);
    curl_close($ch);

    return json_decode($response, true);
}
```

---

## Using Access Tokens

### Making Authenticated Requests

```php
<?php
/**
 * Make authenticated API request
 */
function apiRequest(
    string $accessToken,
    string $endpoint,
    string $method = 'GET',
    array $data = []
): array {

    $whmcsUrl = WHMCS\Config\Setting::getValue('SystemURL');
    $url = $whmcsUrl . 'api/v2/' . ltrim($endpoint, '/');

    $ch = curl_init($url);
    curl_setopt_array($ch, [
        CURLOPT_RETURNTRANSFER => true,
        CURLOPT_TIMEOUT => 30,
        CURLOPT_HTTPHEADER => [
            'Authorization: Bearer ' . $accessToken,
            'Accept: application/json',
            'Content-Type: application/json',
        ],
    ]);

    if ($method !== 'GET') {
        curl_setopt($ch, CURLOPT_CUSTOMREQUEST, $method);
        curl_setopt($ch, CURLOPT_POSTFIELDS, json_encode($data));
    }

    $response = curl_exec($ch);
    $httpCode = curl_getinfo($ch, CURLINFO_HTTP_CODE);
    curl_close($ch);

    return [
        'status' => $httpCode,
        'body' => json_decode($response, true),
    ];
}

// Usage examples
$response = apiRequest($token, 'clients');
$response = apiRequest($token, 'invoices/' . $invoiceId);
$response = apiRequest($token, 'services', 'POST', [
    'client_id' => 123,
    'product_id' => 1,
]);
```

---

## JWT Token Validation

### Validating Access Tokens

```php
<?php
use Firebase\JWT\JWT;
use Firebase\JWT\Key;

/**
 * Validate and decode JWT access token
 */
function validateAccessToken(string $jwt, string $clientSecret): array
{
    try {
        // Decode JWT
        $decoded = JWT::decode(
            $jwt,
            new Key($clientSecret, 'HS256')
        );

        return [
            'valid' => true,
            'claims' => [
                'sub' => $decoded->sub,      // User ID
                'scope' => $decoded->scope,  // Granted scopes
                'exp' => $decoded->exp,      // Expiration
                'iat' => $decoded->iat,      // Issued at
                'iss' => $decoded->iss,      // Issuer (WHMCS URL)
                'aud' => $decoded->aud,      // Client ID
            ],
        ];

    } catch (\Exception $e) {
        return [
            'valid' => false,
            'error' => $e->getMessage(),
        ];
    }
}

/**
 * JWT Payload Structure
 */
$jwtPayload = (object) [
    'iss' => 'https://whmcs.example.com/',
    'sub' => '123',                    // WHMCS user ID
    'aud' => 'your_client_id',         // OAuth client ID
    'exp' => time() + 3600,           // Expiration timestamp
    'iat' => time(),                   // Issued at timestamp
    'scope' => 'openid profile invoices:read',
    'email' => 'admin@example.com',
    'name' => 'Admin User',
];
```

---

## Security Best Practices

### PKCE Implementation

```php
<?php
/**
 * PKCE (Proof Key for Code Exchange) implementation
 */

// 1. Generate code verifier (43-128 chars, alphanumeric + - . _ ~)
$codeVerifier = bin2hex(random_bytes(32));

// 2. Generate code challenge (S256 method recommended)
$codeChallenge = rtrim(strtr(
    base64_encode(hash('sha256', $codeVerifier, true)),
    '+/', '-_'
), '=');

// 3. Include in authorization request
$authUrl = buildAuthorizationUrl(
    $clientId,
    $redirectUri,
    $scopes,
    $state,
    $codeChallenge
);

// 4. Include verifier in token exchange
$tokens = exchangeCodeForTokens(
    $authCode,
    $clientId,
    $clientSecret,
    $redirectUri,
    $codeVerifier  // Required for PKCE
);
```

### Token Storage

```php
<?php
// NEVER store tokens in localStorage or sessionStorage

// Secure storage for web applications
$_SESSION['oauth_tokens'] = [
    'access_token' => encryptSensitive($tokens['access_token']),
    'refresh_token' => encryptSensitive($tokens['refresh_token']),
    'expires_at' => time() + $tokens['expires_in'],
];

// Database storage (encrypted)
Capsule::table('oauth_tokens')->insert([
    'user_id' => $userId,
    'client_id' => $clientId,
    'access_token' => encrypt($tokens['access_token']),
    'refresh_token' => encrypt($tokens['refresh_token']),
    'expires_at' => date('Y-m-d H:i:s', time() + $tokens['expires_in']),
    'created_at' => date('Y-m-d H:i:s'),
]);

// Encryption helper
function encryptSensitive(string $value): string
{
    $key = \WHMCS\Session::get('encryption_key');
    $iv = random_bytes(16);
    $encrypted = openssl_encrypt($value, 'aes-256-cbc', $key, 0, $iv);
    return base64_encode($iv . $encrypted);
}
```

### Token Revocation

```php
<?php
/**
 * Revoke OAuth token
 */
function revokeToken(string $accessToken, string $clientId, string $clientSecret): bool
{
    $whmcsUrl = WHMCS\Config\Setting::getValue('SystemURL');

    $ch = curl_init($whmcsUrl . 'oauth/revoke.php');
    curl_setopt_array($ch, [
        CURLOPT_RETURNTRANSFER => true,
        CURLOPT_POST => true,
        CURLOPT_POSTFIELDS => http_build_query([
            'token' => $accessToken,
            'client_id' => $clientId,
            'client_secret' => $clientSecret,
        ]),
    ]);

    $response = curl_exec($ch);
    $httpCode = curl_getinfo($ch, CURLINFO_HTTP_CODE);
    curl_close($ch);

    return $httpCode === 200;
}
```

---

## Use Cases

### 1. Billing Portal Integration

```php
<?php
// Third-party billing portal accessing WHMCS invoices
class BillingPortalIntegration
{
    private string $clientId;
    private string $clientSecret;
    private string $redirectUri;

    public function __construct()
    {
        $this->clientId = getenv('OAUTH_CLIENT_ID');
        $this->clientSecret = getenv('OAUTH_CLIENT_SECRET');
        $this->redirectUri = 'https://portal.example.com/callback';
    }

    public function getClientInvoices(string $accessToken, int $clientId): array
    {
        $response = apiRequest(
            $accessToken,
            "clients/{$clientId}/invoices"
        );

        return $response['body']['data'] ?? [];
    }

    public function getClientServices(string $accessToken, int $clientId): array
    {
        $response = apiRequest(
            $accessToken,
            "clients/{$clientId}/services"
        );

        return $response['body']['data'] ?? [];
    }
}
```

### 2. CRM Synchronization

```php
<?php
// Sync WHMCS clients to external CRM
class CrmSyncIntegration
{
    public function syncClient(string $accessToken, array $clientData): bool
    {
        // Get full client details
        $response = apiRequest(
            $accessToken,
            "clients/{$clientData['id']}"
        );

        if ($response['status'] !== 200) {
            return false;
        }

        $client = $response['body']['data'];

        // Push to CRM
        $crm = new ExternalCrmClient();
        $crm->upsertContact([
            'email' => $client['email'],
            'name' => $client['fullname'],
            'company' => $client['companyname'],
            'phone' => $client['phonenumber'],
            'whmcs_id' => $client['id'],
        ]);

        return true;
    }
}
```

---

## Error Handling

### OAuth Error Responses

```php
<?php
/**
 * OAuth error handling
 */
function handleOAuthError(string $error, string $description = null): void
{
    $errorMessages = [
        'invalid_client' => 'Client authentication failed',
        'invalid_grant' => 'Invalid or expired authorization code',
        'invalid_request' => 'Missing required parameter',
        'invalid_scope' => 'Requested scope not allowed',
        'unauthorized_client' => 'Client not authorized for this grant',
        'unsupported_grant_type' => 'Grant type not supported',
        'access_denied' => 'User denied the authorization request',
    ];

    $message = $errorMessages[$error] ?? $error;

    if ($description) {
        $message .= ': ' . $description;
    }

    logActivity("OAuth Error: {$message}");

    throw new \Exception($message);
}
```

---

## Related Documentation

- [API Integration Patterns](api-integration-patterns.md)
- [API Gateway](api-gateway.md)
- [SSO Integration](../skills/whmcs-sso-integration.md)
- [Security Best Practices](security-best-practices.md)
