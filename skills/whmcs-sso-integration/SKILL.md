# WHMCS SSO Integration Skill
# Version: 1.0 | Updated: 2026-05-28

## Purpose

Guide for implementing Single Sign-On in WHMCS modules.

## When to Use

- Connecting to identity providers
- Building SSO portals
- Implementing OAuth/OpenID Connect

## SSO Patterns

### OAuth 2.0 Integration
```php
<?php
class OAuthProvider {
    private string $clientId;
    private string $clientSecret;
    private string $redirectUri;
    private string $authUrl;
    private string $tokenUrl;

    public function getAuthorizationUrl(): string {
        $_SESSION['oauth_state'] = bin2hex(random_bytes(16));

        $params = [
            'client_id' => $this->clientId,
            'redirect_uri' => $this->redirectUri,
            'response_type' => 'code',
            'scope' => 'openid profile email',
            'state' => $_SESSION['oauth_state'],
        ];

        return $this->authUrl . '?' . http_build_query($params);
    }

    public function handleCallback(string $code, string $state): array {
        if ($state !== $_SESSION['oauth_state']) {
            throw new \Exception('Invalid state parameter');
        }

        $token = $this->exchangeCode($code);
        $userInfo = $this->getUserInfo($token['access_token']);

        return $this->loginOrCreateUser($userInfo);
    }

    private function exchangeCode(string $code): array {
        $ch = curl_init($this->tokenUrl);

        curl_setopt_array($ch, [
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

        return json_decode(curl_exec($ch), true);
    }
}
```

### SSO Hook
```php
add_hook('UserLoginSSO', 1, function($vars) {
    $oauth = new OAuthProvider();
    $user = $oauth->handleCallback(
        $_GET['code'],
        $_GET['state']
    );

    return [
        'user_id' => $user['id'],
        'logged_in' => true,
    ];
});
```

---

**Related Skills:**
- whmcs-security-hardening
- whmcs-hooks-development
- whmcs-clientarea-builder
