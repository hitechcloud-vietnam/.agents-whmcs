# WHMCS User Groups

## Overview
Guide for implementing custom user group management in WHMCS. Covers group creation, assignment, and automated group rules.

## User Group System

### Create Group

```php
<?php
// /includes/hooks/user_groups.php

add_hook("CreateUserGroup", 1, function(array $params) {
    $groupName = $params["name"];
    
    // Check if group exists
    $existing = Capsule::table("mod_user_groups")
        ->where("name", $groupName)
        ->first();
    
    if ($existing) {
        return ["error" => "Group name already exists.", "group_id" => $existing->id];
    }
    
    $groupId = Capsule::table("mod_user_groups")->insertGetId([
        "name" => $groupName,
        "description" => $params["description"] ?? "",
        "discount_percent" => $params["discount"] ?? 0,
        "suspension_threshold" => $params["suspension_threshold"] ?? 0,
        "auto_suspend_days" => $params["auto_suspend_days"] ?? 7,
        "payment_terms" => $params["payment_terms"] ?? "default",
        "support_level" => $params["support_level"] ?? "standard",
        "created_at" => date("Y-m-d H:i:s"),
        "updated_at" => date("Y-m-d H:i:s")
    ]);
    
    // Apply default permissions
    applyGroupPermissions($groupId, $params["permissions"] ?? []);
    
    return ["success" => true, "group_id" => $groupId];
});

function applyGroupPermissions(int $groupId, array $permissions): void
{
    foreach ($permissions as $permission) {
        Capsule::table("mod_group_permissions")->insert([
            "group_id" => $groupId,
            "permission" => $permission
        ]);
    }
}
```

### Assign User to Group

```php
add_hook("AssignUserToGroup", 1, function(array $params) {
    $clientId = $params["client_id"];
    $groupId = $params["group_id"];
    
    $group = Capsule::table("mod_user_groups")->where("id", $groupId)->first();
    if (!$group) {
        return ["error" => "Group not found."];
    }
    
    $oldGroupId = Capsule::table("tblclients")->where("id", $clientId)->value("groupid");
    
    // Update client group
    Capsule::table("tblclients")
        ->where("id", $clientId)
        ->update([
            "groupid" => $groupId,
            "group_changed_at" => date("Y-m-d H:i:s")
        ]);
    
    // Apply group-specific settings
    applyGroupSettings($clientId, $group);
    
    // Log change
    Capsule::table("mod_group_assignments")->insert([
        "client_id" => $clientId,
        "old_group_id" => $oldGroupId,
        "new_group_id" => $groupId,
        "assigned_by" => $params["assigned_by"] ?? 0,
        "reason" => $params["reason"] ?? "",
        "assigned_at" => date("Y-m-d H:i:s")
    ]);
    
    return ["success" => true];
});

function applyGroupSettings(int $clientId, object $group): void
{
    // Apply group discounts to existing services
    if ($group->discount_percent > 0) {
        $services = Capsule::table("tblhosting")
            ->where("userid", $clientId)
            ->whereIn("domainstatus", ["Active"])
            ->get();
        
        foreach ($services as $service) {
            $discount = $service->amount * ($group->discount_percent / 100);
            Capsule::table("tblhosting")
                ->where("id", $service->id)
                ->update([
                    "discount" => ($service->discount ?? 0) + $discount
                ]);
        }
    }
}
```

### Auto-Assignment Rules

```php
add_hook("ClientAdd", 1, function(array $params) {
    autoAssignGroup($params["clientid"]);
    return $params;
});

function autoAssignGroup(int $clientId): void
{
    $client = Capsule::table("tblclients")->where("id", $clientId)->first();
    
    // Get all active rules
    $rules = Capsule::table("mod_group_rules")
        ->where("active", 1)
        ->orderBy("priority", "desc")
        ->get();
    
    foreach ($rules as $rule) {
        $conditions = json_decode($rule->conditions, true);
        
        if (evaluateGroupRule($client, $conditions)) {
            add_hook("AssignUserToGroup", 1, function($params) use ($rule) {
                $params["group_id"] = $rule->group_id;
                $params["reason"] = "Auto-assigned by rule: " . $rule->name;
                return $params;
            });
            
            Capsule::table("tblclients")
                ->where("id", $clientId)
                ->update(["groupid" => $rule->group_id]);
            
            break; // Apply first matching rule only
        }
    }
}

function evaluateGroupRule(object $client, array $conditions): bool
{
    foreach ($conditions as $field => $condition) {
        $value = $client->{$field} ?? "";
        
        switch ($condition["operator"]) {
            case "equals":
                if ($value !== $condition["value"]) return false;
                break;
            case "contains":
                if (stripos($value, $condition["value"]) === false) return false;
                break;
            case "starts_with":
                if (strpos($value, $condition["value"]) !== 0) return false;
                break;
            case "in":
                if (!in_array($value, $condition["values"])) return false;
                break;
            case "greater_than":
                if (!(float)$value > (float)$condition["value"]) return false;
                break;
        }
    }
    
    return true;
}
```

### Group-Based Features

```php
function hasGroupFeature(int $clientId, string $feature): bool
{
    $client = Capsule::table("tblclients")->where("id", $clientId)->first();
    
    // Check group features
    $groupFeature = Capsule::table("mod_group_features")
        ->where("group_id", $client->groupid)
        ->where("feature", $feature)
        ->first();
    
    return $groupFeature && $groupFeature->enabled;
}

add_hook("GetClientGroupFeatures", 1, function(array $params) {
    $clientId = $params["client_id"];
    $client = Capsule::table("tblclients")->where("id", $clientId)->first();
    
    $features = Capsule::table("mod_group_features")
        ->where("group_id", $client->groupid)
        ->where("enabled", 1)
        ->pluck("feature")
        ->toArray();
    
    return ["features" => $features];
});
```

## Group Template

```smarty
<!-- /templates/admin_user_groups.tpl -->
<div class="admin-user-groups">
    <h2>User Groups</h2>
    
    <div class="group-list">
        {foreach $groups as $group}
            <div class="group-card">
                <h3>{$group.name}</h3>
                <p>{$group.description}</p>
                
                <div class="group-stats">
                    <span class="stat">
                        <strong>{$group.member_count}</strong> Members
                    </span>
                    <span class="stat">
                        <strong>{$group.discount_percent}%</strong> Discount
                    </span>
                </div>
                
                <div class="group-actions">
                    <a href="edit_group.php?id={$group.id}" class="btn btn-sm">
                        Edit
                    </a>
                    <a href="view_group.php?id={$group.id}" class="btn btn-sm">
                        View Members
                    </a>
                </div>
            </div>
        {/foreach}
    </div>
    
    <a href="create_group.php" class="btn btn-primary">
        <i class="fa fa-plus"></i> Create New Group
    </a>
</div>
```

## Best Practices

1. **Logical Naming**: Clear, descriptive group names
2. **Tiered Structure**: Support levels (basic, standard, premium)
3. **Auto-Assignment**: Automated group rules based on criteria
4. **Discount Integration**: Automatic discount application
5. **Feature Flags**: Group-based feature access
6. **Migration**: Tools to move users between groups
7. **Audit Trail**: Log all group assignments
8. **Reporting**: Group-based analytics and reporting
