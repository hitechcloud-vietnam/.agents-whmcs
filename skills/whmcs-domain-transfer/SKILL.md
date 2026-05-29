# WHMCS Domain Transfer

## Concept
Domain transfer workflow and management.

## Code
```php
<?php
class DomainTransfer {
    public static function initiateTransfer($domain, $authCode) {
        return localAPI("TransferDomain", ["domain" => $domain, "eppcode" => $authCode]);
    }
    
    public static function checkTransferStatus($domain) {
        return localAPI("GetDomainTransferStatus", ["domain" => $domain]);
    }
}
```
