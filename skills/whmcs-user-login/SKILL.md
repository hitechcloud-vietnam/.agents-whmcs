# WHMCS User Login

## Overview
Guide for customizing user authentication and login flows in WHMCS. Covers login forms, SSO integration, and security features.

## Login Hooks

### Pre-Login Validation

```php
<?php
// /includes/hooks/user_login.php

add_hook("PreLogin", 1, function(array $params) {
    $email = $params["username"];
    
    // Check if account exists
    $client = Capsule::table("tblclients")
        ->where("email", $email)
        ->first();
    
    if (!$client) {
        return ["error" => "Invalid email or password"];
    }
    
    // Check if account is blocked
    if ($client->status === "Closed") {
        return ["error" => "This account has been closed"];
    }
    
    if ($client->status === "Inactive") {
        return ["error" => "Please verify your email first"];
    }
    
    // Check login restrictions
    $blocked = Capsule::table("mod_blocked_ips")
        ->where("ip", $_SERVER["REMOTE_ADDR"])
        ->first();
    
    if ($blocked) {
        return ["error" => "Access denied from this location"];
    }
    
    return ["success" => true];
});
```

### Post-Login Actions

```php
add_hook("ClientLogin", 1, function(array $params) {
    $clientId = $params["userid"];
    
    // Update last login
    Capsule::table("tblclients")
        ->where("id", $clientId)
        ->update([
            "lastlogin" => date("Y-m-d H:i:s"),
            "lastloginip" => $_SERVER["REMOTE_ADDR"]
        ]);
    
    // Record login history
    Capsule::table("mod_login_history")->insert([
        "client_id" => $clientId,
        "ip_address" => $_SERVER["REMOTE_ADDR"],
        "user_agent" => $_SERVER["HTTP_USER_AGENT"] ?? "",
        "login_time" => date("Y-m-d H:i:s"),
        "success" => 1
    ]);
    
    // Check for pending actions
    $pendingTickets = Capsule::table("tbltickets")
        ->where("userid", $clientId)
        ->whereIn("status", ["Open", "Customer Reply"])
        ->count();
    
    if ($pendingTickets > 0) {
        set_session_var("pending_tickets", $pendingTickets);
    }
    
    return $params;
});
```

### Failed Login Tracking

```php
add_hook("FailedLoginAttempt", 1, function(array $params) {
    $ip = $_SERVER["REMOTE_ADDR"];
    
    // Record failed attempt
    Capsule::table("mod_login_attempts")->insert([
        "ip_address" => $ip,
        "email" => $params["username"],
        "attempt_time" => date("Y-m-d H:i:s"),
        "user_agent" => $_SERVER["HTTP_USER_AGENT"] ?? ""
    ]);
    
    // Count recent failures
    $recentFailures = Capsule::table("mod_login_attempts")
        ->where("ip_address", $ip)
        ->where("attempt_time", ">", date("Y-m-d H:i:s", strtotime("-15 minutes")))
        ->count();
    
    // Block after 5 failures
    if ($recentFailures >= 5) {
        Capsule::table("mod_blocked_ips")->insert([
            "ip_address" => $ip,
            "blocked_until" => date("Y-m-d H:i:s", strtotime("+30 minutes")),
            "reason" => "Too many failed login attempts"
        ]);
        
        return ["error" => "Too many failed attempts. Please try again later."];
    }
    
    return $params;
});
```

## Login Form Template

```smarty
<!-- /templates/client_login.tpl -->
<div class="login-container">
    <form method="post" action="dologin.php" class="login-form">
        <input type="hidden" name="token" value="{$token}">
        
        <div class="form-header">
            <h2>Welcome Back</h2>
            <p>Sign in to your account</p>
        </div>
        
        {if $loginError}
            <div class="alert alert-danger">
                {$loginError}
            </div>
        {/if}
        
        <div class="form-group">
            <label for="email">Email Address</label>
            <input type="email" name="username" id="email" 
                   value="{$username}" required autofocus
                   class="form-control"
                   placeholder="your@email.com">
        </div>
        
        <div class="form-group">
            <label for="password">
                Password
                <a href="password-reminder.php" class="forgot-link">
                    Forgot password?
                </a>
            </label>
            <div class="password-input-wrapper">
                <input type="password" name="password" id="password" 
                       required class="form-control"
                       placeholder="Enter your password">
                <button type="button" class="toggle-password" 
                        aria-label="Show password">
                    <i class="fa fa-eye"></i>
                </button>
            </div>
        </div>
        
        <div class="form-group captcha">
            {if $captcha}
                <div class="captcha-container">
                    <img src="includes/classes/reCAPTCHA.php?captcha=true" 
                         alt="CAPTCHA">
                    <input type="text" name="captcha" required>
                </div>
            {/if}
        </div>
        
        <div class="form-group checkbox">
            <label>
                <input type="checkbox" name="rememberme" value="1">
                Remember me on this device
            </label>
        </div>
        
        <button type="submit" class="btn btn-primary btn-block">
            Sign In
        </button>
        
        <div class="social-login">
            <p>Or sign in with</p>
            <div class="social-buttons">
                <a href="oauth.php?provider=google" class="btn btn-social btn-google">
                    <i class="fa fa-google"></i> Google
                </a>
                <a href="oauth.php?provider=microsoft" class="btn btn-social btn-microsoft">
                    <i class="fa fa-microsoft"></i> Microsoft
                </a>
            </div>
        </div>
        
        <div class="login-footer">
            <p>Don't have an account? 
               <a href="register.php">Create one now</a>
            </p>
        </div>
    </form>
</div>

<script>
document.querySelector('.toggle-password').addEventListener('click', function() {
    const input = document.getElementById('password');
    const icon = this.querySelector('i');
    
    if (input.type === 'password') {
        input.type = 'text';
        icon.classList.replace('fa-eye', 'fa-eye-slash');
    } else {
        input.type = 'password';
        icon.classList.replace('fa-eye-slash', 'fa-eye');
    }
});
</script>
```

## SSO Integration

```php
<?php
// Single Sign-On implementation
add_hook("ClientAreaPrimarySidebar", 1, function(array $params) {
    if (isLoggedIn()) {
        $userId = $_SESSION["uid"];
        
        // Generate SSO token for integrated apps
        $ssoToken = generateSSOToken($userId);
        
        $params["primarySidebar"]->addItem(
            "Quick Access",
            '<div class="sso-quick-access">
                <a href="app1.php?sso=' . $ssoToken . '">App 1</a>
                <a href="app2.php?sso=' . $ssoToken . '">App 2</a>
             </div>'
        );
    }
    return $params;
});

function generateSSOToken(int $userId): string
{
    $payload = [
        "user_id" => $userId,
        "exp" => time() + 3600,
        "ip" => $_SERVER["REMOTE_ADDR"]
    ];
    
    $key = WHMCS\Config\Setting::getValue("SSO_SECRET_KEY");
    
    return base64_encode(json_encode($payload)) . "." .
           hash_hmac("sha256", json_encode($payload), $key);
}
```

## Two-Factor Authentication

```php
add_hook("ClientLogin", 1, function(array $params) {
    $clientId = $params["userid"];
    
    $twoFactorEnabled = Capsule::table("tblclients")
        ->where("id", $clientId)
        ->value("twofactorenabled");
    
    if ($twoFactorEnabled) {
        // Generate 2FA code and redirect
        $code = generate2FACode($clientId);
        send2FACode($clientId, $code);
        
        redirect("security-verification.php?step=2fa");
    }
    
    return $params;
});

function generate2FACode(int $clientId): string
{
    $code = str_pad((string)random_int(0, 999999), 6, "0", STR_PAD_LEFT);
    
    Capsule::table("mod_2fa_codes")->insert([
        "client_id" => $clientId,
        "code" => password_hash($code, PASSWORD_DEFAULT),
        "expires_at" => date("Y-m-d H:i:s", strtotime("+5 minutes")),
        "created_at" => date("Y-m-d H:i:s")
    ]);
    
    return $code;
}
```

## Best Practices

1. **Rate Limiting**: Limit login attempts per IP/email
2. **Password Visibility**: Toggle for password field
3. **Remember Me**: Secure persistent sessions
4. **Failed Attempt Tracking**: Monitor and block brute force
5. **2FA Support**: Encourage two-factor authentication
6. **Session Management**: Secure session handling
7. **SSO Options**: Integrate with common identity providers
8. **Audit Logging**: Track all login attempts
