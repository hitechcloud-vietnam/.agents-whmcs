# WHMCS Client Authentication

## Overview
Master skill for client authentication in WHMCS. Covers login, two-factor authentication, SSO, and security.

## Authentication Hooks

```php
<?php
// /includes/hooks/authentication_hooks.php

add_hook("ClientLogin", 1, function(array $params) {
    $clientId = $params["userid"];
    $ip = $_SERVER["REMOTE_ADDR"];
    
    logActivity("Client login: User ID {$clientId} from {$ip}");
    
    updateLastLogin($clientId);
    
    checkLoginNotifications($clientId);
    
    return true;
});

add_hook("CustomLoginValidation", 1, function(array $params) {
    $email = $params["email"];
    $ip = $_SERVER["REMOTE_ADDR"];
    
    if (isBruteForceAttack($ip)) {
        return ["success" => false, "error" => "Too many login attempts"];
    }
    
    if (isSuspiciousLogin($email, $ip)) {
        sendLoginAlert($email, $ip);
    }
    
    return ["success" => true];
});

add_hook("TwoFactorAuthSetup", 1, function(array $params) {
    $secret = generateTOTPSecret();
    
    return [
        "secret" => $secret,
        "qr_uri" => generateTOTPUri($secret, $params["email"]),
    ];
});

function isBruteForceAttack(string $ip): bool
{
    $attempts = \Illuminate\Database\Capsule\Manager::table("mod_login_attempts")
        ->where("ip", $ip)
        ->where("created_at", ">", date("Y-m-d H:i:s", strtotime("-15 minutes")))
        ->count();
    
    return $attempts >= 5;
}

function isSuspiciousLogin(string $email, string $ip): bool
{
    $lastLogin = \Illuminate\Database\Capsule\Manager::table("tblactivitylog")
        ->where("description", "like", "%Client Login%")
        ->orderBy("id", "desc")
        ->first();
    
    if (!$lastLogin) {
        return false;
    }
    
    $lastIp = $lastLogin->ipaddr ?? "";
    return $lastIp !== "" && $lastIp !== $ip;
}
```

## Two-Factor Authentication

```php
<?php
// /includes/security/two_factor.php

class TwoFactorAuth
{
    public static function generateSecret(): string
    {
        return bin2hex(random_bytes(20));
    }
    
    public static function getQRCodeUri(string $secret, string $email): string
    {
        $encoded = urlencode("otpauth://totp/{$email}?secret={$secret}&issuer=WHMCS");
        return "https://api.qrserver.com/v1/create-qr-code/?size=200x200&data={$encoded}";
    }
    
    public static function verify(string $secret, string $code): bool
    {
        $timeSlice = floor(time() / 30);
        
        for ($i = -1; $i <= 1; $i++) {
            if (hash_equals(self::getCode($secret, $timeSlice + $i), $code)) {
                return true;
            }
        }
        
        return false;
    }
    
    private static function getCode(string $secret, int $timeSlice): string
    {
        $decoded = base32_decode($secret);
        $time = pack("N", $timeSlice);
        $hash = hash_hmac("sha1", $time, $decoded, true);
        
        return str_pad(
            (ord($hash[19]) & 0xf) << 24 |
            (ord($hash[18]) & 0xff) << 16 |
            (ord($hash[17]) & 0xff) << 8 |
            (ord($hash[16]) & 0xff),
            6, "0", STR_PAD_LEFT
        );
    }
}
```

## Best Practices

1. **Password Requirements**: Enforce strong passwords
2. **Two-Factor**: Enable 2FA for security
3. **Brute Force**: Implement login attempt limits
4. **Sessions**: Use secure session management
5. **Logging**: Track all authentication events
6. **Password Reset**: Secure password recovery
7. **Remember Me**: Implement secure remember me
8. **SSO**: Support SSO where appropriate
