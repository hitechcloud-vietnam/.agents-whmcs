# WHMCS User Device Tracking

## Overview
Guide for implementing device tracking and management for WHMCS users. Covers device detection, trusted devices, and session management.

## Device Tracking System

### Track Device

```php
<?php
// /includes/hooks/device_tracking.php

add_hook("ClientLogin", 1, function(array $params) {
    $userId = $params["userid"];
    $fingerprint = generateDeviceFingerprint();
    
    // Check if device is known
    $existingDevice = Capsule::table("mod_user_devices")
        ->where("user_id", $userId)
        ->where("fingerprint", $fingerprint)
        ->first();
    
    if ($existingDevice) {
        // Update existing device
        Capsule::table("mod_user_devices")
            ->where("id", $existingDevice->id)
            ->update([
                "last_used_at" => date("Y-m-d H:i:s"),
                "last_ip" => $_SERVER["REMOTE_ADDR"],
                "login_count" => $existingDevice->login_count + 1
            ]);
        
        $deviceId = $existingDevice->id;
    } else {
        // New device
        $deviceId = Capsule::table("mod_user_devices")->insertGetId([
            "user_id" => $userId,
            "fingerprint" => $fingerprint,
            "user_agent" => $_SERVER["HTTP_USER_AGENT"] ?? "",
            "ip_address" => $_SERVER["REMOTE_ADDR"],
            "browser" => parseBrowser(),
            "os" => parseOS(),
            "device_type" => detectDeviceType(),
            "first_seen_at" => date("Y-m-d H:i:s"),
            "last_used_at" => date("Y-m-d H:i:s"),
            "login_count" => 1,
            "trusted" => 0,
            "created_at" => date("Y-m-d H:i:s")
        ]);
        
        // Notify user of new device
        sendNewDeviceAlert($userId, $deviceId);
    }
    
    // Check for suspicious activity
    checkDeviceSuspiciousActivity($userId, $deviceId);
    
    return $params;
});

function generateDeviceFingerprint(): string
{
    $components = [
        $_SERVER["HTTP_USER_AGENT"] ?? "",
        $_SERVER["HTTP_ACCEPT_LANGUAGE"] ?? "",
        $_SERVER["HTTP_ACCEPT_ENCODING"] ?? "",
        isset($_SERVER["HTTP_SEC_CH_UA"]) ? $_SERVER["HTTP_SEC_CH_UA"] : "",
        isset($_SERVER["HTTP_SEC_CH_UA_PLATFORM"]) ? $_SERVER["HTTP_SEC_CH_UA_PLATFORM"] : "",
    ];
    
    return hash("sha256", implode("|", $components));
}

function parseBrowser(): string
{
    $ua = $_SERVER["HTTP_USER_AGENT"] ?? "";
    
    if (preg_match("/Edge\/[\d.]+/", $ua)) return "Edge";
    if (preg_match("/OPR\/[\d.]+/", $ua)) return "Opera";
    if (preg_match("/Chrome\/[\d.]+/", $ua)) return "Chrome";
    if (preg_match("/Safari\/[\d.]+/", $ua)) return "Safari";
    if (preg_match("/Firefox\/[\d.]+/", $ua)) return "Firefox";
    if (preg_match("/MSIE [\d.]+/", $ua)) return "Internet Explorer";
    
    return "Unknown";
}

function parseOS(): string
{
    $ua = $_SERVER["HTTP_USER_AGENT"] ?? "";
    
    if (preg_match("/Windows NT 10/", $ua)) return "Windows 10";
    if (preg_match("/Windows NT 6.3/", $ua)) return "Windows 8.1";
    if (preg_match("/Windows/", $ua)) return "Windows";
    if (preg_match("/Mac OS X/", $ua)) return "macOS";
    if (preg_match("/Linux/", $ua)) return "Linux";
    if (preg_match("/Android/", $ua)) return "Android";
    if (preg_match("/iPhone|iPad/", $ua)) return "iOS";
    
    return "Unknown";
}

function detectDeviceType(): string
{
    $ua = $_SERVER["HTTP_USER_AGENT"] ?? "";
    
    if (preg_match("/Mobile|Android|iPhone/", $ua)) return "mobile";
    if (preg_match("/Tablet|iPad/", $ua)) return "tablet";
    
    return "desktop";
}
```

### Trusted Devices

```php
add_hook("TrustDevice", 1, function(array $params) {
    $userId = $params["user_id"];
    $deviceId = $params["device_id"];
    
    // Verify device belongs to user
    $device = Capsule::table("mod_user_devices")
        ->where("id", $deviceId)
        ->where("user_id", $userId)
        ->first();
    
    if (!$device) {
        return ["error" => "Device not found"];
    }
    
    Capsule::table("mod_user_devices")
        ->where("id", $deviceId)
        ->update([
            "trusted" => 1,
            "trusted_at" => date("Y-m-d H:i:s")
        ]);
    
    // Extend session for trusted devices
    Capsule::table("mod_user_sessions")
        ->where("user_id", $userId)
        ->where("device_id", $deviceId)
        ->update(["expires_at" => date("Y-m-d H:i:s", strtotime("+30 days"))]);
    
    logUserActivity($userId, "device_trusted", "security", [
        "device_id" => $deviceId,
        "device_name" => $device->browser . " on " . $device->os
    ]);
    
    return ["success" => true];
});

add_hook("UntrustDevice", 1, function(array $params) {
    $userId = $params["user_id"];
    $deviceId = $params["device_id"];
    
    Capsule::table("mod_user_devices")
        ->where("id", $deviceId)
        ->where("user_id", $userId)
        ->update([
            "trusted" => 0,
            "trusted_at" => null
        ]);
    
    // Terminate sessions on this device
    Capsule::table("mod_user_sessions")
        ->where("user_id", $userId)
        ->where("device_id", $deviceId)
        ->update(["active" => 0, "terminated_at" => date("Y-m-d H:i:s")]);
    
    logUserActivity($userId, "device_untrusted", "security", [
        "device_id" => $deviceId
    ]);
    
    return ["success" => true];
});
```

### Suspicious Activity Detection

```php
function checkDeviceSuspiciousActivity(int $userId, int $deviceId): void
{
    $device = Capsule::table("mod_user_devices")->where("id", $deviceId)->first();
    
    // Check for new location
    $recentLocations = Capsule::table("mod_user_devices")
        ->where("user_id", $userId)
        ->where("id", "!=", $deviceId)
        ->where("last_used_at", ">", date("Y-m-d H:i:s", strtotime("-7 days")))
        ->get()
        ->pluck("ip_address")
        ->toArray();
    
    $currentCountry = getCountryFromIP($_SERVER["REMOTE_ADDR"]);
    
    foreach ($recentLocations as $recentIP) {
        $recentCountry = getCountryFromIP($recentIP);
        
        if ($currentCountry !== $recentCountry) {
            // New country detected
            sendSuspiciousActivityAlert($userId, [
                "type" => "new_country",
                "device" => $device->browser . " on " . $device->os,
                "new_country" => $currentCountry,
                "ip_address" => $_SERVER["REMOTE_ADDR"]
            ]);
            break;
        }
    }
    
    // Check for too many new devices
    $newDevicesCount = Capsule::table("mod_user_devices")
        ->where("user_id", $userId)
        ->where("first_seen_at", ">", date("Y-m-d H:i:s", strtotime("-24 hours")))
        ->count();
    
    if ($newDevicesCount > 5) {
        sendSuspiciousActivityAlert($userId, [
            "type" => "too_many_devices",
            "count" => $newDevicesCount,
            "ip_address" => $_SERVER["REMOTE_ADDR"]
        ]);
    }
}

function sendNewDeviceAlert(int $userId, int $deviceId): void
{
    $device = Capsule::table("mod_user_devices")->where("id", $deviceId)->first();
    
    send_email("NewDeviceLogin", $userId, [
        "device" => $device->browser . " on " . $device->os,
        "ip_address" => $device->ip_address,
        "location" => getLocationFromIP($device->ip_address),
        "time" => date("Y-m-d H:i:s"),
        "trust_link" => "trust-device.php?id=" . $deviceId,
        "device_id" => $deviceId
    ]);
}
```

## Device Management Template

```smarty
<!-- /templates/clientarea_devices.tpl -->
<div class="devices-container">
    <h2>Trusted Devices</h2>
    <p>Manage devices that can access your account without additional verification.</p>
    
    <div class="devices-list">
        {foreach $devices as $device}
            <div class="device-card {if $device->trusted}trusted{/if}">
                <div class="device-icon">
                    <i class="fa fa-{if $device->device_type eq 'mobile'}mobile{elseif $device->device_type eq 'tablet'}tablet{else}desktop{/if}"></i>
                </div>
                
                <div class="device-info">
                    <h3>
                        {$device.browser} on {$device.os}
                        {if $device->trusted}
                            <span class="badge badge-success">Trusted</span>
                        {/if}
                    </h3>
                    <p class="device-meta">
                        <span>Last used: {$device->last_used_at}</span>
                        <span>IP: {$device->last_ip}</span>
                    </p>
                    <p class="device-location">
                        <i class="fa fa-map-marker"></i>
                        {$device->location ?? 'Unknown location'}
                    </p>
                    <p class="device-stats">
                        Used {$device->login_count} times
                    </p>
                </div>
                
                <div class="device-actions">
                    {if !$device->trusted}
                        <a href="?trust={$device->id}" class="btn btn-sm btn-success">
                            Trust Device
                        </a>
                    {/if}
                    <a href="?remove={$device->id}" class="btn btn-sm btn-danger">
                        Remove
                    </a>
                </div>
            </div>
        {/foreach}
    </div>
    
    <div class="remove-all-link">
        <a href="?remove_all=1" class="text-danger">
            Remove All Other Devices
        </a>
    </div>
</div>
```

## Best Practices

1. **Fingerprinting**: Use multiple signals for device identification
2. **Trust System**: Allow users to mark devices as trusted
3. **Location Tracking**: Detect and alert on new locations
4. **Activity Limits**: Flag unusual device activity
5. **Notifications**: Alert users of new device logins
6. **Easy Management**: Allow quick removal of devices
7. **Session Sync**: Extend sessions for trusted devices
8. **Privacy**: Be transparent about tracking scope
