# WHMCS Upsell Flows

## Concept Explanation
Upselling encourages customers to upgrade to higher-tier products or add premium features at strategic moments.

### Upsell Triggers
- **Post-purchase**: After initial order
- **Checkout**: During checkout process
- **Renewal**: During service renewal
- **Usage-based**: Based on usage patterns

## Code Patterns

```php
<?php
// includes/UpsellManager.php

class UpsellManager {
    
    public static function getUpsellOffer($clientId, $context = '') {
        $services = getClientServices($clientId);
        $recommendations = [];
        
        foreach ($services as $service) {
            $upsell = self::findUpsellForService($service);
            if ($upsell) {
                $recommendations[] = $upsell;
            }
        }
        
        return $recommendations;
    }
    
    public static function findUpsellForService($service) {
        $currentPlan = getProduct($service['packageid']);
        $upsellProducts = getRelatedUpsellProducts($currentPlan['id']);
        
        foreach ($upsellProducts as $upsell) {
            $savings = self::calculateUpgradeSavings($service, $upsell);
            
            if ($savings) {
                return [
                    'current_product' => $currentPlan,
                    'upsell_product' => $upsell,
                    'savings' => $savings
                ];
            }
        }
        
        return null;
    }
    
    public static function calculateUpgradeSavings($service, $upsell) {
        $monthlyCurrent = $service['amount'];
        $monthlyNew = $upsell['monthly'];
        
        if ($monthlyNew <= $monthlyCurrent) return null;
        
        $savings = ($monthlyCurrent * 12) - ($monthlyNew * 10);
        
        return [
            'monthly_difference' => $monthlyNew - $monthlyCurrent,
            'free_months' => 2,
            'total_savings' => $savings
        ];
    }
}

add_hook('OrderCompletePage', 1, function($vars) {
    $upsells = UpsellManager::getUpsellOffer($vars['user_id'], 'post_purchase');
    return ['upsell_offers' => $upsells];
});
```
