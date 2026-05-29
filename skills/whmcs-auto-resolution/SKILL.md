# WHMCS Auto Resolution

## Concept
Automatic ticket resolution for common issues.

## Code
```php
<?php
class AutoResolver {
    public static function checkAndResolve($ticketId) {
        $ticket = getTicket($ticketId);
        if (self::isAutoResolvable($ticket)) {
            update_query("tbltickets", ["status" => "Resolved"], ["id" => $ticketId]);
        }
    }
}
```
