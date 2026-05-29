# WHMCS Premium Domains

## Concept
Premium domain handling and pricing.

## Code
```php
<?php
class PremiumDomains {
    public static function isPremiumDomain($domain) {
        return in_array(getTLD($domain), getPremiumTLDs());
    }
    
    public static function getPremiumPrice($domain) {
        return getRegistrarPremiumPrice($domain);
    }
}
```
