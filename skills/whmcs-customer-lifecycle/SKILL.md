# WHMCS Customer Lifecycle

## Concept
Customer lifecycle stage management.

## Code
```php
<?php
class CustomerLifecycle {
    public static function determineStage($clientId) {
        $orders = getClientOrderCount($clientId);
        $totalSpent = getClientTotalSpent($clientId);
        
        if ($orders == 0) return "prospect";
        if ($orders == 1) return "new_customer";
        if ($totalSpent > 1000) return "loyal";
        if ($totalSpent > 5000) return "vip";
        return "regular";
    }
}
```
