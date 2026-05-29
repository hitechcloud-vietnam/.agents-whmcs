# WHMCS Admin Two-Factor Authentication

## Overview
Guide for implementing two-factor authentication for WHMCS admin users. Covers 2FA setup, verification, and recovery.

## Two-Factor Authentication

### Enable 2FA

```php
<?php
// /includes/hooks/admin_two_factor.php

add_hook("AdminEnableTwoFactor", 1, function(array $params) {
    $adminId = $params["admin_id"];
    $method = $params["method"];
    $data = $params["data"];
    
    switch ($method) {
        case "totp":
            // Verify TOTP setup first
            if (!verifyTOTPCode($data["secret"], $data["code"])) {
                return ["error" => "Invalid verification code"];
            }
            
            Capsule::table("mod_admin_2fa")->insert([
                "admin_id" => $adminId,
                "method" => "totp",
                "data" => encrypt($data["secret"]),
                "enabled_at" => date("Y-m-d H:i:s")
            ]);
            break;
            
        case "email":
            // Send verification email
            $code = generate2FACode();
            sendAdmin2FACode($adminId, $code);
            
            Capsule::table("mod_admin_2fa_pending")->insert([
                "admin_id" => $adminId,
                "method" => "email",
                "code_hash" => password_hash($code, PASSWORD_DEFAULT),
                "expires_at" => date("Y-m-d H:i:s", strtotime("+10 minutes"))
            ]);
            
            return ["pending_verification" => true];
    }
    
    // Enable 2FA on admin
    Capsule::table("tbladmins")
        ->where("id", $adminId)
        ->update(["twofa" => 1]);
    
    // Generate backup codes
    $backupCodes = generateBackupCodes();
    storeBackupCodes($adminId, $backupCodes);
    
    logAdminActivity("2fa_enabled", $adminId);
    
    return [
        "success" => true,
        "backup_codes" => $backupCodes
    ];
});

function verifyTOTPCode(string $secret, string $code): bool
{
    require_once __DIR__ . "/lib/TOTP.php";
    
    $totp = new TOTP($secret);
    return $totp->verify($code, 1); // 1 = allow 1 period drift
}

function generateBackupCodes(): array
{
    $codes = [];
    for ($i = 0; $i < 10; $i++) {
        $codes[] = bin2hex(random_bytes(4)) . "-" . bin2hex(random_bytes(4));
    }
    return $codes;
}

function storeBackupCodes(int $adminId, array $codes): void
{
    $hashedCodes = [];
    foreach ($codes as $code) {
        $hashedCodes[] = sha1($code);
    }
    
    Capsule::table("mod_admin_2fa")->insert([
        "admin_id" => $adminId,
        "method" => "backup",
        "data" => encrypt(json_encode($hashedCodes))
    ]);
}
```

### Verify 2FA

```php
add_hook("AdminVerifyTwoFactor", 1, function(array $params) {
    $adminId = $params["admin_id"];
    $code = $params["code"];
    
    $twoFactor = Capsule::table("mod_admin_2fa")
        ->where("admin_id", $adminId)
        ->where("method", "!=", "backup")
        ->first();
    
    if (!$twoFactor) {
        return ["error" => "2FA not enabled"];
    }
    
    switch ($twoFactor->method) {
        case "totp":
            $secret = decrypt($twoFactor->data);
            if (verifyTOTPCode($secret, $code)) {
                return ["success" => true];
            }
            break;
            
        case "backup":
            $backupCodes = json_decode(decrypt($twoFactor->data), true);
            $codeHash = sha1(strtoupper($code));
            
            $index = array_search($codeHash, $backupCodes);
            if ($index !== false) {
                unset($backupCodes[$index]);
                Capsule::table("mod_admin_2fa")
                    ->where("admin_id", $adminId)
                    ->where("method", "backup")
                    ->update(["data" => encrypt(json_encode(array_values($backupCodes)))]);
                
                return ["success" => true];
            }
            break;
    }
    
    return ["error" => "Invalid verification code"];
});
```

### Disable 2FA

```php
add_hook("AdminDisableTwoFactor", 1, function(array $params) {
    $adminId = $params["admin_id"];
    $password = $params["password"];
    
    // Verify password
    $admin = Capsule::table("tbladmins")->where("id", $adminId)->first();
    if (!password_verify($password, $admin->password)) {
        return ["error" => "Invalid password"];
    }
    
    // Remove 2FA
    Capsule::table("mod_admin_2fa")
        ->where("admin_id", $adminId)
        ->delete();
    
    Capsule::table("tbladmins")
        ->where("id", $adminId)
        ->update(["twofa" => 0]);
    
    logAdminActivity("2fa_disabled", $adminId);
    
    return ["success" => true];
});
```

## 2FA Setup Template

```smarty
<!-- /admin/templates/admin_2fa_setup.tpl -->
<div class="two-factor-setup">
    <h2>Two-Factor Authentication</h2>
    
    {if !$twofa_enabled}
        <div class="setup-steps">
            <h3>Step 1: Choose Method</h3>
            <div class="method-options">
                <div class="method-card" data-method="totp">
                    <i class="fa fa-mobile-phone"></i>
                    <h4>Authenticator App</h4>
                    <p>Use Google Authenticator or similar app</p>
                </div>
                <div class="method-card" data-method="email">
                    <i class="fa fa-envelope"></i>
                    <h4>Email</h4>
                    <p>Receive code via email</p>
                </div>
            </div>
            
            <div class="totp-setup" style="display:none;">
                <h3>Step 2: Scan QR Code</h3>
                <div class="qr-code">
                    <img src="{$qr_code_url}" alt="QR Code">
                </div>
                <p class="secret-key">Secret Key: <code>{$secret_key}</code></p>
                
                <h3>Step 3: Verify</h3>
                <form method="post">
                    <input type="hidden" name="token" value="{$token}">
                    <input type="hidden" name="action" value="verify_totp">
                    <input type="hidden" name="secret" value="{$secret_key}">
                    
                    <div class="form-group">
                        <label>Enter code from app</label>
                        <input type="text" name="code" class="form-control" 
                               maxlength="6" required autofocus>
                    </div>
                    
                    <button type="submit" class="btn btn-primary">
                        Verify & Enable
                    </button>
                </form>
            </div>
        </div>
    {else}
        <div class="two-factor-enabled">
            <div class="status-badge">
                <i class="fa fa-shield"></i>
                2FA is enabled
            </div>
            
            <h3>Backup Codes</h3>
            <p>Save these codes in a safe place. Each code can only be used once.</p>
            <div class="backup-codes">
                {foreach $backup_codes as $code}
                    <code>{$code}</code>
                {/foreach}
            </div>
            
            <div class="actions">
                <a href="generate_backup_codes.php" class="btn btn-default">
                    Regenerate Backup Codes
                </a>
                <a href="disable_2fa.php" class="btn btn-danger">
                    Disable 2FA
                </a>
            </div>
        </div>
    {/if}
</div>
```

## Best Practices

1. **Multiple Methods**: Support multiple 2FA methods
2. **Backup Codes**: Provide backup codes for account recovery
3. **Recovery Process**: Secure recovery process
4. **User Education**: Guide users through setup
5. **QR Code**: Generate scannable QR codes
6. **Remember Device**: Option to remember trusted devices
7. **Grace Period**: Optional grace period before enforcing
8. **Audit Trail**: Log all 2FA changes
