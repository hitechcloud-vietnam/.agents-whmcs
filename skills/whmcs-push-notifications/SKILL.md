# WHMCS Push Notifications

## Concept
Browser/app push notifications for real-time updates.

## Code
```php
<?php
class PushNotifier {
    public static function sendPush($userId, $title, $body) {
        $tokens = getUserPushTokens($userId);
        foreach ($tokens as $token) {
            sendPushNotification($token, $title, $body);
        }
    }
}
```
