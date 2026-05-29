# WHMCS Client Profile

## Overview
Master skill for client profile management in WHMCS. Covers profile updates, custom fields, and data management.

## Profile Hooks

```php
<?php
// /includes/hooks/profile_hooks.php

add_hook("ClientUpdate", 1, function(array $params) {
    $clientId = $params["clientid"];
    
    validateProfileUpdate($params);
    
    syncProfileToCRM($clientId, $params);
    
    return $params;
});

add_hook("CustomFieldSave", 1, function(array $params) {
    $clientId = $params["clientid"];
    saveCustomProfileData($clientId, $params["fields"]);
    return $params;
});

function validateProfileUpdate(array $params): bool
{
    if (empty($params["firstname"]) || empty($params["lastname"])) {
        throw new \Exception("Name is required");
    }
    
    if (!empty($params["email"]) && !filter_var($params["email"], FILTER_VALIDATE_EMAIL)) {
        throw new \Exception("Invalid email address");
    }
    
    return true;
}
```

## Profile Helper

```php
<?php
// /includes/helpers/profile_helper.php

function getClientProfile(int $clientId): array
{
    $client = \WHMCS\User\Client::find($clientId);
    
    return [
        "id" => $client->id,
        "name" => $client->fullName,
        "email" => $client->email,
        "company" => $client->companyname,
        "address" => [
            "address1" => $client->address1,
            "address2" => $client->address2,
            "city" => $client->city,
            "state" => $client->state,
            "postcode" => $client->postcode,
            "country" => $client->country,
        ],
        "phone" => $client->phonenumber,
        "custom_fields" => getCustomFields($clientId),
    ];
}

function updateClientProfile(int $clientId, array $data): array
{
    $client = \WHMCS\User\Client::find($clientId);
    
    foreach (["firstname", "lastname", "companyname", "email", "address1", "address2", "city", "state", "postcode", "country", "phonenumber"] as $field) {
        if (isset($data[$field])) {
            $client->$field = $data[$field];
        }
    }
    
    $client->save();
    
    return ["success" => true];
}

function getCustomFields(int $clientId): array
{
    return \Illuminate\Database\Capsule\Manager::table("tblcustomfieldsvalues")
        ->join("tblcustomfields", "tblcustomfieldsvalues.fieldid", "=", "tblcustomfields.id")
        ->where("tblcustomfieldsvalues.relid", $clientId)
        ->select("tblcustomfields.fieldname", "tblcustomfieldsvalues.value")
        ->get()
        ->toArray();
}
```

## Best Practices

1. **Validation**: Validate all profile updates
2. **Permissions**: Control who can update profiles
3. **Custom Fields**: Use custom fields for additional data
4. **Verification**: Verify email on change
5. **Audit Trail**: Log profile changes
6. **Privacy**: Respect data protection laws
7. **Completeness**: Track profile completion
8. **Sync**: Keep external systems in sync
