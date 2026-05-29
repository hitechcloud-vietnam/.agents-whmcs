# WHMCS Loyalty Points

## Concept Explanation
Loyalty points systems reward customers for purchases and engagement. Points accumulate over time and can be redeemed for discounts, credits, or products.

### Point Types
- **Purchase Points**: Earned per dollar spent
- **Bonus Points**: Special promotions
- **Review Points**: Earned for product reviews
- **Referral Points**: For successful referrals

## Code Patterns

```php
<?php
// includes/LoyaltyPointsManager.php

class LoyaltyPointsManager {
    
    public static function awardPoints($clientId, $points, $reason, $relatedId = 0) {
        insert_query('tbl_loyalty_points', [
            'client_id' => $clientId,
            'points' => $points,
            'type' => 'earned',
            'reason' => $reason,
            'related_id' => $relatedId,
            'created_at' => date('Y-m-d H:i:s')
        ]);
        
        self::updateTotalPoints($clientId);
        return true;
    }
    
    public static function redeemPoints($clientId, $points, $reason) {
        $balance = self::getPointsBalance($clientId);
        
        if ($balance < $points) {
            throw new Exception("Insufficient points balance");
        }
        
        insert_query('tbl_loyalty_points', [
            'client_id' => $clientId,
            'points' => -$points,
            'type' => 'redeemed',
            'reason' => $reason,
            'created_at' => date('Y-m-d H:i:s')
        ]);
        
        self::updateTotalPoints($clientId);
        return true;
    }
    
    public static function getPointsBalance($clientId) {
        $result = full_query("SELECT SUM(CASE WHEN type = 'earned' THEN points ELSE -points END) as balance 
            FROM tbl_loyalty_points WHERE client_id = " . (int)$clientId);
        $row = mysql_fetch_array($result);
        return max(0, (int)($row['balance'] ?? 0));
    }
    
    public static function convertToCredit($clientId, $points) {
        $creditValue = $points / 100; // 100 points = $1
        self::redeemPoints($clientId, $points, "Converted to account credit");
        CreditManager::addCredit($clientId, $creditValue, "Points redemption");
        return $creditValue;
    }
}

add_hook('OrderPaid', 1, function($vars) {
    $order = getOrder($vars['order_id']);
    $points = floor($order['total']); // 1 point per dollar
    
    LoyaltyPointsManager::awardPoints(
        $order['userid'],
        $points,
        "Purchase points for order #" . $vars['order_id'],
        $vars['order_id']
    );
});
```
