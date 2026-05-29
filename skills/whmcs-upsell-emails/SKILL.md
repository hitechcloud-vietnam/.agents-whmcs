# WHMCS Upsell Emails

## Concept
Automated upgrade notification campaigns.

## Code
```php
<?php
class UpsellEmails {
    public static function sendUpgradeOffer($clientId, $serviceId) {
        $service = getService($serviceId);
        $upsell = getUpgradeOffer($service);
        
        if ($upsell) {
            send_email("upgrade_offer", $clientId, ["service" => $service, "offer" => $upsell]);
        }
    }
}
```
