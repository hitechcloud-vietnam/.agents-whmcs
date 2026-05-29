# WHMCS Review System

## Concept
Product/service review integration.

## Code
```php
<?php
class ReviewSystem {
    public static function addReview($productId, $clientId, $rating, $review) {
        return insert_query("tbl_reviews", [
            "product_id" => $productId, "client_id" => $clientId,
            "rating" => $rating, "review" => $review, "created_at" => now()
        ]);
    }
    
    public static function getProductRating($productId) {
        return full_query("SELECT AVG(rating) as avg_rating, COUNT(*) as count FROM tbl_reviews WHERE product_id = ?", [$productId]);
    }
}
```
