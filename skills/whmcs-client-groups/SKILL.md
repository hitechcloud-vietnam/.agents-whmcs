# WHMCS Client Groups

## Overview
Master skill for managing client groups in WHMCS. Covers group assignment, permissions, and automated group management.

## Client Group Hooks

```php
<?php
// /includes/hooks/client_groups_hooks.php

add_hook("ClientAdd", 1, function(array $params) {
    $clientId = $params["clientid"];
    
    autoAssignClientGroup($clientId);
    
    return $params;
});

add_hook("InvoicePaid", 1, function(array $params) {
    $clientId = $params["userid"];
    
    updateClientGroupBasedOnSpend($clientId);
    
    return true;
});

function autoAssignClientGroup(int $clientId): void
{
    $client = \WHMCS\User\Client::find($clientId);
    
    $rules = \Illuminate\Database\Capsule\Manager::table("mod_client_group_rules")
        ->where("is_active", 1)
        ->get();
    
    foreach ($rules as $rule) {
        if (evaluateGroupRule($client, $rule)) {
            \Illuminate\Database\Capsule\Manager::table("tblclients")
                ->where("id", $clientId)
                ->update(["groupid" => $rule->group_id]);
            break;
        }
    }
}

function evaluateGroupRule($client, $rule): bool
{
    if ($rule->condition === "country") {
        return $client->country === $rule->value;
    }
    
    if ($rule->condition === "total_spent") {
        $spent = getClientTotalSpent($client->id);
        return $spent >= (float)$rule->value;
    }
    
    if ($rule->condition === "email_domain") {
        $domain = explode("@", $client->email)[1] ?? "";
        return $domain === $rule->value;
    }
    
    return false;
}

function getClientTotalSpent(int $clientId): float
{
    return \Illuminate\Database\Capsule\Manager::table("tblinvoices")
        ->where("userid", $clientId)
        ->where("status", "Paid")
        ->sum("total");
}
```

## Best Practices

1. **Clear Naming**: Use descriptive group names
2. **Automation**: Auto-assign based on criteria
3. **Permissions**: Control access by group
4. **Pricing**: Apply group-specific pricing
5. **Notifications**: Send group-specific emails
6. **Reporting**: Track group performance
7. **Migration**: Smooth group transitions
8. **Documentation**: Document group purposes
