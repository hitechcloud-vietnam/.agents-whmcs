# WHMCS User Registration

## Overview
Comprehensive guide for customizing user registration flows in WHMCS. Covers form customization, validation, verification, and automation.

## Registration Hooks

### Pre-Registration Validation

```php
<?php
// /includes/hooks/user_registration.php

add_hook("PreClientAdd", 1, function(array $params) {
    $errors = [];
    
    // Validate email domain
    $domain = explode("@", $params["email"])[1] ?? "";
    $blockedDomains = Capsule::table("mod_blocked_domains")->pluck("domain");
    if (in_array($domain, $blockedDomains)) {
        $errors[] = "This email domain is not allowed";
    }
    
    // Check for existing account
    if (emailExists($params["email"])) {
        $errors[] = "An account with this email already exists";
    }
    
    // Custom validation
    if (!empty($params["companyname"]) && strlen($params["companyname"]) < 2) {
        $errors[] = "Company name must be at least 2 characters";
    }
    
    if (!empty($errors)) {
        return ["error" => implode(". ", $errors)];
    }
    
    return ["success" => true];
});
```

### Post-Registration Actions

```php
add_hook("ClientAdd", 1, function(array $params) {
    $clientId = $params["clientid"];
    
    // Assign to default group
    $defaultGroup = Capsule::table("tblclientgroups")
        ->where("is_default", 1)
        ->first();
    if ($defaultGroup) {
        Capsule::table("tblclients")
            ->where("id", $clientId)
            ->update(["groupid" => $defaultGroup->id]);
    }
    
    // Create welcome ticket
    openSupportTicket($clientId, "Welcome to " . WHMCS\Config\Setting::getValue("CompanyName"), "Thank you for registering!");
    
    // Add welcome credit
    addCredit($clientId, 0.00, "Welcome bonus", "welcome_bonus");
    
    // Send welcome email
    send_email("WelcomeNewClient", $clientId, []);
    
    return $params;
});
```

### Custom Registration Fields

```php
add_hook("CustomFieldSave", 1, function(array $params) {
    if ($params["fieldname"] === "referral_code") {
        $referrerId = validateReferralCode($params["value"]);
        if ($referrerId) {
            recordReferral($referrerId, $params["clientid"]);
        }
    }
    return $params;
});

function validateReferralCode(string $code): ?int
{
    $referral = Capsule::table("mod_referral_codes")
        ->where("code", $code)
        ->where("used", 0)
        ->first();
    return $referral ? $referral->client_id : null;
}
```

## Form Template Customization

### Client Area Registration Template

```smarty
<!-- /templates/client_register.tpl -->
<form method="post" action="register.php" class="registration-form">
    <input type="hidden" name="token" value="{$token}">
    
    <div class="form-group">
        <label for="firstname">First Name *</label>
        <input type="text" name="firstname" id="firstname" 
               value="{$client->firstName}" required
               class="form-control">
    </div>
    
    <div class="form-group">
        <label for="lastname">Last Name *</label>
        <input type="text" name="lastname" id="lastname" 
               value="{$client->lastName}" required
               class="form-control">
    </div>
    
    <div class="form-group">
        <label for="email">Email Address *</label>
        <input type="email" name="email" id="email" 
               value="{$client->email}" required
               class="form-control" 
               data-ajax-check="validate_email">
        <span class="email-status"></span>
    </div>
    
    <div class="form-group">
        <label for="password">Password *</label>
        <input type="password" name="password" id="password" 
               required minlength="8"
               class="form-control">
        <div class="password-strength"></div>
    </div>
    
    <div class="form-group">
        <label for="phone">Phone Number</label>
        <input type="tel" name="phonenumber" id="phone" 
               class="form-control">
    </div>
    
    <div class="form-group">
        <label for="company">Company Name</label>
        <input type="text" name="companyname" id="company" 
               class="form-control">
    </div>
    
    <div class="form-group">
        <label for="country">Country *</label>
        <select name="country" id="country" required class="form-control">
            <option value="">Select Country</option>
            {foreach $countries as $code => $name}
                <option value="{$code}" {if $code == $client->country}selected{/if}>
                    {$name}
                </option>
            {/foreach}
        </select>
    </div>
    
    <div class="form-group">
        <label for="referral">Referral Code</label>
        <input type="text" name="customfields[referral_code]" 
               id="referral" class="form-control">
    </div>
    
    <div class="form-group checkbox">
        <label>
            <input type="checkbox" name="marketing" value="1">
            Subscribe to newsletter
        </label>
    </div>
    
    <div class="form-group checkbox">
        <label>
            <input type="checkbox" name="terms" value="1" required>
            I agree to <a href="terms.php">Terms of Service</a>
        </label>
    </div>
    
    <button type="submit" class="btn btn-primary btn-block">
        Create Account
    </button>
</form>
```

## API Registration

```php
<?php
// API-based registration
function registerClientViaAPI(array $data): array
{
    $command = "AddClient";
    $postData = [
        "firstname" => $data["firstname"],
        "lastname" => $data["lastname"],
        "email" => $data["email"],
        "password2" => $data["password"],
        "country" => $data["country"],
        "state" => $data["state"] ?? "",
        "city" => $data["city"] ?? "",
        "address1" => $data["address1"] ?? "",
        "phonenumber" => $data["phone"] ?? "",
        "companyname" => $data["company"] ?? "",
        "clientip" => $_SERVER["REMOTE_ADDR"],
    ];
    
    $results = localAPI($command, $postData);
    
    if ($results["result"] === "success") {
        return [
            "success" => true,
            "client_id" => $results["clientid"]
        ];
    }
    
    return [
        "success" => false,
        "error" => $results["message"] ?? "Registration failed"
    ];
}
```

## Best Practices

1. **Validation**: Validate all inputs server-side and client-side
2. **Password Strength**: Enforce strong password requirements
3. **Email Verification**: Implement email verification flow
4. **Spam Prevention**: Use CAPTCHA, rate limiting, email domain blocks
5. **Custom Fields**: Collect only necessary information
6. **GDPR Compliance**: Clear consent for data processing
7. **User Experience**: Progressive form fields, real-time validation
8. **Security**: CSRF tokens, honeypot fields, rate limiting
