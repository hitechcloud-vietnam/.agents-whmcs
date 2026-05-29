# WHMCS Revenue Reporting

## Concept
Comprehensive revenue analytics and financial reporting.

## Code
```php
<?php
class RevenueReporter {
    public static function getMonthlyRevenue($year, $month) {
        return full_query("SELECT SUM(total) as revenue FROM tblinvoices WHERE status = "Paid" AND MONTH(date) = ? AND YEAR(date) = ?", [$month, $year]);
    }
    
    public static function getRevenueByProduct() {
        return full_query("SELECT p.name, SUM(i.amount) as revenue FROM tblinvoiceitems i JOIN tblproducts p ON i.relid = p.id WHERE i.type = "hosting" GROUP BY p.id");
    }
}
```
