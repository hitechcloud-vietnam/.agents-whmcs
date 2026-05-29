# WHMCS Domain Forwarding

## Concept
Domain forwarding configuration.

## Code
```php
<?php
class DomainForwarding {
    public static function setForwarding($domainId, $targetUrl, $type = "301") {
        update_query("tbldomains", ["forwarding" => $targetUrl, "forward_type" => $type], ["id" => $domainId]);
    }
}
```
