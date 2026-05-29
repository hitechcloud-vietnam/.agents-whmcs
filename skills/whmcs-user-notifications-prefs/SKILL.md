# WHMCS User Notification Preferences

## Overview
Guide for implementing granular notification preferences in WHMCS. Covers preference management, channel selection, and subscription handling.

## Notification Preferences System

### Define Notification Types

```php
<?php
// /includes/hooks/notification_prefs.php

define("NOTIFICATION_TYPES", [
    "invoice_generated" => [
        "name" => "Invoice Generated",
        "description" => "When a new invoice is created",
        "channels" => ["email", "sms", "push"],
        "default" => ["email"]
    ],
    "payment_received" => [
        "name" => "Payment Received",
        "description" => "When a payment is processed",
        "channels" => ["email", "sms", "push"],
        "default" => ["email", "sms"]
    ],
    "service_expiring" => [
        "name" => "Service Expiring Soon",
        "description" => "When a service is about to expire",
        "channels" => ["email", "sms"],
        "default" => ["email"]
    ],
    "ticket_response" => [
        "name" => "Ticket Response",
        "description" => "When you receive a reply to a ticket",
        "channels" => ["email", "push"],
        "default" => ["email"]
    ],
    "account_security" => [
        "name" => "Account Security Alerts",
        "description" => "Login attempts, password changes, etc.",
        "channels" => ["email", "sms"],
        "default" => ["email", "sms"],
        "required" => ["email"] // Cannot be disabled
    ],
    "promotions" => [
        "name" => "Promotions & Offers",
        "description" => "Special offers and promotional emails",
        "channels" => ["email"],
        "default" => []
    ],
]);
```

### Save Preferences

```php
add_hook("SaveNotificationPreferences", 1, function(array $params) {
    $userId = $params["user_id"];
    $preferences = $params["preferences"];
    
    foreach ($preferences as $type => $channels) {
        // Get notification type info
        $typeInfo = NOTIFICATION_TYPES[$type] ?? null;
        if (!$typeInfo) continue;
        
        // Ensure required channels are always included
        if (!empty($typeInfo["required"])) {
            $channels = array_unique(array_merge($channels, $typeInfo["required"]));
        }
        
        // Delete existing preferences
        Capsule::table("mod_notification_preferences")
            ->where("user_id", $userId)
            ->where("notification_type", $type)
            ->delete();
        
        // Insert new preferences
        foreach ($channels as $channel) {
            if (in_array($channel, $typeInfo["channels"])) {
                Capsule::table("mod_notification_preferences")->insert([
                    "user_id" => $userId,
                    "notification_type" => $type,
                    "channel" => $channel,
                    "enabled" => 1
                ]);
            }
        }
    }
    
    // Log change
    logUserActivity($userId, "notification_prefs_updated", "account");
    
    return ["success" => true];
});
```

### Get User Preferences

```php
function getNotificationPreferences(int $userId): array
{
    $preferences = [];
    
    // Get saved preferences
    $saved = Capsule::table("mod_notification_preferences")
        ->where("user_id", $userId)
        ->where("enabled", 1)
        ->get();
    
    foreach ($saved as $pref) {
        $preferences[$pref->notification_type][] = $pref->channel;
    }
    
    // Fill in defaults for missing types
    foreach (NOTIFICATION_TYPES as $type => $info) {
        if (!isset($preferences[$type])) {
            $preferences[$type] = $info["default"];
        }
    }
    
    return $preferences;
}

function isNotificationEnabled(
    int $userId,
    string $type,
    string $channel
): bool {
    // Check required notifications first
    $typeInfo = NOTIFICATION_TYPES[$type] ?? [];
    if (!empty($typeInfo["required"]) && in_array($channel, $typeInfo["required"])) {
        return true;
    }
    
    // Check user preferences
    $pref = Capsule::table("mod_notification_preferences")
        ->where("user_id", $userId)
        ->where("notification_type", $type)
        ->where("channel", $channel)
        ->where("enabled", 1)
        ->first();
    
    return (bool)$pref;
}
```

### Send Notification with Preferences

```php
function sendUserNotification(
    int $userId,
    string $type,
    string $channel,
    array $data
): bool {
    if (!isNotificationEnabled($userId, $type, $channel)) {
        return false;
    }
    
    $client = Capsule::table("tblclients")->where("id", $userId)->first();
    
    switch ($channel) {
        case "email":
            return send_email($data["template"], $userId, $data["vars"] ?? []);
            
        case "sms":
            return sendSMSNotification($userId, $data["message"]);
            
        case "push":
            return sendPushNotification($userId, $data["title"], $data["body"]);
            
        case "webhook":
            return sendWebhookNotification($userId, $data);
    }
    
    return false;
}

function shouldSendNotification(int $userId, string $type): array
{
    $prefs = getNotificationPreferences($userId);
    return $prefs[$type] ?? [];
}
```

### Hook Integration

```php
add_hook("InvoicePaid", 1, function(array $params) {
    $invoice = Capsule::table("tblinvoices")->where("id", $params["invoiceid"])->first();
    
    $channels = shouldSendNotification($invoice->userid, "payment_received");
    
    foreach ($channels as $channel) {
        sendUserNotification($invoice->userid, "payment_received", $channel, [
            "template" => "Payment Confirmation",
            "vars" => [
                "invoice_id" => $invoice->id,
                "amount" => $invoice->total
            ]
        ]);
    }
    
    return $params;
});
```

## Preferences Template

```smarty
<!-- /templates/clientarea_notifications.tpl -->
<div class="notification-prefs-container">
    <h2>Notification Preferences</h2>
    <p>Choose how you want to receive notifications about your account.</p>
    
    <form method="post" action="clientarea.php?action=notifications" 
          class="notification-form">
        <input type="hidden" name="token" value="{$token}">
        
        {foreach NOTIFICATION_TYPES as $type => $info}
            <div class="notification-section">
                <div class="notification-header">
                    <h3>{$info.name}</h3>
                    <p>{$info.description}</p>
                </div>
                
                <div class="channel-options">
                    {foreach $info.channels as $channel}
                        <label class="channel-option">
                            <input type="checkbox" 
                                   name="preferences[{$type}][]" 
                                   value="{$channel}"
                                   {if in_array($channel, $user_prefs[$type])}
                                       checked
                                   {/if}
                                   {if $channel in $info.required}
                                       disabled
                                   {/if}>
                            <span class="channel-label">
                                <i class="fa fa-{if $channel eq 'email'}envelope{elseif $channel eq 'sms'}mobile{elseif $channel eq 'push'}bell{elseif $channel eq 'webhook'}cloud{/if}"></i>
                                {$channel|ucfirst}
                            </span>
                            {if $channel in $info.required}
                                <span class="required-badge">Required</span>
                            {/if}
                        </label>
                    {/foreach}
                </div>
            </div>
        {/foreach}
        
        <div class="form-actions">
            <button type="submit" class="btn btn-primary">
                Save Preferences
            </button>
        </div>
    </form>
</div>
```

## Best Practices

1. **Granular Control**: Allow per-notification, per-channel settings
2. **Smart Defaults**: Sensible default settings for new users
3. **Required Notifications**: Some notifications cannot be disabled (security)
4. **Channel Availability**: Only show available channels
5. **Group Integration**: Allow group-level preference defaults
6. **Opt-In Promotions**: Marketing should be opt-in by default
7. **Preview**: Show notification preview when toggling
8. **Audit Trail**: Log preference changes
