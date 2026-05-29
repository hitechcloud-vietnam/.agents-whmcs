# WHMCS Cross-sell Emails

## Concept
Cross-sell product recommendations.

## Code
```php
<?php
class CrossSellEmails {
    public static function sendCrossSellRecommendations($clientId) {
        $purchased = getClientProducts($clientId);
        $recommendations = getProductRecommendations($purchased);
        
        if (!empty($recommendations)) {
            send_email("cross_sell_recommendations", $clientId, ["recommendations" => $recommendations]);
        }
    }
}
```
