# WHMCS Coupon Generation

## Concept
Dynamic coupon generation system.

## Code
```php
<?php
class CouponGenerator {
    public static function generateCoupon($prefix, $length = 8) {
        $code = $prefix . strtoupper(substr(md5(rand()), 0, $length));
        
        insert_query("tblpromotions", [
            "code" => $code, "type" => "percentage", "value" => 10,
            "uses" => 1, "maxuses" => 1, "expiredate" => date("Y-m-d", strtotime("+30 days"))
        ]);
        
        return $code;
    }
}
```
