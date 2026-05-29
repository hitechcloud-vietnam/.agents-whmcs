# WHMCS Referral Program

## Concept Explanation

Referral programs incentivize existing customers to recommend services to others. WHMCS tracks referrals, credits referrers when new customers sign up, and manages referral payouts. Effective referral programs reduce customer acquisition costs.

### Referral Components

- **Referral Tracking**: Unique codes/links per customer
- **Signup Rewards**: Credit for successful referrals
- **Purchase Rewards**: Commission on referral purchases
- **Tiered Rewards**: Increased rewards for more referrals
- **Payout Processing**: Commission withdrawal requests

## Code Patterns

```php
<?php
// includes/ReferralManager.php

class ReferralManager {
    
    public static function trackReferral($referrerId, $referredEmail) {
        // Check if referral already exists
        $existing = full_query("SELECT id FROM tblreferrals WHERE 
            referrer_id = " . (int)$referrerId . " AND referred_email = '" . e($referredEmail) . "'");
        
        if (mysql_num_rows($existing) == 0) {
            insert_query('tblreferrals', [
                'referrer_id' => $referrerId,
                'referred_email' => $referredEmail,
                'referred_id' => 0,
                'status' => 'pending',
                'created_at' => date('Y-m-d H:i:s')
            ]);
        }
    }
    
    public static function processSignupReward($referralId, $referredClientId) {
        $referral = getReferral($referralId);
        $referrer = getClientsDetails($referral['referrer_id']);
        
        // Get reward amount based on tier
        $tier = self::getReferrerTier($referral['referrer_id']);
        $reward = self::getSignupReward($tier);
        
        // Apply credit to referrer
        \WHMCS\Credit\CreditManager::addCredit(
            $referral['referrer_id'],
            $reward,
            "Referral signup bonus for " . getClientName($referredClientId),
            'referral_signup'
        );
        
        update_query('tblreferrals', [
            'referred_id' => $referredClientId,
            'status' => 'completed',
            'reward_amount' => $reward,
            'completed_at' => date('Y-m-d H:i:s')
        ], ['id' => $referralId]);
        
        return $reward;
    }
    
    public static function getReferrerTier($clientId) {
        $count = full_query("SELECT COUNT(*) as total FROM tblreferrals WHERE 
            referrer_id = " . (int)$clientId . " AND status = 'completed'");
        $row = mysql_fetch_array($count);
        
        if ($row['total'] >= 50) return 'diamond';
        if ($row['total'] >= 20) return 'gold';
        if ($row['total'] >= 5) return 'silver';
        return 'bronze';
    }
    
    public static function getSignupReward($tier) {
        $rewards = ['bronze' => 5, 'silver' => 10, 'gold' => 15, 'diamond' => 25];
        return $rewards[$tier] ?? 5;
    }
}

add_hook('ClientSignup', 1, function($vars) {
    $clientId = $vars['client_id'];
    $email = $vars['email'];
    
    // Find matching referral
    $referral = full_query("SELECT * FROM tblreferrals WHERE 
        referred_email = '" . e($email) . "' AND status = 'pending'");
    
    if ($ref = mysql_fetch_array($referral)) {
        ReferralManager::processSignupReward($ref['id'], $clientId);
    }
});
```

## Implementation Checklist

- [ ] Design referral reward structure
- [ ] Create referral tracking system
- [ ] Set up tiered rewards
- [ ] Configure payout system
- [ ] Test referral workflow
