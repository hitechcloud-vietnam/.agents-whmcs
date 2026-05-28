# WHMCS Two-Factor Authentication Skill
# Version: 1.0 | Updated: 2026-05-28

## Purpose

Guide for implementing 2FA in WHMCS modules.

## When to Use

- Adding extra security
- Building custom login modules
- OTP verification systems

## 2FA Patterns

```php
<?php
class TwoFactorAuth {
    public function generateSecret(): string {
        return bin2hex(random_bytes(20));
    }

    public function generateTOTP(string $secret): string {
        $time = floor(time() / 30);
        $hash = hash_hmac('sha1', pack('J*', $time), pack('H*', $secret));
        $offset = hexdec($hash[39]) & 0xf;
        $totp = (
            (hexdec($hash[$offset * 2]) << 24) |
            (hexdec($hash[$offset * 2 + 1]) << 16) |
            (hexdec($hash[$offset * 2 + 2]) << 8) |
            hexdec($hash[$offset * 2 + 3])
        ) & 0x7fffffff;
        return str_pad($totp % pow(10, 6), 6, '0', STR_PAD_LEFT);
    }

    public function verifyTOTP(string $secret, string $code): bool {
        $current = $this->generateTOTP($secret);
        $previous = $this->generateTOTP($secret, -1);
        $next = $this->generateTOTP($secret, 1);

        return $code === $current || $code === $previous || $code === $next;
    }

    public function generateBackupCodes(int $count = 10): array {
        $codes = [];
        for ($i = 0; $i < $count; $i++) {
            $codes[] = strtoupper(bin2hex(random_bytes(4)) . '-' . bin2hex(random_bytes(4)));
        }
        return $codes;
    }
}
```

### Hook Integration
```php
add_hook('UserLoginTwoFactor', 1, function($vars) {
    $userId = $vars['user_id'];
    $code = $vars['code'];

    $auth = new TwoFactorAuth();
    $secret = getUser2FASecret($userId);

    if (!$auth->verifyTOTP($secret, $code)) {
        return ['error' => 'Invalid verification code'];
    }

    return ['success' => true];
});
```

---

**Related Skills:**
- whmcs-security-hardening
- whmcs-clientarea-builder
- whmcs-hooks-development
