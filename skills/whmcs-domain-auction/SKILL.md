# WHMCS Domain Auction

## Concept
Domain auction system integration.

## Code
```php
<?php
class DomainAuction {
    public static function listDomain($domainId, $startingBid, $reserve = 0) {
        return insert_query("tbl_domain_auctions", [
            "domain_id" => $domainId, "starting_bid" => $startingBid,
            "reserve_price" => $reserve, "status" => "active"
        ]);
    }
}
```
