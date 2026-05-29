# WHMCS API OAuth Workflow

## Purpose
Guide developers through implementing OAuth 2.0 authentication for WHMCS API access.

## Prerequisites
- WHMCS installation
- OAuth 2.0 understanding
- Third-party OAuth provider
- HTTPS environment

## Steps

### Phase 1: OAuth Overview

1. OAuth flow types
   ```
   OAuth 2.0 Flows:
   ├── Authorization Code (Web apps)
   ├── Client Credentials (Server-to-server)
   ├── Device Flow (CLI tools)
   └── Refresh Token (Long-lived access)
   ```

2. WHMCS OAuth support
   ```
   OAuth in WHMCS:
   - Third-party integrations
   - WHMCS as OAuth provider
   - OAuth for SSO
   ```

### Phase 2: OAuth Implementation

1. Create OAuth handler
   ```php
   <?php
   class WHMCSOAuthHandler {
       private $clientId;
       private $clientSecret;
       private $redirectUri;
       private $authorizationUrl;
       private $tokenUrl;
       
       public function __construct($config) {
           $this->clientId = $config['client_id'];
           $this->clientSecret = $config['client_secret'];
           $this->redirectUri = $config['redirect_uri'];
           $this->authorizationUrl = $config['authorization_url'];
           $this->tokenUrl = $config['token_url'];
       }
       
       public function getAuthorizationUrl($state = null): string {
           $state = $state ?? bin2hex(random_bytes(16));
           $_SESSION['oauth_state'] = $state;
           
           $params = [
               'client_id' => $this->clientId,
               'redirect_uri' => $this->redirectUri,
               'response_type' => 'code',
               'scope' => 'read write',
               'state' => $state,
           ];
           
           return $this->authorizationUrl . '?' . http_build_query($params);
       }
       
       public function exchangeCodeForToken($code, $state): array {
           if (!isset($_SESSION['oauth_state']) || $_SESSION['oauth_state'] !== $state) {
               throw new Exception('Invalid OAuth state');
           }
           
           $ch = curl_init();
           curl_setopt_array($ch, [
               CURLOPT_URL => $this->tokenUrl,
               CURLOPT_POST => true,
               CURLOPT_POSTFIELDS => http_build_query([
                   'grant_type' => 'authorization_code',
                   'code' => $code,
                   'redirect_uri' => $this->redirectUri,
                   'client_id' => $this->clientId,
                   'client_secret' => $this->clientSecret,
               ]),
               CURLOPT_RETURNTRANSFER => true,
           ]);
           
           $response = curl_exec($ch);
           curl_close($ch);
           
           return json_decode($response, true);
       }
       
       public function refreshToken($refreshToken): array {
           $ch = curl_init();
           curl_setopt_array($ch, [
               CURLOPT_URL => $this->tokenUrl,
               CURLOPT_POST => true,
               CURLOPT_POSTFIELDS => http_build_query([
                   'grant_type' => 'refresh_token',
                   'refresh_token' => $refreshToken,
                   'client_id' => $this->clientId,
                   'client_secret' => $this->clientSecret,
               ]),
               CURLOPT_RETURNTRANSFER => true,
           ]);
           
           return json_decode(curl_exec($ch), true);
       }
   }
   ```

2. OAuth callback handler
   ```php
   // Endpoint: /oauth/callback.php
   if (isset($_GET['code']) && isset($_GET['state'])) {
       $handler = new WHMCSOAuthHandler($config);
       
       try {
           $tokens = $handler->exchangeCodeForToken($_GET['code'], $_GET['state']);
           
           // Store tokens securely
           Capsule::table('mod_oauth_tokens')->insert([
               'user_id' => $_SESSION['uid'],
               'access_token' => encrypt($tokens['access_token']),
               'refresh_token' => encrypt($tokens['refresh_token']),
               'expires_at' => date('Y-m-d H:i:s', time() + $tokens['expires_in']),
           ]);
           
           header('Location: ' . $returnUrl);
       } catch (Exception $e) {
           die('OAuth error: ' . $e->getMessage());
       }
   }
   ```

### Phase 3: OAuth in Modules

1. Module OAuth activation
   ```php
   function mymodule_activate(): array {
       Capsule::schema()->create('mod_oauth_tokens', function($table) {
           $table->increments('id');
           $table->integer('user_id');
           $table->string('provider', 100);
           $table->text('access_token');
           $table->text('refresh_token');
           $table->timestamp('expires_at');
           $table->timestamps();
       });
       
       return ['status' => 'success'];
   }
   ```

2. OAuth service wrapper
   ```php
   class OAuthService {
       public function getAccessToken($userId, $provider) {
           $token = Capsule::table('mod_oauth_tokens')
               ->where('user_id', $userId)
               ->where('provider', $provider)
               ->first();
           
           if (!$token) {
               throw new Exception('No OAuth token found');
           }
           
           // Check if expired and refresh if needed
           if (strtotime($token->expires_at) < time()) {
               return $this->refreshAccessToken($token);
           }
           
           return decrypt($token->access_token);
       }
   }
   ```

## Related Workflows
- whmcs-api-authentication
- whmcs-api-sso
- whmcs-api-integration
