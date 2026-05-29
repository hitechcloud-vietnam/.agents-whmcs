# WHMCS User Password Reset

## Overview
Guide for implementing secure password reset functionality in WHMCS. Covers email-based reset, token validation, and security best practices.

## Password Reset Flow

### Request Reset Hook

```php
<?php
// /includes/hooks/password_reset.php

add_hook("PasswordResetRequest", 1, function(array $params) {
    $email = $params["email"];
    
    // Check rate limiting
    $recentRequests = Capsule::table("mod_password_reset_tokens")
        ->where("email", $email)
        ->where("created_at", ">", date("Y-m-d H:i:s", strtotime("-1 hour")))
        ->count();
    
    if ($recentRequests >= 3) {
        return [
            "error" => "Too many reset requests. Please try again later."
        ];
    }
    
    // Check if email exists
    $client = Capsule::table("tblclients")
        ->where("email", $email)
        ->first();
    
    if (!$client) {
        // Don't reveal if email exists
        return [
            "success" => true,
            "message" => "If an account exists, a reset email has been sent."
        ];
    }
    
    // Invalidate existing tokens
    Capsule::table("mod_password_reset_tokens")
        ->where("client_id", $client->id)
        ->where("used", 0)
        ->update(["used" => 1]);
    
    // Generate secure token
    $token = bin2hex(random_bytes(32));
    $hashedToken = password_hash($token, PASSWORD_DEFAULT);
    
    // Store token
    Capsule::table("mod_password_reset_tokens")->insert([
        "client_id" => $client->id,
        "email" => $email,
        "token_hash" => $hashedToken,
        "ip_address" => $_SERVER["REMOTE_ADDR"],
        "created_at" => date("Y-m-d H:i:s"),
        "expires_at" => date("Y-m-d H:i:s", strtotime("+1 hour")),
        "used" => 0
    ]);
    
    // Send reset email
    $resetLink = "https://" . $_SERVER["HTTP_HOST"] . 
                 "/password-reset.php?token=" . $token . 
                 "&id=" . $client->id;
    
    send_email("PasswordResetVerification", $client->id, [
        "reset_link" => $resetLink,
        "client_name" => $client->firstname . " " . $client->lastname,
        "ip_address" => $_SERVER["REMOTE_ADDR"],
        "expires_in" => "1 hour"
    ]);
    
    return [
        "success" => true,
        "message" => "Password reset email sent."
    ];
});
```

### Token Validation Hook

```php
add_hook("ValidatePasswordResetToken", 1, function(array $params) {
    $token = $params["token"];
    $clientId = (int)$params["client_id"];
    
    // Find valid token
    $resetRecord = Capsule::table("mod_password_reset_tokens")
        ->where("client_id", $clientId)
        ->where("token_hash", "!=", "") // Legacy compatibility
        ->where("used", 0)
        ->where("expires_at", ">", date("Y-m-d H:i:s"))
        ->first();
    
    if (!$resetRecord) {
        return ["error" => "Invalid or expired reset link."];
    }
    
    // Verify token hash (legacy method)
    if (!password_verify($token, $resetRecord->token_hash)) {
        return ["error" => "Invalid reset token."];
    }
    
    return ["success" => true, "client_id" => $clientId];
});
```

### Password Update Hook

```php
add_hook("PasswordResetComplete", 1, function(array $params) {
    $clientId = $params["client_id"];
    
    // Mark token as used
    Capsule::table("mod_password_reset_tokens")
        ->where("client_id", $clientId)
        ->where("used", 0)
        ->update([
            "used" => 1,
            "used_at" => date("Y-m-d H:i:s")
        ]);
    
    // Invalidate all existing sessions
    Capsule::table("mod_user_sessions")
        ->where("client_id", $clientId)
        ->update(["invalidated_at" => date("Y-m-d H:i:s")]);
    
    // Log the password change
    Capsule::table("mod_security_log")->insert([
        "client_id" => $clientId,
        "action" => "password_reset",
        "ip_address" => $_SERVER["REMOTE_ADDR"],
        "user_agent" => $_SERVER["HTTP_USER_AGENT"] ?? "",
        "created_at" => date("Y-m-d H:i:s")
    ]);
    
    // Send confirmation email
    send_email("PasswordChangedNotification", $clientId, [
        "ip_address" => $_SERVER["REMOTE_ADDR"],
        "change_time" => date("Y-m-d H:i:s")
    ]);
    
    return $params;
});
```

## Password Reset Form Template

```smarty
<!-- /templates/password_reset_request.tpl -->
<div class="password-reset-container">
    <div class="reset-card">
        <h2>Reset Your Password</h2>
        <p>Enter your email address and we'll send you a link to reset your password.</p>
        
        {if $success}
            <div class="alert alert-success">
                <i class="fa fa-check-circle"></i>
                {$success}
            </div>
            <p>Please check your email for the reset link.</p>
            <a href="login.php" class="btn btn-link">Back to Login</a>
        {else}
            {if $error}
                <div class="alert alert-danger">
                    {$error}
                </div>
            {/if}
            
            <form method="post" action="password-reset.php" class="reset-form">
                <input type="hidden" name="token" value="{$token}">
                <input type="hidden" name="action" value="request">
                
                <div class="form-group">
                    <label for="email">Email Address</label>
                    <input type="email" name="email" id="email" 
                           required autofocus class="form-control"
                           placeholder="your@email.com">
                </div>
                
                <button type="submit" class="btn btn-primary btn-block">
                    <i class="fa fa-paper-plane"></i>
                    Send Reset Link
                </button>
            </form>
        {/if}
        
        <div class="reset-footer">
            <a href="login.php">
                <i class="fa fa-arrow-left"></i> Back to Login
            </a>
        </div>
    </div>
</div>
```

```smarty
<!-- /templates/password_reset_confirm.tpl -->
<div class="password-reset-container">
    <div class="reset-card">
        <h2>Set New Password</h2>
        <p>Create a new password for your account.</p>
        
        {if $error}
            <div class="alert alert-danger">{$error}</div>
        {/if}
        
        <form method="post" action="password-reset.php" class="reset-form">
            <input type="hidden" name="token" value="{$token}">
            <input type="hidden" name="client_id" value="{$client_id}">
            <input type="hidden" name="action" value="reset">
            
            <div class="form-group">
                <label for="password">New Password</label>
                <input type="password" name="password" id="password" 
                       required minlength="8" class="form-control"
                       placeholder="Enter new password">
                <div class="password-requirements">
                    <p>Password must contain:</p>
                    <ul>
                        <li data-requirement="length">At least 8 characters</li>
                        <li data-requirement="uppercase">One uppercase letter</li>
                        <li data-requirement="lowercase">One lowercase letter</li>
                        <li data-requirement="number">One number</li>
                        <li data-requirement="special">One special character</li>
                    </ul>
                </div>
            </div>
            
            <div class="form-group">
                <label for="confirm_password">Confirm New Password</label>
                <input type="password" name="confirm_password" 
                       id="confirm_password" required
                       equalTo="#password"
                       class="form-control"
                       placeholder="Confirm new password">
                <span class="password-match-status"></span>
            </div>
            
            <div class="password-strength-meter">
                <div class="strength-bar">
                    <div class="strength-fill" style="width: 0%"></div>
                </div>
                <span class="strength-text">Password Strength: <span></span></span>
            </div>
            
            <button type="submit" class="btn btn-primary btn-block">
                <i class="fa fa-lock"></i>
                Reset Password
            </button>
        </form>
    </div>
</div>

<script>
document.getElementById('password').addEventListener('input', function(e) {
    const password = e.target.value;
    const requirements = document.querySelectorAll('.password-requirements li');
    const strengthBar = document.querySelector('.strength-fill');
    const strengthText = document.querySelector('.strength-text span');
    
    let strength = 0;
    let requirementsMet = 0;
    
    const checks = {
        length: password.length >= 8,
        uppercase: /[A-Z]/.test(password),
        lowercase: /[a-z]/.test(password),
        number: /[0-9]/.test(password),
        special: /[^A-Za-z0-9]/.test(password)
    };
    
    requirements.forEach(req => {
        const reqType = req.dataset.requirement;
        if (checks[reqType]) {
            req.classList.add('met');
            req.classList.remove('unmet');
            requirementsMet++;
        } else {
            req.classList.add('unmet');
            req.classList.remove('met');
        }
    });
    
    strength = (requirementsMet / 5) * 100;
    strengthBar.style.width = strength + '%';
    
    const levels = ['Very Weak', 'Weak', 'Fair', 'Good', 'Strong'];
    strengthText.textContent = levels[Math.ceil(strength / 25) - 1] || 'Very Weak';
    
    if (strength >= 60) {
        strengthBar.className = 'strength-fill strong';
    } else if (strength >= 40) {
        strengthBar.className = 'strength-fill medium';
    } else {
        strengthBar.className = 'strength-fill weak';
    }
});
</script>
```

## Security Best Practices

1. **Token Generation**: Use cryptographically secure random tokens (random_bytes)
2. **Token Storage**: Hash tokens before storing in database
3. **Expiration**: Set reasonable token expiration (1 hour)
4. **Rate Limiting**: Limit reset requests per email/IP
5. **Email Confirmation**: Notify user of password change
6. **Session Invalidation**: Log out user from all devices
7. **Logging**: Track all password reset attempts
8. **Password Policy**: Enforce strong password requirements

## Database Schema

```php
// Migration for password reset tokens
use WHMCS\Database\Capsule;

Capsule::schema()->create('mod_password_reset_tokens', function($t) {
    $t->increments('id');
    $t->integer('client_id');
    $t->string('email', 255);
    $t->string('token_hash', 255);
    $t->string('ip_address', 45);
    $t->timestamp('created_at');
    $t->timestamp('expires_at');
    $t->tinyInteger('used')->default(0);
    $t->timestamp('used_at')->nullable();
    
    $t->index(['client_id', 'used']);
    $t->index(['expires_at']);
});
```
