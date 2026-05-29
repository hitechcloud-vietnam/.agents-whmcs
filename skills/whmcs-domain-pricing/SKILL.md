# WHMCS Domain Pricing

## Concept
Dynamic domain pricing management.

## Code
```php
<?php
class DomainPricing {
    public static function getDomainPrice($domain, $tld) {
        $price = getTLDPricing($tld);
        if (isPremiumDomain($domain)) {
            $price *= getPremiumMultiplier();
        }
        return $price;
    }
}
```
