# WHMCS API Authentication

## Overview

This document covers all authentication methods available in the WHMCS API.

## Authentication Methods

### 1. API Identifier and Secret Key

The primary authentication method uses an API identifier and secret key.

```php
<?php
// Using cURL
$apiUrl = 'https://your-whmcs.com/includes/api.php';
$apiIdentifier = 'your_api_identifier';
$apiSecret = 'your_api_secret';

$postData = [
    'action' => 'GetClients',
    'identifier' => $apiIdentifier,
    'secret' => $apiSecret,
    'username' => 'admin', // Optional admin username for logging
    'responsetype' => 'json',
];

$ch = curl_init();
curl_setopt($ch, CURLOPT_URL, $apiUrl);
curl_setopt($ch, CURLOPT_POST, 1);
curl_setopt($ch, CURLOPT_POSTFIELDS, http_build_query($postData));
curl_setopt($ch, CURLOPT_RETURNTRANSFER, 1);
curl_setopt($ch, CURLOPT_SSL_VERIFYPEER, false);

$response = curl_exec($ch);
curl_close($ch);

$result = json_decode($response, true);
```

### 2. OAuth 2.0 Authentication

For third-party integrations, OAuth 2.0 is supported.

```php
<?php
class WhmcsOAuthClient {
    private string $clientId;
    private string $clientSecret;
    private string $redirectUri;
    private string $authUrl;
    private string $tokenUrl;
    
    public function __construct(
        string $clientId,
        string $clientSecret,
        string $redirectUri,
        string $whmcsUrl
    ) {
        $this->clientId = $clientId;
        $this->clientSecret = $clientSecret;
        $this->redirectUri = $redirectUri;
        $this->authUrl = $whmcsUrl . '/oauth/authorize';
        $this->tokenUrl = $whmcsUrl . '/oauth/token';
    }
    
    public function getAuthorizationUrl(string $state): string
    {
        $params = [
            'client_id' => $this->clientId,
            'redirect_uri' => $this->redirectUri,
            'response_type' => 'code',
            'state' => $state,
        ];
        
        return $this->authUrl . '?' . http_build_query($params);
    }
    
    public function exchangeCodeForToken(string $code): array
    {
        $postData = [
            'grant_type' => 'authorization_code',
            'client_id' => $this->clientId,
            'client_secret' => $this->clientSecret,
            'code' => $code,
            'redirect_uri' => $this->redirectUri,
        ];
        
        return $this->makeTokenRequest($postData);
    }
    
    public function refreshToken(string $refreshToken): array
    {
        $postData = [
            'grant_type' => 'refresh_token',
            'client_id' => $this->clientId,
            'client_secret' => $this->clientSecret,
            'refresh_token' => $refreshToken,
        ];
        
        return $this->makeTokenRequest($postData);
    }
    
    private function makeTokenRequest(array $postData): array
    {
        $ch = curl_init($this->tokenUrl);
        curl_setopt_array($ch, [
            CURLOPT_POST => true,
            CURLOPT_POSTFIELDS => http_build_query($postData),
            CURLOPT_RETURNTRANSFER => true,
            CURLOPT_HTTPHEADER => ['Content-Type: application/x-www-form-urlencoded'],
        ]);
        
        $response = curl_exec($ch);
        curl_close($ch);
        
        return json_decode($response, true);
    }
    
    public function makeApiRequest(string $accessToken, string $endpoint, array $data = []): array
    {
        $url = rtrim($this->redirectUri, '/') . '/api' . $endpoint;
        $data['access_token'] = $accessToken;
        
        $ch = curl_init($url);
        curl_setopt_array($ch, [
            CURLOPT_POST => true,
            CURLOPT_POSTFIELDS => http_build_query($data),
            CURLOPT_RETURNTRANSFER => true,
        ]);
        
        $response = curl_exec($ch);
        curl_close($ch);
        
        return json_decode($response, true);
    }
}
```

### 3. Admin Authentication Token

For internal admin operations, use the admin session token.

```php
<?php
// In WHMCS hooks or modules
use WHMCS\User\Admin;

// Get current admin
$admin = Auth::admin();

// Get API credentials for current admin
$apiIdentifier = $admin->apiIdentifier;
$apiSecret = generateApiSecret(); // Use admin's secret
```

## API Key Management

### Creating API Credentials

```php
<?php
// Via database (admin API management)
function createApiKey(Admin $admin, string $name): array
{
    $apiKey = bin2hex(random_bytes(32));
    $apiSecret = password_hash(bin2hex(random_bytes(32)), PASSWORD_DEFAULT);
    
    Database::table('tblapikeys')
        ->insert([
            'admin_id' => $admin->id,
            'description' => $name,
            'api_key' => hash('sha256', $apiKey),
            'api_secret_hash' => $apiSecret,
            'created_at' => Carbon::now(),
            'last_used' => null,
        ]);
    
    return [
        'api_key' => $apiKey,
        'api_secret' => $apiSecret,
    ];
}
```

### Best Practices

1. **Store credentials securely** - Never commit API secrets to version control
2. **Use environment variables** - Store credentials in `.env` files
3. **Rotate keys regularly** - Implement key rotation policies
4. **Use minimum permissions** - Create role-based API keys
5. **Enable IP restrictions** - Restrict API access by IP address
6. **Monitor usage** - Track API calls for suspicious activity

### Environment Variable Setup

```php
<?php
// config/local.php
return [
    'whmcs_api_url' => getenv('WHMCS_API_URL'),
    'whmcs_api_identifier' => getenv('WHMCS_API_IDENTIFIER'),
    'whmcs_api_secret' => getenv('WHMCS_API_SECRET'),
];
```

## Related Documentation

- [WHMCS API Error Handling](/docs/whmcs-api-error-handling.md)
- [WHMCS API Rate Limiting](/docs/whmcs-api-rate-limiting.md)
- [WHMCS API Webhooks](/docs/whmcs-api-webhooks.md)