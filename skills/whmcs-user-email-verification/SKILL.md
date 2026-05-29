# WHMCS User Email Verification

## Overview
Guide for implementing email verification workflows in WHMCS. Covers verification email generation, token validation, and resend functionality.

## Email Verification Flow

### Generate Verification Token

```php
<?php
// /includes/hooks/email_verification.php

add_hook("ClientAdd", 1, function(array $params) {
    $clientId = $params["clientid"];
    
    // Generate verification token
    $token = bin2hex(random_bytes(32));
    
    // Store token
    Capsule::table("mod_email_verifications")->insert([
        "client_id" => $clientId,
        "email" => $params["email"],
        "token" => hash("sha256", $token),
        "created_at" => date("Y-m-d H:i:s"),
        "expires_at" => date("Y-m-d H:i:s", strtotime("+24 hours"))
    ]);
    
    // Build verification link
    $verifyLink = "https://" . $_SERVER["HTTP_HOST"] . 
                  "/verify-email.php?token=" . $token . 
                  "&id=" . $clientId;
    
    // Send verification email
    send_email("EmailVerification", $clientId, [
        "verification_link" => $verifyLink,
        "client_name" => $params["firstname"] . " " . $params["lastname"],
        "expires_in" => "24 hours"
    ]);
    
    return $params;
});
```

### Verify Email Hook

```php
add_hook("VerifyEmailToken", 1, function(array $params) {
    $token = $params["token"];
    $clientId = (int)$params["client_id"];
    
    // Find verification record
    $record = Capsule::table("mod_email_verifications")
        ->where("client_id", $clientId)
        ->where("token", hash("sha256", $token))
        ->where("verified", 0)
        ->first();
    
    if (!$record) {
        return ["error" => "Invalid verification link."];
    }
    
    if (strtotime($record->expires_at) < time()) {
        return ["error" => "Verification link has expired."];
    }
    
    // Mark as verified
    Capsule::table("mod_email_verifications")
        ->where("id", $record->id)
        ->update([
            "verified" => 1,
            "verified_at" => date("Y-m-d H:i:s"),
            "ip_address" => $_SERVER["REMOTE_ADDR"]
        ]);
    
    // Update client status if needed
    $client = Capsule::table("tblclients")->where("id", $clientId)->first();
    if ($client->status === "Inactive") {
        Capsule::table("tblclients")
            ->where("id", $clientId)
            ->update(["status" => "Active"]);
    }
    
    // Send confirmation
    send_email("EmailVerifiedConfirmation", $clientId, []);
    
    return ["success" => true, "client_id" => $clientId];
});
```

### Resend Verification Email

```php
add_hook("ResendEmailVerification", 1, function(array $params) {
    $clientId = $params["client_id"];
    
    // Check rate limit (max 3 per day)
    $todayCount = Capsule::table("mod_email_verifications")
        ->where("client_id", $clientId)
        ->where("resend_count", ">", 0)
        ->where("created_at", ">=", date("Y-m-d 00:00:00"))
        ->count();
    
    if ($todayCount >= 3) {
        return ["error" => "Maximum verification emails reached for today."];
    }
    
    // Get client email
    $client = Capsule::table("tblclients")->where("id", $clientId)->first();
    
    // Generate new token
    $token = bin2hex(random_bytes(32));
    
    // Invalidate old tokens
    Capsule::table("mod_email_verifications")
        ->where("client_id", $clientId)
        ->where("verified", 0)
        ->update(["verified" => 2]); // Mark as superseded
    
    // Create new verification
    Capsule::table("mod_email_verifications")->insert([
        "client_id" => $clientId,
        "email" => $client->email,
        "token" => hash("sha256", $token),
        "created_at" => date("Y-m-d H:i:s"),
        "expires_at" => date("Y-m-d H:i:s", strtotime("+24 hours")),
        "resend_count" => $todayCount + 1
    ]);
    
    // Send email
    $verifyLink = "https://" . $_SERVER["HTTP_HOST"] . 
                  "/verify-email.php?token=" . $token . 
                  "&id=" . $clientId;
    
    send_email("EmailVerification", $clientId, [
        "verification_link" => $verifyLink
    ]);
    
    return ["success" => true];
});
```

## Email Verification Template

```smarty
<!-- /templates/verify_email.tpl -->
<div class="email-verification-container">
    <div class="verification-card">
        {if $verified}
            <div class="success-message">
                <i class="fa fa-check-circle"></i>
                <h2>Email Verified!</h2>
                <p>Your email address has been successfully verified.</p>
                <a href="clientarea.php" class="btn btn-primary">
                    Go to Dashboard
                </a>
            </div>
        {elseif $error}
            <div class="error-message">
                <i class="fa fa-exclamation-circle"></i>
                <h2>Verification Failed</h2>
                <p>{$error}</p>
                <div class="resend-options">
                    <p>Didn't receive the email?</p>
                    <a href="resend-verification.php" class="btn btn-secondary">
                        Resend Verification Email
                    </a>
                </div>
            </div>
        {else}
            <div class="pending-message">
                <i class="fa fa-envelope"></i>
                <h2>Verify Your Email</h2>
                <p>We've sent a verification link to:</p>
                <strong>{$email}</strong>
                <p class="instructions">
                    Click the link in the email to verify your account.
                    The link will expire in 24 hours.
                </p>
                <div class="resend-options">
                    <a href="resend-verification.php" class="btn btn-outline">
                        <i class="fa fa-refresh"></i>
                        Resend Email
                    </a>
                </div>
            </div>
        {/if}
    </div>
</div>
```

## Best Practices

1. **Secure Tokens**: Use cryptographic random tokens with hashing
2. **Expiration**: Set reasonable expiration times (24 hours)
3. **Rate Limiting**: Limit resend attempts
4. **User Experience**: Clear instructions and status messages
5. **Fallback**: Alternative verification methods (support ticket)
6. **Logging**: Track all verification attempts
7. **Auto-Activate**: Optional auto-activation on verification
8. **Grace Period**: Allow unverified accounts limited access
