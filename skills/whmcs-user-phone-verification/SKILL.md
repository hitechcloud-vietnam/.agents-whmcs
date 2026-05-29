# WHMCS User Phone Verification

## Overview
Guide for implementing SMS phone verification in WHMCS. Covers OTP generation, SMS delivery, and verification flow.

## Phone Verification Implementation

### Generate OTP

```php
<?php
// /includes/hooks/phone_verification.php

add_hook("GeneratePhoneOTP", 1, function(array $params) {
    $clientId = $params["client_id"];
    $phone = formatPhoneNumber($params["phone"]);
    
    // Rate limiting - max 5 OTP requests per hour
    $recentRequests = Capsule::table("mod_phone_otp")
        ->where("client_id", $clientId)
        ->where("created_at", ">", date("Y-m-d H:i:s", strtotime("-1 hour")))
        ->count();
    
    if ($recentRequests >= 5) {
        return ["error" => "Maximum OTP requests reached. Please try again later."];
    }
    
    // Generate 6-digit OTP
    $otp = str_pad((string)random_int(0, 999999), 6, "0", STR_PAD_LEFT);
    
    // Store OTP
    Capsule::table("mod_phone_otp")->insert([
        "client_id" => $clientId,
        "phone" => $phone,
        "otp_hash" => password_hash($otp, PASSWORD_DEFAULT),
        "purpose" => $params["purpose"] ?? "verification",
        "attempts" => 0,
        "created_at" => date("Y-m-d H:i:s"),
        "expires_at" => date("Y-m-d H:i:s", strtotime("+10 minutes")),
        "verified" => 0
    ]);
    
    // Send SMS
    $result = sendSMS($phone, "Your verification code is: {$otp}. Valid for 10 minutes.");
    
    if (!$result["success"]) {
        return ["error" => "Failed to send SMS: " . $result["error"]];
    }
    
    return ["success" => true, "message" => "Verification code sent"];
});

function formatPhoneNumber(string $phone): string
{
    // Remove all non-digit characters
    $digits = preg_replace("/[^0-9]/", "", $phone);
    
    // Add country code if missing
    if (strlen($digits) === 10) {
        $digits = "1" . $digits; // Assume US/Canada
    }
    
    return "+" . $digits;
}
```

### Verify OTP

```php
add_hook("VerifyPhoneOTP", 1, function(array $params) {
    $clientId = $params["client_id"];
    $otp = $params["otp"];
    
    // Find valid OTP record
    $record = Capsule::table("mod_phone_otp")
        ->where("client_id", $clientId)
        ->where("verified", 0)
        ->where("expires_at", ">", date("Y-m-d H:i:s"))
        ->orderBy("id", "desc")
        ->first();
    
    if (!$record) {
        return ["error" => "Invalid or expired verification code."];
    }
    
    // Check attempt limit
    if ($record->attempts >= 5) {
        Capsule::table("mod_phone_otp")
            ->where("id", $record->id)
            ->update(["verified" => 2]); // Mark as exceeded
        return ["error" => "Too many attempts. Please request a new code."];
    }
    
    // Increment attempts
    Capsule::table("mod_phone_otp")
        ->where("id", $record->id)
        ->increment("attempts");
    
    // Verify OTP
    if (!password_verify($otp, $record->otp_hash)) {
        return ["error" => "Invalid verification code."];
    }
    
    // Mark as verified
    Capsule::table("mod_phone_otp")
        ->where("id", $record->id)
        ->update([
            "verified" => 1,
            "verified_at" => date("Y-m-d H:i:s")
        ]);
    
    // Update client phone if needed
    if (!empty($params["phone"])) {
        Capsule::table("tblclients")
            ->where("id", $clientId)
            ->update(["phonenumber" => $record->phone]);
    }
    
    // Log verification
    Capsule::table("mod_verification_log")->insert([
        "client_id" => $clientId,
        "type" => "phone",
        "phone" => $record->phone,
        "verified_at" => date("Y-m-d H:i:s"),
        "ip_address" => $_SERVER["REMOTE_ADDR"]
    ]);
    
    return ["success" => true];
});

function sendSMS(string $phone, string $message): array
{
    // SMS gateway integration (e.g., Twilio)
    $gateway = Capsule::table("tblpaymentgateways")
        ->where("gateway", "twilio")
        ->first();
    
    if (!$gateway) {
        return ["success" => false, "error" => "SMS gateway not configured"];
    }
    
    $settings = json_decode($gateway->settings, true);
    
    $ch = curl_init();
    curl_setopt_array($ch, [
        CURLOPT_URL => "https://api.twilio.com/2010-04-01/Accounts/" . 
                      $settings["account_sid"] . "/Messages.json",
        CURLOPT_RETURNTRANSFER => true,
        CURLOPT_POST => true,
        CURLOPT_POSTFIELDS => http_build_query([
            "To" => $phone,
            "From" => $settings["from_number"],
            "Body" => $message
        ]),
        CURLOPT_HTTPHEADER => [
            "Authorization: Basic " . base64_encode(
                $settings["account_sid"] . ":" . $settings["auth_token"]
            )
        ]
    ]);
    
    $response = curl_exec($ch);
    $httpCode = curl_getinfo($ch, CURLINFO_HTTP_CODE);
    curl_close($ch);
    
    $result = json_decode($response, true);
    
    if ($httpCode === 201) {
        return ["success" => true, "message_sid" => $result["sid"]];
    }
    
    return ["success" => false, "error" => $result["message"] ?? "SMS failed"];
}
```

## Phone Verification Template

```smarty
<!-- /templates/verify_phone.tpl -->
<div class="phone-verification-container">
    <div class="verification-card">
        <h2>Verify Your Phone Number</h2>
        <p>We'll send you a verification code to:</p>
        <strong class="phone-display">{$phone}</strong>
        
        {if $error}
            <div class="alert alert-danger">{$error}</div>
        {/if}
        
        <form method="post" action="verify-phone.php" class="otp-form">
            <input type="hidden" name="token" value="{$token}">
            <input type="hidden" name="phone" value="{$phone}">
            
            <div class="form-group">
                <label for="otp">Enter Verification Code</label>
                <div class="otp-inputs">
                    <input type="text" name="otp1" maxlength="1" 
                           class="otp-digit" autofocus>
                    <input type="text" name="otp2" maxlength="1" 
                           class="otp-digit">
                    <input type="text" name="otp3" maxlength="1" 
                           class="otp-digit">
                    <input type="text" name="otp4" maxlength="1" 
                           class="otp-digit">
                    <input type="text" name="otp5" maxlength="1" 
                           class="otp-digit">
                    <input type="text" name="otp6" maxlength="1" 
                           class="otp-digit">
                </div>
            </div>
            
            <button type="submit" class="btn btn-primary btn-block">
                Verify Phone
            </button>
            
            <div class="resend-options">
                <p>Didn't receive the code?</p>
                <a href="?action=resend" class="btn btn-link">
                    Resend Code
                </a>
                <span class="countdown" data-seconds="60">
                    Resend available in <span></span>s
                </span>
            </div>
        </form>
    </div>
</div>

<script>
// OTP input auto-advance
document.querySelectorAll('.otp-digit').forEach((input, index, inputs) => {
    input.addEventListener('input', function() {
        if (this.value.length === 1 && index < inputs.length - 1) {
            inputs[index + 1].focus();
        }
    });
    
    input.addEventListener('keydown', function(e) {
        if (e.key === 'Backspace' && this.value === '' && index > 0) {
            inputs[index - 1].focus();
        }
    });
});
</script>
```

## Best Practices

1. **OTP Security**: Use cryptographic random and hash storage
2. **Rate Limiting**: Prevent brute force attacks
3. **Expiration**: Short expiry times (10 minutes)
4. **Attempts**: Limit verification attempts
5. **SMS Delivery**: Use reliable SMS gateway (Twilio, Nexmo)
6. **Fallback**: Alternative verification methods
7. **International**: Support multiple country codes
8. **Logging**: Track all verification attempts
