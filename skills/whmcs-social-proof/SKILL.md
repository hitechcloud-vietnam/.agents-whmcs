# WHMCS Social Proof

## Concept
Social proof integration for conversions.

## Code
```php
<?php
class SocialProof {
    public static function getRecentPurchases() {
        return full_query("SELECT c.firstname, p.name, o.date FROM tblorders o 
            JOIN tblclients c ON o.userid = c.id JOIN tblorderproducts op ON o.id = op.orderid 
            JOIN tblproducts p ON op.productid = p.id WHERE o.status = "Active" ORDER BY o.date DESC LIMIT 10");
    }
}
```
