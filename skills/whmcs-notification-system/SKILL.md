# WHMCS Notification System

## Overview
Master skill for managing notifications in WHMCS. Covers email notifications, in-app notifications, and push notifications.

## Notification Hooks

```php
<?php
// /includes/hooks/notification_hooks.php

add_hook("InvoicePaid", 1, function(array $params) {
    $clientId = $params["userid"];
    
    sendInAppNotification($clientId, "payment", "Payment received", "Invoice paid successfully");
    
    return true;
});

add_hook("ServiceSuspended", 1, function(array $params) {
    $clientId = $params["userid"];
    
    sendPushNotification($clientId, "Service Suspended", "Your service has been suspended");
    
    return true;
});

add_hook("TicketReply", 1, function(array $params) {
    $clientId = $params["userid"];
    $ticketId = $params["ticketid"];
    
    sendNotification($clientId, "support", "New ticket reply", "Ticket #{$ticketId} has been replied");
    
    return true;
});
```

## Notification Manager

```php
<?php
// /includes/managers/NotificationManager.php

namespace WHMCS\Notifications;

class NotificationManager
{
    public function send(int $clientId, string $type, string $title, string $message, array $data = []): int
    {
        return \Illuminate\Database\Capsule\Manager::table("mod_notifications")
            ->insertGetId([
                "client_id" => $clientId,
                "type" => $type,
                "title" => $title,
                "message" => $message,
                "data" => json_encode($data),
                "read" => 0,
                "created_at" => date("Y-m-d H:i:s"),
            ]);
    }
    
    public function getUnread(int $clientId): array
    {
        return \Illuminate\Database\Capsule\Manager::table("mod_notifications")
            ->where("client_id", $clientId)
            ->where("read", 0)
            ->orderBy("created_at", "desc")
            ->get()
            ->toArray();
    }
    
    public function markAsRead(int $notificationId): void
    {
        \Illuminate\Database\Capsule\Manager::table("mod_notifications")
            ->where("id", $notificationId)
            ->update(["read" => 1, "read_at" => date("Y-m-d H:i:s")]);
    }
    
    public function markAllAsRead(int $clientId): void
    {
        \Illuminate\Database\Capsule\Manager::table("mod_notifications")
            ->where("client_id", $clientId)
            ->where("read", 0)
            ->update(["read" => 1, "read_at" => date("Y-m-d H:i:s")]);
    }
}
```

## Best Practices

1. **Relevance**: Send only relevant notifications
2. **Timing**: Send at appropriate times
3. **Channels**: Use multiple notification channels
4. **Preferences**: Respect user preferences
5. **Batching**: Batch notifications when appropriate
6. **Tracking**: Track notification engagement
7. **Unsubscribe**: Easy unsubscribe options
8. **Testing**: Test all notification types
