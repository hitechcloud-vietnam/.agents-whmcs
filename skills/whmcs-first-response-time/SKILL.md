# WHMCS First Response Time

## Concept
Optimizing first response time for better customer experience.

## Code
```php
<?php
class FRTOptimizer {
    public static function trackFRT($ticketId) {
        $ticket = getTicket($ticketId);
        if ($ticket["firstresponse"]) {
            $frt = getTimeDiff($ticket["created"], $ticket["firstresponse"]);
            update_query("tbltickets", ["frt_hours" => $frt], ["id" => $ticketId]);
        }
    }
}
```
