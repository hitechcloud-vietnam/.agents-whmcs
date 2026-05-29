# WHMCS User Impersonation

## Overview
Guide for implementing admin user impersonation in WHMCS. Covers impersonation logging, session management, and security controls.

## Impersonation Implementation

### Start Impersonation

```php
<?php
// /includes/hooks/impersonation.php

add_hook("AdminImpersonateStart", 1, function(array $params) {
    $adminId = $params["admin_id"];
    $targetClientId = $params["client_id"];
    
    // Verify admin permissions
    $admin = Capsule::table("tbladmins")->where("id", $adminId)->first();
    if (!$admin || !hasPermission($admin, "impersonate_clients")) {
        return ["error" => "Insufficient permissions for impersonation."];
    }
    
    // Rate limiting
    $recentImpersonations = Capsule::table("mod_impersonation_log")
        ->where("admin_id", $adminId)
        ->where("started_at", ">", date("Y-m-d H:i:s", strtotime("-1 hour")))
        ->count();
    
    if ($recentImpersonations >= 10) {
        return ["error" => "Impersonation rate limit exceeded."];
    }
    
    // Generate impersonation session
    $sessionToken = bin2hex(random_bytes(32));
    
    Capsule::table("mod_impersonation_sessions")->insert([
        "admin_id" => $adminId,
        "client_id" => $targetClientId,
        "session_token" => hash("sha256", $sessionToken),
        "ip_address" => $_SERVER["REMOTE_ADDR"],
        "user_agent" => $_SERVER["HTTP_USER_AGENT"] ?? "",
        "started_at" => date("Y-m-d H:i:s"),
        "expires_at" => date("Y-m-d H:i:s", strtotime("+30 minutes")),
        "actions_performed" => json_encode([])
    ]);
    
    // Log impersonation start
    Capsule::table("mod_impersonation_log")->insert([
        "admin_id" => $adminId,
        "client_id" => $targetClientId,
        "action" => "start",
        "ip_address" => $_SERVER["REMOTE_ADDR"],
        "performed_at" => date("Y-m-d H:i:s")
    ]);
    
    // Store in session
    $_SESSION["impersonating"] = [
        "admin_id" => $adminId,
        "client_id" => $targetClientId,
        "session_token" => $sessionToken,
        "original_session" => session_id()
    ];
    
    return [
        "success" => true,
        "session_token" => $sessionToken,
        "expires_at" => date("Y-m-d H:i:s", strtotime("+30 minutes"))
    ];
});

function hasPermission(array $admin, string $permission): bool
{
    $roleid = $admin["roleid"];
    $permissions = Capsule::table("tbladminperms")
        ->where("roleid", $roleid)
        ->pluck("permid");
    
    $permissionId = Capsule::table("tblpermissions")
        ->where("name", $permission)
        ->value("id");
    
    return in_array($permissionId, $permissions);
}
```

### During Impersonation

```php
add_hook("ClientAreaPage", 1, function(array $params) {
    if (!isImpersonating()) {
        return $params;
    }
    
    $impersonationData = $_SESSION["impersonating"];
    
    // Check session validity
    $session = Capsule::table("mod_impersonation_sessions")
        ->where("admin_id", $impersonationData["admin_id"])
        ->where("client_id", $impersonationData["client_id"])
        ->where("session_token", hash("sha256", $impersonationData["session_token"]))
        ->where("expires_at", ">", date("Y-m-d H:i:s"))
        ->first();
    
    if (!$session) {
        endImpersonation();
        return $params;
    }
    
    // Add impersonation indicator
    $params["impersonation_warning"] = [
        "admin_name" => getAdminName($impersonationData["admin_id"]),
        "started_at" => $session->started_at,
        "remaining_time" => strtotime($session->expires_at) - time()
    ];
    
    return $params;
});

add_hook("ClientAreaPrimarySidebar", 1, function(array $params) {
    if (!isImpersonating()) {
        return $params;
    }
    
    $impersonationData = $_SESSION["impersonating"];
    
    $params["primarySidebar"]->addItem(
        "Impersonation Active",
        '<div class="impersonation-banner">
            <i class="fa fa-eye"></i>
            <p>Viewing as client</p>
            <a href="end-impersonation.php" class="btn btn-danger btn-sm">
                End Impersonation
            </a>
         </div>',
        "impersonation-indicator"
    );
    
    return $params;
});
```

### End Impersonation

```php
add_hook("AdminImpersonateEnd", 1, function(array $params) {
    if (!isImpersonating()) {
        return ["error" => "Not currently impersonating."];
    }
    
    $impersonationData = $_SESSION["impersonating"];
    
    // Get session for logging
    $session = Capsule::table("mod_impersonation_sessions")
        ->where("admin_id", $impersonationData["admin_id"])
        ->where("client_id", $impersonationData["client_id"])
        ->where("session_token", hash("sha256", $impersonationData["session_token"]))
        ->first();
    
    if ($session) {
        // Log end with actions summary
        Capsule::table("mod_impersonation_log")->insert([
            "admin_id" => $impersonationData["admin_id"],
            "client_id" => $impersonationData["client_id"],
            "action" => "end",
            "duration_seconds" => strtotime("now") - strtotime($session->started_at),
            "actions_count" => count(json_decode($session->actions_performed, true) ?? []),
            "ip_address" => $_SERVER["REMOTE_ADDR"],
            "performed_at" => date("Y-m-d H:i:s")
        ]);
        
        // Mark session as ended
        Capsule::table("mod_impersonation_sessions")
            ->where("id", $session->id)
            ->update([
                "ended_at" => date("Y-m-d H:i:s"),
                "ended_by" => $params["ended_by"] ?? "admin"
            ]);
    }
    
    // Clear session
    unset($_SESSION["impersonating"]);
    
    return ["success" => true];
});
```

### Track Impersonation Actions

```php
add_hook("PreClientUpdate", 1, function(array $params) {
    if (!isImpersonating()) {
        return $params;
    }
    
    trackImpersonationAction("client_update", $params);
    return $params;
});

add_hook("AfterModuleCreate", 1, function(array $params) {
    if (!isImpersonating()) {
        return $params;
    }
    
    trackImpersonationAction("service_create", $params);
    return $params;
});

function trackImpersonationAction(string $action, array $data): void
{
    if (!isImpersonating()) {
        return;
    }
    
    $impersonationData = $_SESSION["impersonating"];
    
    $session = Capsule::table("mod_impersonation_sessions")
        ->where("admin_id", $impersonationData["admin_id"])
        ->where("client_id", $impersonationData["client_id"])
        ->where("session_token", hash("sha256", $impersonationData["session_token"]))
        ->first();
    
    if ($session) {
        $actions = json_decode($session->actions_performed, true) ?? [];
        $actions[] = [
            "action" => $action,
            "data" => sanitizeActionData($data),
            "timestamp" => date("Y-m-d H:i:s")
        ];
        
        Capsule::table("mod_impersonation_sessions")
            ->where("id", $session->id)
            ->update(["actions_performed" => json_encode($actions)]);
    }
}
```

## Impersonation Template

```smarty
<!-- /templates/admin_impersonation_bar.tpl -->
{if $impersonation_warning}
<div class="impersonation-warning-bar">
    <div class="container">
        <div class="warning-content">
            <i class="fa fa-eye"></i>
            <span>
                <strong>Impersonating:</strong> 
                Client Account 
                <a href="clientssummary.php?userid={$client_id}">
                    #{$client_id}
                </a>
            </span>
            <span class="admin-info">
                Started by: {$impersonation_warning.admin_name}
            </span>
            <span class="time-info">
                Time remaining: <span class="countdown" 
                    data-expires="{$impersonation_warning.expires_at}">
                    {$impersonation_warning.remaining_time}
                </span>
            </span>
        </div>
        <div class="warning-actions">
            <a href="clientssummary.php?userid={$client_id}&end_impersonation=1" 
               class="btn btn-danger btn-sm">
                <i class="fa fa-stop-circle"></i>
                End Impersonation
            </a>
        </div>
    </div>
</div>
{/if}
```

## Best Practices

1. **Permission Control**: Require specific admin permission
2. **Rate Limiting**: Limit impersonation frequency
3. **Session Timeout**: Auto-expire after short duration
4. **Action Logging**: Track all changes made during impersonation
5. **Visual Indicator**: Clear warning in client area
6. **Audit Trail**: Complete log of impersonation events
7. **Manual End**: Allow ending impersonation at any time
8. **Email Notification**: Optionally notify client of impersonation
9. **Security**: Verify session token on each request
