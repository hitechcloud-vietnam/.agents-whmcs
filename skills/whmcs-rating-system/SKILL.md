# WHMCS Rating System

## Concept
Multi-dimensional rating tracking.

## Code
```php
<?php
class RatingSystem {
    public static function submitRating($entityType, $entityId, $clientId, $ratings) {
        foreach ($ratings as $dimension => $rating) {
            insert_query("tbl_ratings", [
                "entity_type" => $entityType, "entity_id" => $entityId,
                "client_id" => $clientId, "dimension" => $dimension,
                "rating" => $rating, "created_at" => now()
            ]);
        }
    }
}
```
