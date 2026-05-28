# WHMCS SSO Bridge DevKit
# Version: 1.0 | Updated: 2026-05-28

## DevKit Structure

```
devkits/whmcs-sso-bridge/
├── sso-bridge.php          # Main SSO bridge module
├── lib/
│   ├── SsoProvider.php      # SSO provider interface
│   ├── TokenManager.php     # Token generation/validation
│   └── SessionBridge.php    # Session bridging
├── templates/
│   └── sso-config.tpl       # Configuration template
└── sso-admin.php            # Admin interface
```

## Main SSO Bridge Module

```php
<?php
/**
 * WHMCS SSO Bridge
 * DevKit Template
 * 
 * Provides SSO integration between WHMCS and external systems
 */

if (!defined("WHMCS")) {
    die("Direct access denied");
}

use WHMCS\Database\Capsule;

/**
 * Config function
 */
function {module}_config(): array {
    return [
        'name' => '{SSO Bridge}',
        'description' => 'SSO bridge for external authentication',
        'version' => '1.0',
        'author' => '{Author}',
    ];
}

/**
 * Activate
 */
function {module}_activate(): array {
    Capsule::schema()->create('mod_{module}_sso_providers', function($t) {
        $t->increments('id');
        $t->string('name');
        $t->string('provider_type'); // oauth, saml, jwt, custom
        $t->text('config');
        $t->boolean('is_default');
        $t->boolean('is_active');
        $t->timestamp('created_at');
    });
    
    Capsule::schema()->create('mod_{module}_sso_sessions', function($t) {
        $t->increments('id');
        $t->string('token', 64)->unique();
        $t->integer('user_id');
        $t->string('provider_id');
        $t->string('external_id');
        $t->timestamp('expires_at');
        $t->timestamp('created_at');
    });
    
    Capsule::schema()->create('mod_{module}_sso_logs', function($t) {
        $t->increments('id');
        $t->string('provider');
        $t->string('action');
        $t->text('request_data');
        $t->text('response_data');
        $t->integer('user_id')->nullable();
        $t->boolean('success');
        $t->string('ip_address', 45);
        $t->timestamp('created_at');
    });
    
    return ['status' => 'success', 'description' => 'SSO Bridge activated'];
}

/**
 * Deactivate
 */
function {module}_deactivate(): array {
    Capsule::schema()->dropIfExists('mod_{module}_sso_providers');
    Capsule::schema()->dropIfExists('mod_{module}_sso_sessions');
    Capsule::schema()->dropIfExists('mod_{module}_sso_logs');
    
    return ['status' => 'success'];
}

/**
 * Output function (Admin Interface)
 */
function {module}_output(array $vars): void {
    $action = $_REQUEST['action'] ?? 'dashboard';
    
    if ($_SERVER['REQUEST_METHOD'] === 'POST') {
        check_token('WHMCS.admin.default');
    }
    
    switch ($action) {
        case 'providers':
            {module}_manageProviders();
            break;
        case 'settings':
            {module}_showSettings();
            break;
        case 'logs':
            {module}_viewLogs();
            break;
        case 'test':
            {module}_testConnection();
            break;
        default:
            {module}_showDashboard();
    }
}
```

## SSO Provider Interface

```php
<?php
/**
 * SSO Provider Interface
 */

namespace SsoBridge;

interface SsoProviderInterface {
    public function authenticate(array $params): array;
    public function getUserInfo(string $accessToken): array;
    public function validateToken(string $token): bool;
    public function refreshToken(string $refreshToken): array;
    public function logout(string $accessToken): bool;
}

/**
 * Abstract Base Provider
 */
abstract class AbstractSsoProvider implements SsoProviderInterface {
    
    protected array $config = [];
    protected string $providerName = '';
    
    public function __construct(array $config) {
        $this->config = $config;
    }
    
    abstract public function authenticate(array $params): array;
    abstract public function getUserInfo(string $accessToken): array;
    
    public function validateToken(string $token): bool {
        try {
            $session = Capsule::table('mod_{module}_sso_sessions')
                ->where('token', $token)
                ->where('expires_at', '>', date('Y-m-d H:i:s'))
                ->first();
            
            return $session !== null;
        } catch (\Exception $e) {
            return false;
        }
    }
    
    public function refreshToken(string $refreshToken): array {
        throw new \Exception('Refresh token not supported by this provider');
    }
    
    public function logout(string $accessToken): bool {
        return Capsule::table('mod_{module}_sso_sessions')
            ->where('token', $accessToken)
            ->delete() > 0;
    }
    
    protected function makeRequest(string $url, string $method = 'GET', array $data = [], array $headers = []): array {
        $ch = curl_init();
        
        $defaultHeaders = [
            'Accept: application/json',
        ];
        
        $allHeaders = array_merge($defaultHeaders, $headers);
        
        $options = [
            CURLOPT_URL => $url,
            CURLOPT_RETURNTRANSFER => true,
            CURLOPT_TIMEOUT => 30,
            CURLOPT_HTTPHEADER => $allHeaders,
        ];
        
        if ($method === 'POST') {
            $options[CURLOPT_POST] = true;
            $options[CURLOPT_POSTFIELDS] = http_build_query($data);
        }
        
        curl_setopt_array($ch, $options);
        
        $response = curl_exec($ch);
        $error = curl_error($ch);
        curl_close($ch);
        
        if ($response === false) {
            throw new \Exception('cURL Error: ' . $error);
        }
        
        return json_decode($response, true) ?? [];
    }
}

/**
 * OAuth2 Provider Implementation
 */
class OAuth2Provider extends AbstractSsoProvider {
    
    private string $providerName = 'oauth2';
    
    public function authenticate(array $params): array {
        $code = $params['code'] ?? '';
        
        if (empty($code)) {
            throw new \Exception('Authorization code required');
        }
        
        // Exchange code for access token
        $tokenResponse = $this->exchangeCodeForToken($code);
        
        if (!isset($tokenResponse['access_token'])) {
            throw new \Exception('Failed to obtain access token');
        }
        
        // Get user info
        $userInfo = $this->getUserInfo($tokenResponse['access_token']);
        
        // Create or update WHMCS user
        $userId = $this->syncUser($userInfo);
        
        // Create SSO session
        $sessionToken = $this->createSession($userId, $tokenResponse);
        
        return [
            'success' => true,
            'user_id' => $userId,
            'session_token' => $sessionToken,
            'access_token' => $tokenResponse['access_token'],
            'refresh_token' => $tokenResponse['refresh_token'] ?? null,
            'expires_in' => $tokenResponse['expires_in'] ?? 3600,
        ];
    }
    
    private function exchangeCodeForToken(string $code): array {
        $params = [
            'grant_type' => 'authorization_code',
            'code' => $code,
            'client_id' => $this->config['client_id'],
            'client_secret' => $this->config['client_secret'],
            'redirect_uri' => $this->config['redirect_uri'],
        ];
        
        return $this->makeRequest(
            $this->config['token_url'],
            'POST',
            $params
        );
    }
    
    public function getUserInfo(string $accessToken): array {
        $headers = ['Authorization: Bearer ' . $accessToken];
        
        $response = $this->makeRequest(
            $this->config['userinfo_url'],
            'GET',
            [],
            $headers
        );
        
        return [
            'external_id' => $response['id'] ?? $response['sub'] ?? '',
            'email' => $response['email'] ?? '',
            'first_name' => $response['given_name'] ?? $response['first_name'] ?? '',
            'last_name' => $response['family_name'] ?? $response['last_name'] ?? '',
            'name' => $response['name'] ?? '',
            'picture' => $response['picture'] ?? '',
        ];
    }
    
    private function syncUser(array $userInfo): int {
        $email = $userInfo['email'];
        
        // Find existing client
        $client = Capsule::table('tblclients')
            ->where('email', $email)
            ->first();
        
        if ($client) {
            // Update existing user
            Capsule::table('tblclients')
                ->where('id', $client->id)
                ->update([
                    'firstname' => $userInfo['first_name'] ?: $client->firstname,
                    'lastname' => $userInfo['last_name'] ?: $client->lastname,
                    'updated_at' => date('Y-m-d H:i:s'),
                ]);
            return $client->id;
        }
        
        // Create new user
        return Capsule::table('tblclients')->insertGetId([
            'firstname' => $userInfo['first_name'] ?: 'Unknown',
            'lastname' => $userInfo['last_name'] ?: 'User',
            'email' => $email,
            'datecreated' => date('Y-m-d'),
            'status' => 'Active',
        ]);
    }
    
    private function createSession(int $userId, array $tokenResponse): string {
        $token = bin2hex(random_bytes(32));
        $expiresAt = date('Y-m-d H:i:s', time() + ($tokenResponse['expires_in'] ?? 3600));
        
        Capsule::table('mod_{module}_sso_sessions')->insert([
            'token' => $token,
            'user_id' => $userId,
            'provider_id' => $this->config['provider_id'] ?? 'oauth2',
            'external_id' => '',
            'expires_at' => $expiresAt,
            'created_at' => date('Y-m-d H:i:s'),
        ]);
        
        return $token;
    }
}

/**
 * SAML Provider Implementation
 */
class SamlProvider extends AbstractSsoProvider {
    
    private string $providerName = 'saml';
    
    public function authenticate(array $params): array {
        $samlResponse = $params['SAMLResponse'] ?? '';
        
        if (empty($samlResponse)) {
            throw new \Exception('SAML response required');
        }
        
        // Decode SAML response
        $userInfo = $this->decodeSamlResponse($samlResponse);
        
        // Sync user
        $userId = $this->syncUser($userInfo);
        
        // Create session
        $sessionToken = $this->createSession($userId);
        
        return [
            'success' => true,
            'user_id' => $userId,
            'session_token' => $sessionToken,
        ];
    }
    
    private function decodeSamlResponse(string $response): array {
        // Base64 decode
        $decoded = base64_decode($response);
        
        // Parse XML (simplified - use a proper SAML library in production)
        $xml = simplexml_load_string($decoded);
        
        // Extract user attributes (depends on IdP configuration)
        return [
            'external_id' => (string) $xml->xpath('//NameID')[0] ?? '',
            'email' => (string) $xml->xpath('//Email')[0] ?? '',
            'first_name' => (string) $xml->xpath('//FirstName')[0] ?? '',
            'last_name' => (string) $xml->xpath('//LastName')[0] ?? '',
        ];
    }
    
    public function getUserInfo(string $accessToken): array {
        // SAML doesn't use access tokens in the same way
        return [];
    }
    
    private function syncUser(array $userInfo): int {
        // Implementation similar to OAuth2
        $email = $userInfo['email'];
        
        $client = Capsule::table('tblclients')
            ->where('email', $email)
            ->first();
        
        if ($client) {
            return $client->id;
        }
        
        return Capsule::table('tblclients')->insertGetId([
            'firstname' => $userInfo['first_name'] ?: 'Unknown',
            'lastname' => $userInfo['last_name'] ?: 'User',
            'email' => $email,
            'datecreated' => date('Y-m-d'),
            'status' => 'Active',
        ]);
    }
    
    private function createSession(int $userId): string {
        $token = bin2hex(random_bytes(32));
        
        Capsule::table('mod_{module}_sso_sessions')->insert([
            'token' => $token,
            'user_id' => $userId,
            'provider_id' => 'saml',
            'external_id' => '',
            'expires_at' => date('Y-m-d H:i:s', time() + 86400),
            'created_at' => date('Y-m-d H:i:s'),
        ]);
        
        return $token;
    }
}
```

## Token Manager

```php
<?php
/**
 * SSO Token Manager
 */

namespace SsoBridge;

use WHMCS\Database\Capsule;

class TokenManager {
    
    /**
     * Generate secure token
     */
    public static function generate(int $length = 32): string {
        return bin2hex(random_bytes($length));
    }
    
    /**
     * Create session token
     */
    public static function createSession(int $userId, string $providerId, int $expiresIn = 3600): string {
        $token = self::generate();
        
        Capsule::table('mod_{module}_sso_sessions')->insert([
            'token' => hash('sha256', $token),
            'user_id' => $userId,
            'provider_id' => $providerId,
            'external_id' => '',
            'expires_at' => date('Y-m-d H:i:s', time() + $expiresIn),
            'created_at' => date('Y-m-d H:i:s'),
        ]);
        
        return $token; // Return plain token (stored hashed)
    }
    
    /**
     * Validate session token
     */
    public static function validate(string $token): ?array {
        $hash = hash('sha256', $token);
        
        $session = Capsule::table('mod_{module}_sso_sessions')
            ->where('token', $hash)
            ->where('expires_at', '>', date('Y-m-d H:i:s'))
            ->first();
        
        if (!$session) {
            return null;
        }
        
        return [
            'id' => $session->id,
            'user_id' => $session->user_id,
            'provider_id' => $session->provider_id,
            'expires_at' => $session->expires_at,
        ];
    }
    
    /**
     * Invalidate token (logout)
     */
    public static function invalidate(string $token): bool {
        $hash = hash('sha256', $token);
        
        return Capsule::table('mod_{module}_sso_sessions')
            ->where('token', $hash)
            ->delete() > 0;
    }
    
    /**
     * Invalidate all sessions for user
     */
    public static function invalidateUser(int $userId): int {
        return Capsule::table('mod_{module}_sso_sessions')
            ->where('user_id', $userId)
            ->delete();
    }
    
    /**
     * Cleanup expired sessions
     */
    public static function cleanup(): int {
        return Capsule::table('mod_{module}_sso_sessions')
            ->where('expires_at', '<', date('Y-m-d H:i:s'))
            ->delete();
    }
    
    /**
     * Refresh session expiration
     */
    public static function refresh(string $token, int $expiresIn = 3600): bool {
        $hash = hash('sha256', $token);
        
        return Capsule::table('mod_{module}_sso_sessions')
            ->where('token', $hash)
            ->update([
                'expires_at' => date('Y-m-d H:i:s', time() + $expiresIn),
            ]) > 0;
    }
}
```

## Session Bridge

```php
<?php
/**
 * Session Bridge
 * Bridges external SSO session to WHMCS session
 */

namespace SsoBridge;

use WHMCS\Database\Capsule;

class SessionBridge {
    
    /**
     * Bridge SSO session to WHMCS
     */
    public static function bridgeToWhmcs(string $token): bool {
        $session = TokenManager::validate($token);
        
        if (!$session) {
            return false;
        }
        
        $userId = $session['user_id'];
        
        // Get client details
        $client = Capsule::table('tblclients')
            ->where('id', $userId)
            ->first();
        
        if (!$client) {
            return false;
        }
        
        // Set WHMCS session
        $_SESSION['uid'] = $userId;
        $_SESSION['username'] = $client->email;
        $_SESSION['upassword'] = $client->password;
        $_SESSION['userid'] = $userId;
        $_SESSION['clientsdetails']['id'] = $userId;
        $_SESSION['clientsdetails']['email'] = $client->email;
        $_SESSION['clientsdetails']['firstname'] = $client->firstname;
        $_SESSION['clientsdetails']['lastname'] = $client->lastname;
        
        // Regenerate session ID
        if (session_status() === PHP_SESSION_ACTIVE) {
            session_regenerate_id(true);
        }
        
        return true;
    }
    
    /**
     * Create WHMCS session from SSO user
     */
    public static function createWhmcsSession(int $userId): void {
        $client = Capsule::table('tblclients')
            ->where('id', $userId)
            ->first();
        
        if (!$client) {
            throw new \Exception('Client not found');
        }
        
        // Set session variables
        $_SESSION['uid'] = $userId;
        $_SESSION['username'] = $client->email;
        $_SESSION['upassword'] = $client->password;
        $_SESSION['userid'] = $userId;
        $_SESSION['clientsdetails'] = [
            'id' => $client->id,
            'email' => $client->email,
            'firstname' => $client->firstname,
            'lastname' => $client->lastname,
            'companyname' => $client->companyname,
        ];
        
        // Update last login
        Capsule::table('tblclients')
            ->where('id', $userId)
            ->update(['lastlogin' => date('Y-m-d H:i:s')]);
        
        // Regenerate session
        session_regenerate_id(true);
    }
    
    /**
     * Clear SSO and WHMCS session
     */
    public static function clearSession(string $token = null): void {
        if ($token) {
            TokenManager::invalidate($token);
        }
        
        // Clear WHMCS session
        unset($_SESSION['uid']);
        unset($_SESSION['username']);
        unset($_SESSION['upassword']);
        unset($_SESSION['userid']);
        unset($_SESSION['clientsdetails']);
        
        // Regenerate session ID
        if (session_status() === PHP_SESSION_ACTIVE) {
            session_regenerate_id(true);
        }
    }
}
```

## SSO Endpoints

```php
<?php
/**
 * SSO Callback Handler
 * Handles OAuth/SAML callbacks
 */

function {module}_callback(): void {
    $provider = $_REQUEST['provider'] ?? 'default';
    
    try {
        // Load provider
        $providerConfig = Capsule::table('mod_{module}_sso_providers')
            ->where('name', $provider)
            ->where('is_active', 1)
            ->first();
        
        if (!$providerConfig) {
            throw new \Exception('Provider not found');
        }
        
        $config = json_decode($providerConfig->config, true);
        $config['provider_id'] = $providerConfig->id;
        
        // Create provider instance
        $ssoProvider = {module}_createProvider($providerConfig->provider_type, $config);
        
        // Authenticate
        $result = $ssoProvider->authenticate($_REQUEST);
        
        // Bridge to WHMCS session
        SessionBridge::createWhmcsSession($result['user_id']);
        
        // Redirect to WHMCS
        $redirectUrl = $config['success_redirect'] ?? 'clientarea.php';
        header('Location: ' . $redirectUrl);
        exit;
        
    } catch (\Exception $e) {
        // Log error
        {module}_logSsoAction($provider, 'callback_error', [], ['error' => $e->getMessage()], false);
        
        // Redirect to error page
        $errorUrl = $config['error_redirect'] ?? 'index.php';
        header('Location: ' . $errorUrl . '?error=' . urlencode($e->getMessage()));
        exit;
    }
}

/**
 * Create provider instance
 */
function {module}_createProvider(string $type, array $config): SsoProviderInterface {
    switch ($type) {
        case 'oauth':
            return new OAuth2Provider($config);
        case 'saml':
            return new SamlProvider($config);
        default:
            throw new \Exception('Unknown provider type: ' . $type);
    }
}

/**
 * Initiate SSO login
 */
function {module}_initiate(): void {
    $provider = $_REQUEST['provider'] ?? 'default';
    
    $providerConfig = Capsule::table('mod_{module}_sso_providers')
        ->where('name', $provider)
        ->where('is_active', 1)
        ->first();
    
    if (!$providerConfig) {
        throw new \Exception('Provider not found');
    }
    
    $config = json_decode($providerConfig->config, true);
    
    // Build authorization URL based on provider type
    switch ($providerConfig->provider_type) {
        case 'oauth':
            $authUrl = $config['authorization_url'];
            $params = [
                'client_id' => $config['client_id'],
                'redirect_uri' => $config['redirect_uri'],
                'response_type' => 'code',
                'scope' => $config['scope'] ?? 'read:user',
                'state' => bin2hex(random_bytes(16)),
            ];
            $url = $authUrl . '?' . http_build_query($params);
            break;
            
        case 'saml':
            // Build SAML AuthnRequest
            $url = {module}_buildSamlRequest($config);
            break;
            
        default:
            throw new \Exception('Unknown provider type');
    }
    
    header('Location: ' . $url);
    exit;
}

/**
 * SSO Login Hook
 */
function {module}_loginPage(): void {
    $providers = Capsule::table('mod_{module}_sso_providers')
        ->where('is_active', 1)
        ->get();
    
    if (empty($providers)) {
        return;
    }
    
    echo '<div class="sso-login-section">';
    echo '<p>Or sign in with:</p>';
    
    foreach ($providers as $provider) {
        $config = json_decode($provider->config, true);
        $icon = $config['icon'] ?? 'fa-sign-in';
        
        echo '<a href="?module={module}&action=initiate&provider=' . urlencode($provider->name) . '" class="btn btn-' . ($config['button_style'] ?? 'default') . '">';
        echo '<i class="fa ' . $icon . '"></i> ' . htmlspecialchars($provider->name);
        echo '</a> ';
    }
    
    echo '</div>';
}

add_hook('ClientAreaPageLogin', 1, function($vars) {
    {module}_loginPage();
});
```

## Admin Configuration Template

```smarty
<div class="sso-bridge">
    <h2>SSO Bridge Configuration</h2>
    
    <div class="panel panel-default">
        <div class="panel-heading">SSO Providers</div>
        <div class="panel-body">
            <table class="table">
                <thead>
                    <tr>
                        <th>Provider</th>
                        <th>Type</th>
                        <th>Status</th>
                        <th>Actions</th>
                    </tr>
                </thead>
                <tbody>
                    {foreach $providers as $provider}
                    <tr>
                        <td>{$provider.name}</td>
                        <td>{$provider.provider_type}</td>
                        <td>
                            {if $provider.is_active}
                                <span class="label label-success">Active</span>
                            {else}
                                <span class="label label-default">Inactive</span>
                            {/if}
                        </td>
                        <td>
                            <a href="?module={module}&action=providers&sub=edit&id={$provider.id}" class="btn btn-xs">Edit</a>
                        </td>
                    </tr>
                    {/foreach}
                </tbody>
            </table>
            
            <a href="?module={module}&action=providers&sub=add" class="btn btn-success">
                <i class="fa fa-plus"></i> Add Provider
            </a>
        </div>
    </div>
    
    <div class="panel panel-default">
        <div class="panel-heading">OAuth2 Configuration</div>
        <div class="panel-body">
            <form method="post">
                <input type="hidden" name="csrf_token" value="{$csrf_token}">
                
                <div class="form-group">
                    <label>Client ID</label>
                    <input type="text" name="client_id" class="form-control" value="{$config.client_id}">
                </div>
                
                <div class="form-group">
                    <label>Client Secret</label>
                    <input type="password" name="client_secret" class="form-control" value="{$config.client_secret}">
                </div>
                
                <div class="form-group">
                    <label>Authorization URL</label>
                    <input type="url" name="authorization_url" class="form-control" value="{$config.authorization_url}">
                </div>
                
                <div class="form-group">
                    <label>Token URL</label>
                    <input type="url" name="token_url" class="form-control" value="{$config.token_url}">
                </div>
                
                <div class="form-group">
                    <label>User Info URL</label>
                    <input type="url" name="userinfo_url" class="form-control" value="{$config.userinfo_url}">
                </div>
                
                <div class="form-group">
                    <label>Callback URL</label>
                    <input type="text" class="form-control" readonly value="{$base_url}modules/addons/{module}/callback.php">
                    <span class="help-block">Use this URL in your OAuth application</span>
                </div>
                
                <button type="submit" class="btn btn-primary">Save Configuration</button>
            </form>
        </div>
    </div>
</div>
```

## Checklist

```
Pre-Dev:
□ Identify SSO provider type (OAuth, SAML, JWT)
□ Plan token management strategy
□ Define user attribute mapping
□ Plan session bridging approach

Development:
□ Create SSO provider interface
□ Implement OAuth2 provider
□ Implement SAML provider (optional)
□ Create TokenManager class
□ Create SessionBridge class
□ Implement callback handler
□ Implement initiate endpoint
□ Add login page hook
□ Build admin configuration UI
□ Add logging

Testing:
□ Test OAuth flow
□ Test SAML flow (if implemented)
□ Test token validation
□ Test session bridging
□ Test logout functionality
□ Verify user attribute mapping
```