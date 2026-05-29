# WHMCS Client Registration

## Overview
Master skill for customizing client registration in WHMCS. Covers registration forms, validation, and automation.

## Registration Hooks

```php
<?php
// /includes/hooks/registration_hooks.php

add_hook("ClientAdd", 1, function(array $params) {
    $clientId = $params["clientid"];
    
    send_email("WelcomeNewClient", $clientId, []);
    
    addToDefaultGroup($clientId);
    
    createWelcomeTicket($clientId);
    
    return $params;
});

add_hook("CustomFieldSave", 1, function(array $params) {
    if ($params["fieldname"] === "company_type") {
        validateCompanyRegistration($params);
    }
    return $params;
});

add_hook("PreClientAdd", 1, function(array $params) {
    $errors = [];
    
    if (isBlockedEmail($params["email"])) {
        $errors[] = "Email domain is not allowed";
    }
    
    if (emailExists($params["email"])) {
        $errors[] = "Email already registered";
    }
    
    if (!empty($errors)) {
        return ["error" => implode(", ", $errors)];
    }
    
    return ["success" => true];
});

function isBlockedEmail(string $email): bool
{
    $domain = explode("@", $email)[1] ?? "";
    $blocked = ["tempmail.com", "throwaway.com"];
    return in_array($domain, $blocked);
}

function addToDefaultGroup(int $clientId): void
{
    $defaultGroup = \Illuminate\Database\Capsule\Manager::table("tblclientgroups")
        ->where("is_default", 1)
        ->first();
    
    if ($defaultGroup) {
        \Illuminate\Database\Capsule\Manager::table("tblclients")
            ->where("id", $clientId)
            ->update(["groupid" => $defaultGroup->id]);
    }
}
```

## Best Practices

1. **Validation**: Validate all inputs before registration
2. **Verification**: Email verification is recommended
3. **Custom Fields**: Collect necessary additional information
4. **Defaults**: Assign appropriate default groups
5. **Automation**: Welcome emails and onboarding
6. **Security**: Prevent spam registrations
7. **Privacy**: Comply with data protection regulations
8. **UX**: Simple and clear registration flow
