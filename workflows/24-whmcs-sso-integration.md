# WHMCS SSO Integration Workflow

## Overview
This workflow covers implementing Single Sign-On (SSO) for seamless authentication across systems.

## Step 1: SSO Service

```php
<?php
// src/Service/SsoService.php

namespace WHMCS\Module\Addon\YourModule\Service;

use WHMCS\Database\Capsule;

class SsoService
{
    private $ssoSecret;
    private $ssoUrl;

    public function __construct(string $ssoSecret, string $ssoUrl)
    {
        $this->ssoSecret = $ssoSecret;
        $this->ssoUrl = rtrim($ssoUrl, '/');
    }

    public function generateSsoToken(int $clientId, int $expiresIn = 3600): string
    {
        $client = Capsule::table('tblclients')->where('id', $clientId)->first();

        if (!$client) {
            throw new \Exception("Client not found");
        }

        $payload = [
            'iss' => 'whmcs',
            'sub' => $clientId,
            'email' => $client->email,
            'name' => $client->firstname . ' ' . $client->lastname,
            'iat' => time(),
            'exp' => time() + $expiresIn,
            'nonce' => bin2hex(random_bytes(16))
        ];

        $header = $this->base64UrlEncode(json_encode(['alg' => 'HS256', 'typ' => 'JWT']));
        $payload = $this->base64UrlEncode(json_encode($payload));
        $signature = $this->base64UrlEncode(
            hash_hmac('sha256', "$header.$payload", $this->ssoSecret, true)
        );

        return "$header.$payload.$signature";
    }

    public function validateSsoToken(string $token): ?array
    {
        $parts = explode('.', $token);
        if (count($parts) !== 3) {
            return null;
        }

        [$header, $payload, $signature] = $parts;

        // Verify signature
        $expectedSignature = $this->base64UrlEncode(
            hash_hmac('sha256', "$header.$payload", $this->ssoSecret, true)
        );

        if (!hash_equals($expectedSignature, $signature)) {
            return null;
        }

        // Decode payload
        $data = json_decode($this->base64UrlDecode($payload), true);

        // Check expiration
        if ($data['exp'] < time()) {
            return null;
        }

        return $data;
    }

    public function createRedirectUrl(int $clientId, string $destination = ''): string
    {
        $token = $this->generateSsoToken($clientId);

        $params = [
            'token' => $token,
            'redirect' => $destination
        ];

        return $this->ssoUrl . '/sso/login?' . http_build_query($params);
    }

    public function logout(int $clientId): void
    {
        // Invalidate any active sessions
        Capsule::table('mod_sso_sessions')
            ->where('client_id', $clientId)
            ->update(['invalidated_at' => date('Y-m-d H:i:s')]);
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
```

## Step 2: SSO Hook

```php
<?php
// includes/hooks/sso_hook.php

add_hook('ClientAreaPageLogin', 1, function($params) {
    // Check for SSO token in URL
    if (isset($_GET['sso_token'])) {
        $ssoService = new \WHMCS\Module\Addon\YourModule\Service\SsoService(
            Capsule::config('sso_secret'),
            Capsule::config('sso_url')
        );

        $tokenData = $ssoService->validateSsoToken($_GET['sso_token']);

        if ($tokenData) {
            // Log in the user
            $_SESSION['uid'] = $tokenData['sub'];
            $_SESSION['upw'] = md5($tokenData['sub'] . session_id());

            // Redirect to destination
            if (isset($_GET['redirect'])) {
                header('Location: ' . $_GET['redirect']);
                exit;
            }
        }
    }

    return $params;
});
```

## Verification Checklist

- [ ] SSO service implemented
- [ ] Token generation working
- [ ] Token validation working
- [ ] SSO hooks registered
- [ ] Logout functionality working
- [ ] Test SSO login successful
