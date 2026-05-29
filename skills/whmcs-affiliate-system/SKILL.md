# WHMCS Affiliate System

## Concept Explanation

An affiliate system tracks marketing partners who drive customers to your business and pays them commissions. Unlike simple referrals, affiliates typically have formal agreements, tracking pixels, sub-affiliates, and recurring commissions.

### Affiliate Features

- **Unique Tracking**: Affiliate IDs in URLs/cookies
- **Commission Tiers**: Different rates per affiliate
- **Recurring Commissions**: Ongoing payments for subscriptions
- **Sub-Affiliates**: Multi-level affiliate networks
- **Payout Management**: Automated or manual payouts

## Code Patterns

```php
<?php
// includes/AffiliateManager.php

class AffiliateManager {
    
    public static function trackAffiliateClick($affiliateId, $source = '') {
        insert_query('tblaffiliateclicks', [
            'affiliate_id' => $affiliateId,
            'ip' => $_SERVER['REMOTE_ADDR'],
            'referer' => $_SERVER['HTTP_REFERER'] ?? '',
            'source' => $source,
            'clicked_at' => date('Y-m-d H:i:s')
        ]);
    }
    
    public static function processAffiliateCommission($orderId, $affiliateId = null) {
        $order = getOrder($orderId);
        
        if (!$affiliateId) {
            $affiliateId = self::getAffiliateForClient($order['userid']);
        }
        
        if (!$affiliateId) return false;
        
        $affiliate = getAffiliates(['id' => $affiliateId]);
        $commissionRate = self::getCommissionRate($affiliate);
        
        $totalCommission = 0;
        foreach ($order['products'] as $product) {
            $commission = $product['amount'] * ($commissionRate / 100);
            
            insert_query('tblaffiliatesaccounts', [
                'affiliate_id' => $affiliateId,
                'relid' => $product['id'],
                'order_id' => $orderId,
                'commission' => $commission,
                'commission_percentage' => $commissionRate,
                'created_at' => date('Y-m-d H:i:s')
            ]);
            
            // Create recurring commission tracking
            self::createRecurringCommission($affiliateId, $product['id'], $commission);
            
            $totalCommission += $commission;
        }
        
        return $totalCommission;
    }
    
    public static function processRecurringCommission($serviceId) {
        $account = getAffiliateAccount($serviceId);
        if (!$account) return false;
        
        $service = getService($serviceId);
        $affiliate = getAffiliates(['id' => $account['affiliate_id']]);
        $rate = $affiliate['commission_percentage'] ?? self::getCommissionRate($affiliate);
        
        $commission = $service['amount'] * ($rate / 100);
        
        insert_query('tblaffiliateshistory', [
            'affiliate_id' => $account['affiliate_id'],
            'relid' => $serviceId,
            'commission' => $commission,
            'description' => 'Recurring commission - ' . date('Y-m'),
            'created_at' => date('Y-m-d H:i:s')
        ]);
        
        return $commission;
    }
    
    public static function processPayoutRequest($affiliateId, $amount) {
        $balance = self::getAffiliateBalance($affiliateId);
        
        if ($amount > $balance) {
            throw new Exception("Requested amount exceeds available balance");
        }
        
        insert_query('tblaffiliatespayouts', [
            'affiliate_id' => $affiliateId,
            'amount' => $amount,
            'status' => 'pending',
            'requested_at' => date('Y-m-d H:i:s')
        ]);
        
        return true;
    }
}

add_hook('OrderPaid', 1, function($vars) {
    $orderId = $vars['order_id'];
    $affiliateId = $_COOKIE['affiliate_id'] ?? null;
    
    if ($affiliateId) {
        AffiliateManager::processAffiliateCommission($orderId, $affiliateId);
    }
});

add_hook('ServiceRenewal', 1, function($vars) {
    AffiliateManager::processRecurringCommission($vars['serviceid']);
});
```

## Implementation Checklist

- [ ] Design commission structure
- [ ] Create affiliate tracking
- [ ] Set up recurring commissions
- [ ] Configure payout thresholds
- [ ] Build affiliate dashboard
