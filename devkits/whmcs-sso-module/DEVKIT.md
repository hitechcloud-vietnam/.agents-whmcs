# WHMCS SSO Module DevKit
# Version: 1.0 | Updated: 2026-05-28

## DevKit Structure

```
devkits/whmcs-sso-module/
├── sso.php              # SSO handler
├── lib/
│   ├── SsoProvider.php   # SSO provider class
│   ├── TokenManager.php  # Token management
│   └── SessionHandler.php
└── templates/
    ├── login.tpl        # Custom login template
    └── admin.tpl         # Admin settings
```

## SSO Handler Template

```php
<?php
/**
 * WHMCS SSO Module: {Module}
 * DevKit Template
 * 
 * Installation: Upload to modules/addons/{module}/
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
        'name' => '{SSO Module}',
        'description' => 'Single Sign-On integration with external systems',
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
        $t->string('provider_name');
        $t->string('provider_type'); // oauth2, saml, ldap
        $t->text('config');
        $t->boolean('is_active');
        $t->timestamp('created_at');
    });
    
    Capsule::schema()->create('mod_{module}_sso_sessions', function($t) {
        $t->increments('id');
        $t->integer('user_id');
        $t->string('token', 64);
        $t->string('provider');
        $t->text('metadata');
        $t->timestamp('expires_at');
        $t->timestamp('created_at');
    });
    
    Capsule::schema()->create('mod_{module}_sso_users', function($t) {
        $t->increments('id');
        $t->integer('user_id');
        $t->string('provider');
        $t->string('provider_uid');
        $t->text('profile_data');
        $t->timestamp('linked_at');
    });
    
    return ['status' => 'success', 'description' => 'Module activated'];
}

/**
 * Deactivate
 */
function {module}_deactivate(): array {
    Capsule::schema()->dropIfExists('mod_{module}_sso_providers');
    Capsule::schema()->dropIfExists('mod_{module}_sso_sessions');
    Capsule::schema()->dropIfExists('mod_{module}_sso_users');
    
    return ['status' => 'success'];
}

/**
 * Output function
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
        case 'save_provider':
            {module}_saveProvider();
            break;
        case 'delete_provider':
            {module}_deleteProvider();
            break;
        case 'sessions':
            {module}_showSessions();
            break;
        case 'settings':
            {module}_showSettings();
            break;
        case 'logs':
            {module}_showLogs();
            break;
        default:
            {module}_showDashboard();
    }
}

/**
 * Show Dashboard
 */
function {module}_showDashboard(): void {
    $stats = [
        'active_providers' => Capsule::table('mod_{module}_sso_providers')
            ->where('is_active', 1)->count(),
        'active_sessions' => Capsule::table('mod_{module}_sso_sessions')
            ->where('expires_at', '>', date('Y-m-d H:i:s'))->count(),
        'linked_users' => Capsule::table('mod_{module}_sso_users')->count(),
    ];
    
    echo <<<HTML
<div class="sso-module">
    <div class="row">
        <div class="col-md-12">
            <div class="panel panel-default">
                <div class="panel-heading">
                    <h3 class="panel-title">Single Sign-On Dashboard</h3>
                </div>
                <div class="panel-body">
                    <div class="row">
                        <div class="col-md-4">
                            <div class="stat-box">
                                <div class="stat-value">{$stats['active_providers']}</div>
                                <div class="stat-label">Active Providers</div>
                            </div>
                        </div>
                        <div class="col-md-4">
                            <div class="stat-box">
                                <div class="stat-value">{$stats['active_sessions']}</div>
                                <div class="stat-label">Active Sessions</div>
                            </div>
                        </div>
                        <div class="col-md-4">
                            <div class="stat-box">
                                <div class="stat-value">{$stats['linked_users']}</div>
                                <div class="stat-label">Linked Users</div>
                            </div>
                        </div>
                    </div>
                </div>
            </div>
        </div>
    </div>
    
    <div class="row">
        <div class="col-md-12">
            <div class="btn-group">
                <a href="?module={module}&action=providers" class="btn btn-primary">
                    <i class="fa fa-plug"></i> Manage Providers
                </a>
                <a href="?module={module}&action=sessions" class="btn btn-default">
                    <i class="fa fa-clock"></i> Active Sessions
                </a>
                <a href="?module={module}&action=settings" class="btn btn-default">
                    <i class="fa fa-cog"></i> Settings
                </a>
                <a href="?module={module}&action=logs" class="btn btn-default">
                    <i class="fa fa-file-alt"></i> Logs
                </a>
            </div>
        </div>
    </div>
    
    <div class="row" style="margin-top: 20px;">
        <div class="col-md-6">
            <div class="panel panel-default">
                <div class="panel-heading">
                    <h3 class="panel-title">Quick Start</h3>
                </div>
                <div class="panel-body">
                    <ol>
                        <li>Go to <strong>Manage Providers</strong> to add SSO providers</li>
                        <li>Configure OAuth2/SAML/LDAP settings</li>
                        <li>Enable the provider</li>
                        <li>Users can link their accounts from the client area</li>
                    </ol>
                    
                    <h4>Available Endpoints</h4>
                    <ul class="list-unstyled">
                        <li><code>./sso.php?provider=google</code> - Initiate SSO login</li>
                        <li><code>./sso.php?callback=1</code> - OAuth callback</li>
                        <li><code>./sso.php?logout=1</code> - Logout</li>
                    </ul>
                </div>
            </div>
        </div>
        
        <div class="col-md-6">
            <div class="panel panel-default">
                <div class="panel-heading">
                    <h3 class="panel-title">Recent Activity</h3>
                </div>
                <div class="panel-body">
                    <table class="table table-striped">
                        <thead>
                            <tr>
                                <th>Time</th>
                                <th>User</th>
                                <th>Provider</th>
                                <th>Action</th>
                            </tr>
                        </thead>
                        <tbody>
HTML;
    
    $recentActivity = Capsule::table('mod_{module}_sso_users')
        ->limit(5)
        ->orderBy('linked_at', 'desc')
        ->get();
    
    foreach ($recentActivity as $activity) {
        echo "<tr>
            <td>{$activity->linked_at}</td>
            <td>{$activity->user_id}</td>
            <td>{$activity->provider}</td>
            <td>Linked</td>
        </tr>";
    }
    
    echo "</tbody></table></div></div></div></div></div>";
}
```

## SSO Provider Class

```php
<?php
namespace {Module};

use WHMCS\Database\Capsule;

class SsoProvider {
    
    public static function oauth2AuthorizeUrl(string $provider, string $redirectUrl, array $config): string {
        $params = [
            'client_id' => $config['client_id'],
            'redirect_uri' => $redirectUrl,
            'response_type' => 'code',
            'scope' => $config['scope'] ?? 'openid email profile',
            'state' => self::generateState($provider),
        ];
        
        return $config['auth_url'] . '?' . http_build_query($params);
    }
    
    public static function oauth2Callback(string $provider, array $config, string $code): array {
        $tokenUrl = $config['token_url'];
        
        $ch = curl_init();
        curl_setopt_array($ch, [
            CURLOPT_URL => $tokenUrl,
            CURLOPT_POST => true,
            CURLOPT_POSTFIELDS => http_build_query([
                'grant_type' => 'authorization_code',
                'code' => $code,
                'redirect_uri' => $config['redirect_uri'],
                'client_id' => $config['client_id'],
                'client_secret' => $config['client_secret'],
            ]),
            CURLOPT_RETURNTRANSFER => true,
        ]);
        
        $tokenResponse = json_decode(curl_exec($ch), true);
        curl_close($ch);
        
        if (isset($tokenResponse['access_token'])) {
            return self::oauth2GetUserInfo($config['user_info_url'], $tokenResponse['access_token']);
        }
        
        throw new \Exception('Failed to get access token');
    }
    
    public static function oauth2GetUserInfo(string $url, string $accessToken): array {
        $ch = curl_init();
        curl_setopt_array($ch, [
            CURLOPT_URL => $url,
            CURLOPT_HTTPHEADER => ['Authorization: Bearer ' . $accessToken],
            CURLOPT_RETURNTRANSFER => true,
        ]);
        
        $response = json_decode(curl_exec($ch), true);
        curl_close($ch);
        
        return $response;
    }
    
    public static function generateState(string $provider): string {
        $state = bin2hex(random_bytes(16));
        $_SESSION['sso_state'][$provider] = $state;
        return $state;
    }
    
    public static function validateState(string $provider, string $state): bool {
        return isset($_SESSION['sso_state'][$provider]) 
            && $_SESSION['sso_state'][$provider] === $state;
    }
    
    public static function linkOrCreateUser(string $provider, array $profile): object {
        // Check if already linked
        $existingLink = Capsule::table('mod_{module}_sso_users')
            ->where('provider', $provider)
            ->where('provider_uid', $profile['id'])
            ->first();
        
        if ($existingLink) {
            return Capsule::table('tblclients')
                ->where('id', $existingLink->user_id)
                ->first();
        }
        
        // Check if user exists by email
        if (!empty($profile['email'])) {
            $existingUser = Capsule::table('tblclients')
                ->where('email', $profile['email'])
                ->first();
            
            if ($existingUser) {
                // Link existing user
                Capsule::table('mod_{module}_sso_users')->insert([
                    'user_id' => $existingUser->id,
                    'provider' => $provider,
                    'provider_uid' => $profile['id'],
                    'profile_data' => json_encode($profile),
                    'linked_at' => date('Y-m-d H:i:s'),
                ]);
                
                return $existingUser;
            }
        }
        
        // Create new user
        $userId = Capsule::table('tblclients')->insertGetId([
            'firstname' => $profile['first_name'] ?? 'SSO',
            'lastname' => $profile['last_name'] ?? 'User',
            'email' => $profile['email'] ?? $profile['id'] . '@' . $provider . '.sso',
            'password' => encrypt(bin2hex(random_bytes(12))),
            'created_at' => date('Y-m-d H:i:s'),
            'notes' => "Created via {$provider} SSO",
        ]);
        
        // Link new user
        Capsule::table('mod_{module}_sso_users')->insert([
            'user_id' => $userId,
            'provider' => $provider,
            'provider_uid' => $profile['id'],
            'profile_data' => json_encode($profile),
            'linked_at' => date('Y-m-d H:i:s'),
        ]);
        
        return Capsule::table('tblclients')->where('id', $userId)->first();
    }
    
    public static function createSession(object $user, string $provider): string {
        $token = bin2hex(random_bytes(32));
        
        Capsule::table('mod_{module}_sso_sessions')->insert([
            'user_id' => $user->id,
            'token' => hash('sha256', $token),
            'provider' => $provider,
            'metadata' => json_encode([
                'ip' => $_SERVER['REMOTE_ADDR'] ?? '',
                'user_agent' => $_SERVER['HTTP_USER_AGENT'] ?? '',
            ]),
            'expires_at' => date('Y-m-d H:i:s', strtotime('+24 hours')),
            'created_at' => date('Y-m-d H:i:s'),
        ]);
        
        return $token;
    }
    
    public static function validateSession(string $token): ?object {
        $hash = hash('sha256', $token);
        
        $session = Capsule::table('mod_{module}_sso_sessions')
            ->where('token', $hash)
            ->where('expires_at', '>', date('Y-m-d H:i:s'))
            ->first();
        
        if ($session) {
            return Capsule::table('tblclients')->where('id', $session->user_id)->first();
        }
        
        return null;
    }
    
    public static function logout(string $token): bool {
        $hash = hash('sha256', $token);
        
        return Capsule::table('mod_{module}_sso_sessions')
            ->where('token', $hash)
            ->delete() > 0;
    }
}
```

## SSO Callback Handler (sso.php)

```php
<?php
/**
 * WHMCS SSO Module - Entry Point
 */

require_once __DIR__ . '/../../init.php';
require_once __DIR__ . '/../../includes/functions.php';

if (!defined("WHMCS")) {
    die("Direct access denied");
}

// Handle SSO initiation
if (isset($_GET['provider'])) {
    $provider = $_GET['provider'];
    $providerConfig = getProviderConfig($provider);
    
    if (!$providerConfig) {
        die('Provider not found');
    }
    
    $redirectUrl = $systemurl . '/modules/addons/{module}/sso.php?callback=1&provider=' . $provider;
    $authUrl = \{Module}\SsoProvider::oauth2AuthorizeUrl($provider, $redirectUrl, $providerConfig);
    
    header('Location: ' . $authUrl);
    exit;
}

// Handle OAuth callback
if (isset($_GET['callback']) && $_GET['callback'] == '1') {
    $provider = $_GET['provider'];
    $code = $_GET['code'];
    $state = $_GET['state'] ?? '';
    
    if (!\{Module}\SsoProvider::validateState($provider, $state)) {
        die('Invalid state parameter');
    }
    
    $providerConfig = getProviderConfig($provider);
    $profile = \{Module}\SsoProvider::oauth2Callback($provider, $providerConfig, $code);
    
    // Link or create user
    $user = \{Module}\SsoProvider::linkOrCreateUser($provider, $profile);
    
    // Create session
    $token = \{Module}\SsoProvider::createSession($user, $provider);
    
    // Redirect to WHMCS with token
    header('Location: ' . $systemurl . '/clientarea.php?sso_token=' . $token);
    exit;
}

// Handle logout
if (isset($_GET['logout'])) {
    if (isset($_COOKIE['sso_token'])) {
        \{Module}\SsoProvider::logout($_COOKIE['sso_token']);
        setcookie('sso_token', '', time() - 3600, '/');
    }
    
    header('Location: ' . $systemurl . '/index.php');
    exit;
}
```

## Checklist

```
Pre-Dev:
□ Identify SSO provider type (OAuth2, SAML, LDAP)
□ Get provider documentation
□ Plan user linking strategy
□ Design token management
□ Plan session handling

Development:
□ Create SSO tables
□ Implement SsoProvider class
□ Add OAuth2 authorization
□ Add OAuth2 callback handling
□ Create user linking logic
□ Implement token management
□ Add session handling
□ Create admin interface
□ Build provider management
□ Add logging

Testing:
□ Test OAuth2 flow
□ Test user linking
□ Test session creation
□ Test token validation
□ Test logout
□ Test with multiple providers
□ Verify security
□ Test error handling
```