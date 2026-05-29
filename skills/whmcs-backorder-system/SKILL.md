# WHMCS Backorder System

## Concept
Domain backordering for expired domains.

## Code
```php
<?php
class BackorderSystem {
    public static function placeBackorder($domain, $clientId) {
        return insert_query("tbl_domain_backorders", [
            "domain" => $domain, "client_id" => $clientId, "status" => "pending"
        ]);
    }
}
```
