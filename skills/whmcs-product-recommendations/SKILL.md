# WHMCS Product Recommendations

## Concept
Recommendation engine for related products.

## Code
```php
<?php
class ProductRecommendations {
    public static function getRecommendations($clientId, $limit = 5) {
        $purchased = getClientProducts($clientId);
        $viewed = getRecentlyViewed($clientId);
        
        return array_merge(
            getRelatedProducts($purchased),
            getFrequentlyBoughtTogether($purchased)
        );
    }
}
```
