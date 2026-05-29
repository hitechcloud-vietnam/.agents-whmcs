# WHMCS Pricing Strategies

## Concept Explanation
Dynamic pricing strategies adjust prices based on demand, time, customer segments, and competitive factors.

### Pricing Models
- **Tiered Pricing**: Volume-based discounts
- **Penetration Pricing**: Low initial, increase later
- **Premium Pricing**: Higher for premium features
- **Seasonal Pricing**: Time-based adjustments

## Code Patterns

```php
<?php
// includes/PricingStrategy.php

class PricingStrategy {
    
    public static function calculateTieredPrice($productId, $quantity) {
        $tiers = getPricingTiers($productId);
        $basePrice = getProductPrice($productId);
        
        $tierPrice = $basePrice;
        foreach ($tiers as $tier) {
            if ($quantity >= $tier['min_qty']) {
                $tierPrice = $tier['unit_price'];
            }
        }
        
        return $tierPrice * $quantity;
    }
    
    public static function applyVolumeDiscount($amount, $quantity) {
        $discounts = [10 => 0.05, 25 => 0.10, 50 => 0.15, 100 => 0.20];
        
        foreach ($discounts as $minQty => $discount) {
            if ($quantity >= $minQty) {
                return $amount * (1 - $discount);
            }
        }
        
        return $amount;
    }
    
    public static function applySegmentedPrice($clientId, $basePrice) {
        $client = getClientsDetails($clientId);
        $segments = ['vip' => 0.85, 'enterprise' => 0.90, 'default' => 1.0];
        
        $multiplier = $segments[strtolower($client['groupid'])] ?? 1.0;
        return $basePrice * $multiplier;
    }
}
```
