# WHMCS TLD Management

## Concept
Top Level Domain configuration and pricing.

## Code
```php
<?php
class TLDManager {
    public static function addTLD($tld, $pricing) {
        return insert_query("tblpricing", ["type" => "domainregister", "currency" => 1, "tld" => $tld, "price" => $pricing]);
    }
}
```
