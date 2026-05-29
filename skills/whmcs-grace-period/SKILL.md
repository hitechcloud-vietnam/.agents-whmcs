# WHMCS Grace Period

## Concept Explanation
Grace periods extend payment deadlines before service suspension to protect revenue and maintain customer relationships.

### Grace Period Configuration
- **Duration**: Days until suspension
- **Actions**: Reminders, warnings, limited access
- **Extensions**: Special circumstances

## Code Patterns

```php
<?php
class GracePeriodManager {
    
    const DEFAULT_GRACE_DAYS = 7;
    
    public static function getGracePeriodEnd($invoiceId) {
        $invoice = getInvoice($invoiceId);
        $graceDays = self::getGraceDaysForClient($invoice['userid']);
        return date('Y-m-d', strtotime($invoice['duedate'] . " + {$graceDays} days"));
    }
    
    public static function getGraceDaysForClient($clientId) {
        $client = getClientsDetails($clientId);
        
        if ($client['groupid'] == VIP_GROUP_ID) return 14;
        if (strtotime($client['regdate']) > strtotime('-30 days')) return 10;
        
        return self::DEFAULT_GRACE_DAYS;
    }
    
    public static function isInGracePeriod($invoiceId) {
        $graceEnd = self::getGracePeriodEnd($invoiceId);
        return date('Y-m-d') <= $graceEnd;
    }
}
```
