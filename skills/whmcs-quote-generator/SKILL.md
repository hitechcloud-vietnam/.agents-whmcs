# WHMCS Quote Generator

## Concept
Quote builder system.

## Code
```php
<?php
class QuoteGenerator {
    public static function createQuote($clientId, $products, $validityDays = 30) {
        $quoteId = insert_query("tblquotes", [
            "userid" => $clientId, "validuntil" => date("Y-m-d", strtotime("+" . $validityDays . " days")),
            "status" => "Draft", "created_at" => now()
        ]);
        
        foreach ($products as $product) {
            insert_query("tblquoteitems", ["quoteid" => $quoteId, "productid" => $product["id"], "qty" => $product["qty"]]);
        }
        
        return $quoteId;
    }
}
```
