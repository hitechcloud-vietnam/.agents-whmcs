# WHMCS WHOIS Privacy

## Concept
WHOIS privacy protection management.

## Code
```php
<?php
class WHOISPrivacy {
    public static function enablePrivacy($domainId) {
        return localAPI("DomainWHOISPrivacy", ["domainid" => $domainId, "enable" => true]);
    }
    
    public static function getPrivacyStatus($domainId) {
        return localAPI("GetDomainWHOISPrivacy", ["domainid" => $domainId]);
    }
}
```
