# WHMCS Behavior Tracking

## Concept
User behavior tracking for personalization.

## Code
```php
<?php
class BehaviorTracker {
    public static function trackEvent($clientId, $event, $data = []) {
        insert_query("tbl_behavior_events", [
            "client_id" => $clientId, "event" => $event,
            "data" => json_encode($data), "created_at" => date("Y-m-d H:i:s")
        ]);
    }
    
    public static function getClientBehavior($clientId) {
        return full_query("SELECT event, COUNT(*) as count FROM tbl_behavior_events WHERE client_id = ? GROUP BY event", [$clientId]);
    }
}
```
