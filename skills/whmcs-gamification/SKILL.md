# WHMCS Gamification

## Concept
Gamification elements for customer engagement.

## Code
```php
<?php
class Gamification {
    public static function awardBadge($clientId, $badge) {
        insert_query("tbl_client_badges", ["client_id" => $clientId, "badge" => $badge, "awarded_at" => now()]);
    }
    
    public static function getClientAchievements($clientId) {
        return full_query("SELECT * FROM tbl_client_badges WHERE client_id = ?", [$clientId]);
    }
}
```
