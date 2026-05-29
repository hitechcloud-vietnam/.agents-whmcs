# WHMCS User Security Settings

## Overview
Guide for implementing user security settings in WHMCS. Covers security features, 2FA management, and login monitoring.

## Security Settings

### Password Security

```php
<?php
// /includes/hooks/user_security.php

function validatePasswordStrength(string $password): array
{
    $errors = [];
    
    if (strlen($password) < 8) {
        $errors[] = "Password must be at least 8 characters";
    }
    
    if (!preg_match("/[A-Z]/", $password)) {
        $errors[] = "Password must contain at least one uppercase letter";
    }
    
    if (!preg_match("/[a-z]/", $password)) {
        $errors[] = "Password must contain at least one lowercase letter";
    }
    
    if (!preg_match("/[0-9]/", $password)) {
        $errors[] = "Password must contain at least one number";
    }
    
    if (!preg_match("/[^A-Za-z0-9]/", $password)) {
        $errors[] = "Password must contain at least one special character";
    }
    
    return $errors;
}

add_hook("ValidatePassword", 1, function(array $params) {
    $errors = validatePasswordStrength($params["password"]);
    
    if (!empty($errors)) {
        return ["error" => implode(". ", $errors)];
    }
    
    // Check against common passwords
    $commonPasswords = file(__DIR__ . "/common_passwords.txt", FILE_IGNORE_NEW_LINES);
    if (in_array(strtolower($params["password"]), array_map("strtolower", $commonPasswords))) {
        return ["error" => "This password is too common. Please choose a more secure password."];
    }
    
    // Check against breached passwords (HaveIBeenPwned API)
    if (checkPasswordBreached($params["password"])) {
        return ["error" => "This password has appeared in a data breach. Please choose a different password."];
    }
    
    return ["success" => true];
});

function checkPasswordBreached(string $password): bool
{
    $hash = strtoupper(sha1($password));
    $prefix = substr($hash, 0, 5);
    $suffix = substr($hash, 5);
    
    $ch = curl_init("https://api.pwnedpasswords.com/range/" . $prefix);
    curl_setopt_array($ch, [
        CURLOPT_RETURNTRANSFER => true,
        CURLOPT_TIMEOUT => 10
    ]);
    $response = curl_exec($ch);
    curl_close($ch);
    
    if (stripos($response, $suffix) !== false) {
        return true;
    }
    
    return false;
}
```

### Two-Factor Authentication

```php
function enableTwoFactor(int $userId, string $method, array $data): bool
{
    $code = generate2FACode();
    
    switch ($method) {
        case "totp":
            // Store TOTP secret
            Capsule::table("mod_user_2fa")->insert([
                "user_id" => $userId,
                "method" => "totp",
                "data" => encrypt($data["secret"]),
                "enabled_at" => date("Y-m-d H:i:s")
            ]);
            break;
            
        case "email":
            // Send verification code
            send_email("TwoFactorSetup", $userId, [
                "code" => $code,
                "expires" => "10 minutes"
            ]);
            // Store hashed code temporarily
            Capsule::table("mod_2fa_pending")->insert([
                "user_id" => $userId,
                "method" => "email",
                "code_hash" => password_hash($code, PASSWORD_DEFAULT),
                "expires_at" => date("Y-m-d H:i:s", strtotime("+10 minutes"))
            ]);
            return false; // Requires verification
            
        case "sms":
            $phone = formatPhoneNumber($data["phone"]);
            sendSMS($phone, "Your 2FA code is: {$code}");
            Capsule::table("mod_2fa_pending")->insert([
                "user_id" => $userId,
                "method" => "sms",
                "phone" => $phone,
                "code_hash" => password_hash($code, PASSWORD_DEFAULT),
                "expires_at" => date("Y-m-d H:i:s", strtotime("+10 minutes"))
            ]);
            return false;
    }
    
    // Enable 2FA
    Capsule::table("tblclients")
        ->where("id", $userId)
        ->update(["twofactorenabled" => 1]);
    
    logUserActivity($userId, "2fa_enabled", "security");
    
    return true;
}

function verifyTwoFactor(int $userId, string $code): bool
{
    $twoFactor = Capsule::table("mod_user_2fa")
        ->where("user_id", $userId)
        ->first();
    
    if (!$twoFactor) {
        return false;
    }
    
    switch ($twoFactor->method) {
        case "totp":
            $secret = decrypt($twoFactor->data);
            return verifyTOTP($secret, $code);
            
        case "backup":
            $backups = json_decode(decrypt($twoFactor->data), true);
            $codeHash = sha1($code);
            
            foreach ($backups as $index => $hash) {
                if ($hash === $codeHash) {
                    // Remove used backup code
                    unset($backups[$index]);
                    Capsule::table("mod_user_2fa")
                        ->where("id", $twoFactor->id)
                        ->update(["data" => encrypt(json_encode($backups))]);
                    return true;
                }
            }
            return false;
    }
    
    return false;
}

function disableTwoFactor(int $userId, string $password): bool
{
    // Verify password first
    $client = Capsule::table("tblclients")->where("id", $userId)->first();
    if (!password_verify($password, $client->password)) {
        return false;
    }
    
    Capsule::table("mod_user_2fa")
        ->where("user_id", $userId)
        ->delete();
    
    Capsule::table("tblclients")
        ->where("id", $userId)
        ->update(["twofactorenabled" => 0]);
    
    logUserActivity($userId, "2fa_disabled", "security");
    
    return true;
}
```

### Login Monitoring

```php
function getLoginHistory(int $userId, int $limit = 10): array
{
    return Capsule::table("mod_login_history")
        ->where("user_id", $userId)
        ->orderBy("login_time", "desc")
        ->limit($limit)
        ->get();
}

function getActiveSessions(int $userId): array
{
    return Capsule::table("mod_user_sessions")
        ->where("user_id", $userId)
        ->where("expires_at", ">", date("Y-m-d H:i:s"))
        ->where("active", 1)
        ->get();
}

function terminateSession(int $userId, int $sessionId): bool
{
    Capsule::table("mod_user_sessions")
        ->where("id", $sessionId)
        ->where("user_id", $userId)
        ->update([
            "terminated_at" => date("Y-m-d H:i:s"),
            "terminated_by" => $userId,
            "active" => 0
        ]);
    
    return true;
}

function terminateAllSessions(int $userId): int
{
    return Capsule::table("mod_user_sessions")
        ->where("user_id", $userId)
        ->where("active", 1)
        ->update([
            "terminated_at" => date("Y-m-d H:i:s"),
            "terminated_by" => $userId,
            "active" => 0
        ]);
}
```

## Security Settings Template

```smarty
<!-- /templates/clientarea_security.tpl -->
<div class="security-settings">
    <h2>Security Settings</h2>
    
    <!-- Password Section -->
    <section class="security-section">
        <h3>Change Password</h3>
        <form method="post" action="clientarea.php?action=security">
            <input type="hidden" name="token" value="{$token}">
            <input type="hidden" name="section" value="password">
            
            <div class="form-group">
                <label>Current Password</label>
                <input type="password" name="current_password" required>
            </div>
            <div class="form-group">
                <label>New Password</label>
                <input type="password" name="new_password" required minlength="8">
                <div class="password-strength"></div>
            </div>
            <div class="form-group">
                <label>Confirm New Password</label>
                <input type="password" name="confirm_password" required>
            </div>
            
            <button type="submit" class="btn btn-primary">Update Password</button>
        </form>
    </section>
    
    <!-- 2FA Section -->
    <section class="security-section">
        <h3>Two-Factor Authentication</h3>
        {if $two_factor_enabled}
            <div class="2fa-status enabled">
                <i class="fa fa-shield"></i>
                <span>2FA is enabled ({$two_factor_method})</span>
                <a href="?action=disable_2fa" class="btn btn-sm btn-danger">
                    Disable 2FA
                </a>
            </div>
        {else}
            <div class="2fa-status disabled">
                <p>Protect your account with two-factor authentication.</p>
                <a href="setup-2fa.php" class="btn btn-primary">
                    Enable 2FA
                </a>
            </div>
        {/if}
    </section>
    
    <!-- Active Sessions -->
    <section class="security-section">
        <h3>Active Sessions</h3>
        <div class="sessions-list">
            {foreach $active_sessions as $session}
                <div class="session-item">
                    <div class="session-info">
                        <i class="fa fa-desktop"></i>
                        <div>
                            <strong>{$session->user_agent}</strong>
                            <small>IP: {$session->ip_address}</small>
                            <small>Last active: {$session->last_activity}</small>
                        </div>
                    </div>
                    <a href="?terminate_session={$session->id}" class="btn btn-sm">
                        Terminate
                    </a>
                </div>
            {/foreach}
        </div>
        <a href="?terminate_all=1" class="btn btn-outline">
            Terminate All Other Sessions
        </a>
    </section>
    
    <!-- Login History -->
    <section class="security-section">
        <h3>Recent Login Activity</h3>
        <table class="login-history">
            <thead>
                <tr>
                    <th>Date</th>
                    <th>IP Address</th>
                    <th>Location</th>
                    <th>Status</th>
                </tr>
            </thead>
            <tbody>
                {foreach $login_history as $login}
                    <tr class="{if $login->success}success{else}danger{/if}">
                        <td>{$login->login_time}</td>
                        <td>{$login->ip_address}</td>
                        <td>{$login->location ?? 'Unknown'}</td>
                        <td>{$login->success ? 'Success' : 'Failed'}</td>
                    </tr>
                {/foreach}
            </tbody>
        </table>
    </section>
</div>
```

## Best Practices

1. **Password Policy**: Enforce strong password requirements
2. **Breach Checking**: Check passwords against known breaches
3. **2FA Support**: Offer multiple 2FA methods
4. **Session Management**: Allow viewing and terminating sessions
5. **Login Alerts**: Notify of new login locations
6. **Backup Codes**: Provide backup codes for 2FA
7. **Password History**: Prevent password reuse
8. **Audit Trail**: Log all security-related events
