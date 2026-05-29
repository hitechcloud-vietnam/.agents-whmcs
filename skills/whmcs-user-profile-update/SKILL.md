# WHMCS User Profile Update

## Overview
Guide for implementing user profile management functionality in WHMCS. Covers profile editing, validation, and custom fields.

## Profile Update Hooks

### Pre-Profile Update Validation

```php
<?php
// /includes/hooks/profile_update.php

add_hook("PreClientUpdate", 1, function(array $params) {
    $errors = [];
    $userId = $params["userid"];
    
    // Validate email change
    if (!empty($params["email"])) {
        $existing = Capsule::table("tblclients")
            ->where("email", $params["email"])
            ->where("id", "!=", $userId)
            ->first();
        
        if ($existing) {
            $errors[] = "This email address is already in use.";
        }
        
        // Require password confirmation for email change
        if ($params["email"] !== getCurrentEmail($userId)) {
            if (empty($params["password_confirm"])) {
                return ["error" => "Password confirmation required to change email."];
            }
            
            if (!verifyPassword($userId, $params["password_confirm"])) {
                return ["error" => "Incorrect password for email change."];
            }
        }
    }
    
    // Validate phone number
    if (!empty($params["phonenumber"])) {
        if (!preg_match("/^[+]?[\d\s-]{10,}$/", $params["phonenumber"])) {
            $errors[] = "Invalid phone number format.";
        }
    }
    
    // Validate custom fields
    if (!empty($params["customfields"])) {
        $customErrors = validateCustomFields($params["customfields"]);
        $errors = array_merge($errors, $customErrors);
    }
    
    if (!empty($errors)) {
        return ["error" => implode(" ", $errors)];
    }
    
    return ["success" => true];
});

function verifyPassword(int $userId, string $password): bool
{
    $client = Capsule::table("tblclients")->where("id", $userId)->first();
    return password_verify($password, $client->password);
}
```

### Post-Profile Update Actions

```php
add_hook("ClientUpdate", 1, function(array $params) {
    $userId = $params["userid"];
    
    // Clear cache
    Capsule::table("mod_user_cache")
        ->where("client_id", $userId)
        ->delete();
    
    // Update contacts if applicable
    if (!empty($params["update_contacts"])) {
        updateSubAccountEmails($userId, $params["email"]);
    }
    
    // Log profile changes
    Capsule::table("mod_profile_changes")->insert([
        "client_id" => $userId,
        "changed_fields" => json_encode(array_keys($params)),
        "ip_address" => $_SERVER["REMOTE_ADDR"],
        "changed_at" => date("Y-m-d H:i:s")
    ]);
    
    // Send notification if email changed
    if (!empty($params["email"]) && $params["email"] !== getCurrentEmail($userId)) {
        send_email("ProfileEmailChanged", $userId, [
            "old_email" => getCurrentEmail($userId),
            "new_email" => $params["email"],
            "ip_address" => $_SERVER["REMOTE_ADDR"]
        ]);
        
        // Send confirmation to new email
        send_email("ProfileEmailChangedConfirm", $userId, [
            "new_email" => $params["email"]
        ]);
    }
    
    return $params;
});
```

### Custom Field Updates

```php
add_hook("CustomFieldSave", 1, function(array $params) {
    $clientId = $params["relid"];
    $fieldName = $params["fieldname"];
    $value = $params["value"];
    
    // Validate specific custom fields
    switch ($fieldName) {
        case "tax_id":
            if (!validateTaxId($value)) {
                return ["error" => "Invalid Tax ID format."];
            }
            break;
            
        case "company_registration":
            if (!verifyCompanyRegistration($value)) {
                return ["error" => "Unable to verify company registration."];
            }
            break;
            
        case "date_of_birth":
            if (!validateAge($value, 18)) {
                return ["error" => "You must be at least 18 years old."];
            }
            break;
    }
    
    return ["success" => true];
});
```

## Profile Template

```smarty
<!-- /templates/clientarea_profile.tpl -->
<div class="profile-container">
    <div class="profile-header">
        <h2>My Profile</h2>
    </div>
    
    <ul class="profile-tabs">
        <li class="active"><a href="#personal">Personal Info</a></li>
        <li><a href="#contact">Contact Details</a></li>
        <li><a href="#security">Security</a></li>
        <li><a href="#preferences">Preferences</a></li>
    </ul>
    
    <form method="post" action="clientarea.php?action=profile" 
          class="profile-form" id="personal">
        <input type="hidden" name="token" value="{$token}">
        <input type="hidden" name="tab" value="personal">
        
        <div class="form-section">
            <h3>Personal Information</h3>
            
            <div class="form-row">
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
            </div>
            
            <div class="form-group">
                <label for="companyname">Company Name</label>
                <input type="text" name="companyname" id="companyname" 
                       value="{$client->companyName}"
                       class="form-control">
            </div>
        </div>
        
        <div class="form-section">
            <h3>Contact Information</h3>
            
            <div class="form-group">
                <label for="email">Email Address *</label>
                <input type="email" name="email" id="email" 
                       value="{$client->email}" required
                       class="form-control">
                <small class="help-text">
                    Changing email requires password confirmation
                </small>
            </div>
            
            <div class="form-group">
                <label for="phonenumber">Phone Number</label>
                <input type="tel" name="phonenumber" id="phonenumber" 
                       value="{$client->phoneNumber}"
                       class="form-control">
            </div>
            
            <div class="form-group">
                <label for="address1">Address Line 1</label>
                <input type="text" name="address1" id="address1" 
                       value="{$client->address1}"
                       class="form-control">
            </div>
            
            <div class="form-group">
                <label for="address2">Address Line 2</label>
                <input type="text" name="address2" id="address2" 
                       value="{$client->address2}"
                       class="form-control">
            </div>
            
            <div class="form-row">
                <div class="form-group">
                    <label for="city">City</label>
                    <input type="text" name="city" id="city" 
                           value="{$client->city}"
                           class="form-control">
                </div>
                
                <div class="form-group">
                    <label for="state">State/Region</label>
                    <input type="text" name="state" id="state" 
                           value="{$client->state}"
                           class="form-control">
                </div>
                
                <div class="form-group">
                    <label for="postcode">Postcode</label>
                    <input type="text" name="postcode" id="postcode" 
                           value="{$client->postcode}"
                           class="form-control">
                </div>
            </div>
            
            <div class="form-group">
                <label for="country">Country</label>
                <select name="country" id="country" class="form-control">
                    {foreach $countries as $code => $name}
                        <option value="{$code}" 
                                {if $code == $client->country}selected{/if}>
                            {$name}
                        </option>
                    {/foreach}
                </select>
            </div>
        </div>
        
        <div class="form-actions">
            <button type="submit" class="btn btn-primary">
                Save Changes
            </button>
        </div>
    </form>
</div>
```

## API Integration

```php
<?php
// API-based profile update
function updateClientProfileAPI(int $userId, array $data): array
{
    $postData = [
        "clientid" => $userId,
    ];
    
    // Only include changed fields
    $allowedFields = [
        "firstname", "lastname", "email", "companyname",
        "phonenumber", "address1", "address2", "city",
        "state", "postcode", "country"
    ];
    
    foreach ($allowedFields as $field) {
        if (isset($data[$field])) {
            $postData[$field] = $data[$field];
        }
    }
    
    $command = "UpdateClient";
    $results = localAPI($command, $postData);
    
    return $results;
}
```

## Best Practices

1. **Validation**: Validate all inputs before saving
2. **Email Changes**: Require password confirmation
3. **Custom Fields**: Support custom field validation
4. **Audit Trail**: Log all profile changes
5. **Notifications**: Notify users of important changes
6. **Partial Updates**: Allow updating individual fields
7. **Version Control**: Track profile change history
8. **Security**: CSRF protection, input sanitization
