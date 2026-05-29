# WHMCS Admin IP Restriction

## Overview
Guide for implementing IP-based access restrictions for WHMCS admin area. Covers whitelist/blacklist management and IP validation.

## IP Restriction System

### IP Validation

```php
<?php
// /includes/hooks/admin_ip_restriction.php

function checkAdminIPAccess(?int $adminId = null): bool
{
    $adminId = $adminId ?? $_SESSION["adminid"];
    $ip = $_SERVER["REMOTE_ADDR"];
    
    // Get admin's IP restrictions
    $restrictions = Capsule::table("mod_admin_ip_restrictions")
        ->where("admin_id", $adminId)
        ->where("active", 1)
        ->first();
    
    if (!$restrictions) {
        // No restrictions - allow all
        return true;
    }
    
    // Parse allowed IPs
    $allowedIPs = array_filter(array_map("trim", explode("\n", $restrictions->allowed_ips)));
    
    // Check if current IP matches
    foreach ($allowedIPs as $allowed) {
        if (ipMatchesPattern($ip, $allowed)) {
            return true;
        }
    }
    
    return false;
}

function ipMatchesPattern(string $ip, string $pattern): bool
{
    // Exact match
    if ($ip === $pattern) {
        return true;
    }
    
    // CIDR notation (e.g., 192.168.1.0/24)
    if (strpos($pattern, "/") !== false) {
        return cidrMatch($ip, $pattern);
    }
    
    // Wildcard (e.g., 192.168.*.*)
    if (strpos($pattern, "*") !== false) {
        $regex = "/^" . str_replace(["*", "."], ["[0-9]{1,3}", "\."], $pattern) . "$/";
        return (bool)preg_match($regex, $ip);
    }
    
    return false;
}

function cidrMatch(string $ip, string $cidr): bool
{
    [$subnet, $bits] = explode("/", $cidr);
    
    $ip = ip2long($ip);
    $subnet = ip2long($subnet);
    
    if ($ip === false || $subnet === false) {
        return false;
    }
    
    $mask = -1 << (32 - $bits);
    $subnet &= $mask;
    
    return ($ip & $mask) == $subnet;
}
```

### IP Restriction Hook

```php
add_hook("AdminPreAuthenticate", 1, function(array $params) {
    // Skip for password recovery
    if (isset($_GET["action"]) && $_GET["action"] === "pwreset") {
        return $params;
    }
    
    $adminId = $params["admin_id"] ?? $_SESSION["adminid"];
    
    if ($adminId && !checkAdminIPAccess($adminId)) {
        return [
            "error" => "Access denied from your current location.",
            "ip_address" => $_SERVER["REMOTE_ADDR"]
        ];
    }
    
    return $params;
});
```

### Manage IP Restrictions

```php
function setAdminIPRestrictions(int $adminId, array $allowedIPs): bool
{
    $ipList = implode("\n", $allowedIPs);
    
    Capsule::table("mod_admin_ip_restrictions")->updateOrInsert(
        ["admin_id" => $adminId],
        [
            "allowed_ips" => $ipList,
            "active" => 1,
            "updated_at" => date("Y-m-d H:i:s")
        ]
    );
    
    logAdminActivity("ip_restrictions_updated", $adminId, [
        "allowed_ips" => $allowedIPs
    ]);
    
    return true;
}

function removeAdminIPRestrictions(int $adminId): bool
{
    Capsule::table("mod_admin_ip_restrictions")
        ->where("admin_id", $adminId)
        ->update(["active" => 0]);
    
    logAdminActivity("ip_restrictions_removed", $adminId);
    
    return true;
}
```

## IP Restriction Template

```smarty
<!-- /admin/templates/admin_ip_restrictions.tpl -->
<div class="ip-restriction-container">
    <h2>IP Access Restrictions</h2>
    <p>Restrict admin access to specific IP addresses or ranges.</p>
    
    <form method="post" action="admin_ip_restrictions.php">
        <input type="hidden" name="token" value="{$token}">
        <input type="hidden" name="action" value="save">
        
        <div class="form-group">
            <label>Allowed IP Addresses</label>
            <p class="help-block">
                Enter one IP address or CIDR range per line.
                Examples: 192.168.1.1, 10.0.0.0/24, 192.168.*.*
            </p>
            <textarea name="allowed_ips" class="form-control" rows="10"
                      placeholder="192.168.1.1
10.0.0.0/24
192.168.*.*">{$current_ips}</textarea>
        </div>
        
        <div class="form-group">
            <label>
                <input type="checkbox" name="notify_on_block" value="1"
                       {if $notify_on_block}checked{/if}>
                Notify admin when blocked
            </label>
        </div>
        
        <div class="form-actions">
            <button type="submit" class="btn btn-primary">
                Save Restrictions
            </button>
            <a href="?action=remove" class="btn btn-danger"
               onclick="return confirm('Remove all IP restrictions?');">
                Remove Restrictions
            </a>
        </div>
    </form>
    
    <div class="current-ip-info">
        <h4>Your Current IP Address</h4>
        <code>{$current_ip}</code>
        <p class="help-text">
            If your IP is not in the allowed list, you will be blocked from logging in.
        </p>
    </div>
</div>
```

## Best Practices

1. **Backup Access**: Always maintain an account without IP restrictions
2. **CIDR Ranges**: Support CIDR notation for ranges
3. **Wildcards**: Support wildcards for flexibility
4. **IPv6**: Support IPv6 addresses
5. **Notification**: Notify admin of blocked attempts
6. **Logging**: Log all blocked access attempts
7. **Fail-Safe**: Default to allow if no restrictions set
8. **Timeout**: Temporary allow after failed attempts
