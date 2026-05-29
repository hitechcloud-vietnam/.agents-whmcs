# WHMCS DNSSEC Management

## Concept
DNSSEC setup and management.

## Code
```php
<?php
class DNSSECManager {
    public static function enableDNSSEC($domainId, $keys) {
        return localAPI("DomainDNSSEC", ["domainid" => $domainId, "enable" => true, "keys" => $keys]);
    }
    
    public static function getDSRecords($domainId) {
        return localAPI("GetDomainDSRecords", ["domainid" => $domainId]);
    }
}
```
