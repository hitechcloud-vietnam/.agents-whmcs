# WHMCS Client Registration Flow Workflow

## Description
Configure and customize client registration workflow in WHMCS.

## Prerequisites
- WHMCS 7.0+
- Customization requirements

## Steps

### Step 1: Configure Registration Settings
```php
<?php
// Configuration > System Settings > General
// Enable/disable client registration

$registration_config = [
    'allow_registration' => true,
    'require_email_verification' => true,
    'auto_generate_password' => false,
    'require_captcha' => true,
    'minimum_password_strength' => 3,
];
```

### Step 2: Custom Registration Fields
```php
<?php
// Add custom fields via admin or code

function addCustomRegistrationFields()
{
    // Company field
    Capsule::schema()->hasTable('tblcustomfields') 
        ? null 
        : createCustomField('Company', 'text', 'required');
    
    // VAT Number
    createCustomField('VAT Number', 'text', 'billing');
    
    // Custom checkbox
    createCustomField('Newsletter', 'tickbox', 'marketing');
}

function createCustomField($name, $type, $fieldArea)
{
    return Capsule::table('tblcustomfields')->insertGetId([
        'fieldname' => $name,
        'fieldtype' => $type,
        'fieldarea' => $fieldArea,
        'required' => 1,
        'showinvoice' => 1,
    ]);
}
```

### Step 3: Custom Validation Hook
```php
<?php
// includes/hooks/custom_registration_validation.php

add_hook('ClientAreaRegistration', 1, function($vars) {
    $email = $_POST['email'];
    $company = $_POST['customfield']['company'] ?? '';
    
    // Validate company name
    if (strlen($company) < 3) {
        return ['error' => 'Company name must be at least 3 characters'];
    }
    
    // Check for disposable email
    if (isDisposableEmail($email)) {
        return ['error' => 'Disposable email addresses are not allowed'];
    }
    
    // Check for existing registration
    $existing = Capsule::table('tblclients')
        ->where('email', $email)
        ->first();
    
    if ($existing) {
        return ['error' => 'An account with this email already exists'];
    }
    
    return true; // Allow registration
});
```

### Step 4: Post-Registration Actions
```php
<?php
add_hook('ClientAreaRegistrationCompleted', 1, function($vars) {
    $clientId = $vars['userid'];
    
    // Assign client group
    Capsule::table('tblclients')
        ->where('id', $clientId)
        ->update(['clientgroupid' => getDefaultClientGroup()]);
    
    // Set default currency
    Capsule::table('tblclients')
        ->where('id', $clientId)
        ->update(['currency' => 1]);
    
    // Send welcome email
    sendEmail('Welcome Email', $clientId);
    
    // Create activity log
    logActivity("New client registered: " . $vars['email'], $clientId);
    
    // Assign to sales rep
    assignToSalesRep($clientId);
});
```

### Step 5: Email Verification
```php
<?php
add_hook('ClientAreaRegistrationCompleted', 1, function($vars) {
    $clientId = $vars['userid'];
    $email = $vars['email'];
    
    // Generate verification token
    $token = bin2hex(random_bytes(32));
    
    Capsule::table('mod_email_verification')->insert([
        'client_id' => $clientId,
        'token' => hash('sha256', $token),
        'expires_at' => date('Y-m-d H:i:s', strtotime('+24 hours')),
        'created_at' => date('Y-m-d H:i:s'),
    ]);
    
    // Send verification email
    $verifyUrl = $systemUrl . '/verify-email.php?token=' . $token;
    sendEmail('EmailVerification', $clientId, ['verify_url' => $verifyUrl]);
    
    // Flag account as unverified
    Capsule::table('tblclients')
        ->where('id', $clientId)
        ->update(['email_verified' => 0]);
});
```

### Step 6: CAPTCHA Integration
```php
<?php
// Add reCAPTCHA to registration

add_hook('ClientAreaPageRegister', 1, function($vars) {
    $recaptchaSiteKey = Capsule::table('tbladdonmodules')
        ->where('module', 'recaptcha')
        ->where('setting', 'site_key')
        ->value('value');
    
    if ($recaptchaSiteKey) {
        $vars['recaptcha_script'] = '
            <script src="https://www.google.com/recaptcha/api.js" async defer></script>
            <div class="g-recaptcha" data-sitekey="' . $recaptchaSiteKey . '"></div>
        ';
    }
    
    return $vars;
});

add_hook('ClientAreaRegistration', 1, function($vars) {
    $recaptchaSecret = getRecaptchaSecret();
    
    $response = $_POST['g-recaptcha-response'];
    $verify = file_get_contents(
        "https://www.google.com/recaptcha/api/siteverify?secret=$recaptchaSecret&response=$response"
    );
    
    $result = json_decode($verify, true);
    
    if (!$result['success']) {
        return ['error' => 'CAPTCHA verification failed'];
    }
    
    return true;
});
```

## Registration Form Customization
```smarty
{* templates/clientregister.tpl *}

<div class="registration-form">
    <h2>Create Your Account</h2>
    
    <form method="post" action="register.php">
        <div class="form-group">
            <label>First Name *</label>
            <input type="text" name="firstname" required>
        </div>
        
        <div class="form-group">
            <label>Last Name *</label>
            <input type="text" name="lastname" required>
        </div>
        
        <div class="form-group">
            <label>Email Address *</label>
            <input type="email" name="email" required>
        </div>
        
        <div class="form-group">
            <label>Password *</label>
            <input type="password" name="password" required>
            <div class="password-strength"></div>
        </div>
        
        <div class="form-group">
            <label>Company Name</label>
            <input type="text" name="customfield[company]">
        </div>
        
        <div class="form-group">
            <label>Country *</label>
            <select name="country" required>
                <option value="">Select Country</option>
                {foreach $countries as $code => $name}
                    <option value="{$code}">{$name}</option>
                {/foreach}
            </select>
        </div>
        
        {$recaptcha_script}
        
        <button type="submit" class="btn btn-primary">Create Account</button>
    </form>
</div>
```

## Tags
- registration
- client-flow
- onboarding
- signup