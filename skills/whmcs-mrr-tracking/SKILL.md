# WHMCS MRR Tracking

## Concept
Monthly Recurring Revenue monitoring for subscription metrics.

## Code
```php
<?php
class MRRTracker {
    public static function calculateMRR() {
        return full_query("SELECT SUM(CASE WHEN billingcycle = "Monthly" THEN amount WHEN billingcycle = "Annual" THEN amount/12 ELSE amount END) as mrr FROM tblhosting WHERE domain != """);
    }
    
    public static function trackMRRChange() {
        $current = self::calculateMRR();
        $previous = getSavedMRR(date("Y-m-01", strtotime("-1 month")));
        
        return ["current" => $current, "previous" => $previous, "growth" => $current - $previous];
    }
}
```
