# WHMCS Domain Registration

## Concept
Automated domain registration through registrars.

## Code
```php
<?php
class DomainRegistration {
    public static function registerDomain($domain, $registrar, $years = 1) {
        $params = ["domain" => $domain, "years" => $years];
        return localAPI("RegisterDomain", $params);
    }
}
```
