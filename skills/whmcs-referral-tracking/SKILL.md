# WHMCS Referral Tracking

## Concept
Referral link and conversion tracking.

## Code
```php
<?php
class ReferralTracking {
    public static function trackReferral($affiliateId, $source) {
        insert_query("tbl_referral_tracking", [
            "affiliate_id" => $affiliateId, "source" => $source,
            "ip_address" => $_SERVER["REMOTE_ADDR"], "clicked_at" => now()
        ]);
    }
    
    public static function convertReferral($affiliateId, $clientId) {
        update_query("tbl_referral_tracking", ["converted" => 1, "converted_at" => now()], 
            ["affiliate_id" => $affiliateId]);
        awardAffiliateCommission($affiliateId, $clientId);
    }
}
```
