# WHMCS DNS Management

## Concept
DNS record management for domains.

## Code
```php
<?php
class DNSManager {
    public static function addRecord($domain, $type, $name, $value, $ttl = 3600) {
        return localAPI("DomainDNS", ["domainid" => $domain, "action" => "add", 
            "type" => $type, "name" => $name, "value" => $value, "ttl" => $ttl]);
    }
    
    public static function getRecords($domainId) {
        return localAPI("GetDomainDNS", ["domainid" => $domainId]);
    }
}
```
