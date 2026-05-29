# WHMCS Downsell Handling

## Concept Explanation
Downsell handling manages customer requests to downgrade services with prorated credits and smooth transitions.

### Downsell Considerations
- **Credit Calculation**: Prorated refund for unused time
- **Data Migration**: Handling reduced resources
- **Feature Limitations**: Communicate new restrictions
- **Retention Offers**: Attempt to retain at current tier

## Code Patterns

```php
<?php
// hooks/downsell_handler.php

add_hook('ServiceDowngradeRequest', 1, function($vars) {
    $serviceId = $vars['serviceid'];
    $newProductId = $vars['new_product_id'];
    
    $service = getService($serviceId);
    $newProduct = getProduct($newProductId);
    
    // Calculate proration credit
    $proration = ProrationCalculator::calculateUnusedCredit($serviceId);
    
    // Apply retention offer if applicable
    if ($proration['credit_amount'] > 10) {
        $retentionOffer = getRetentionOffer($service, $proration);
        
        if ($retentionOffer) {
            return [
                'action' => 'show_retention',
                'offer' => $retentionOffer,
                'original_action' => 'downgrade'
            ];
        }
    }
    
    return [
        'action' => 'proceed',
        'credit' => $proration['credit_amount'],
        'new_monthly' => $newProduct['monthly']
    ];
});
```
