# WHMCS ARR Calculation

## Concept
Annual Recurring Revenue calculation from MRR.

## Code
```php
<?php
class ARRCalculator {
    public static function calculateARR() {
        $mrr = full_query("SELECT SUM(amount) as mrr FROM tblhosting WHERE billingcycle IN ("Monthly", "Quarterly", "Annual")");
        return $mrr["mrr"] * 12;
    }
    
    public static function getARRByPlan() {
        return full_query("SELECT packageid, SUM(amount) * 12 as arr FROM tblhosting GROUP BY packageid");
    }
}
```
