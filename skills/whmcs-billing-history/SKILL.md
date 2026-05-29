# WHMCS Billing History

## Concept Explanation
Billing history tracking provides comprehensive records of all financial transactions for auditing and reporting.

## Code Patterns

```php
<?php
class BillingHistory {
    public static function getClientBillingHistory($clientId, $limit = 50) {
        return full_query("SELECT * FROM tblinvoices WHERE userid = ? ORDER BY date DESC LIMIT ?", [$clientId, $limit]);
    }
}
```
