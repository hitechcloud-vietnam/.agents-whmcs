# WHMCS Registrant Transfer

## Concept
Registrant contact update workflow.

## Code
```php
<?php
class RegistrantTransfer {
    public static function requestTransfer($domainId, $newRegistrant) {
        return localAPI("UpdateDomainRegistrant", ["domainid" => $domainId, "new_registrant" => $newRegistrant]);
    }
}
```
