# WHMCS Domain Renewal

## Concept
Automated domain renewal processing.

## Code
```php
<?php
class DomainRenewal {
    public static function processRenewal($domainId) {
        $domain = getDomain($domainId);
        return localAPI("RenewDomain", ["domainid" => $domainId, "years" => 1]);
    }
}
add_hook("DailyCronJob", 1, function() {
    $expiring = getDomainsExpiringSoon(30);
    foreach ($expiring as $domain) {
        DomainRenewal::processRenewal($domain["id"]);
    }
});
```
