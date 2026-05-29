# WHMCS Ticket Escalation

## Concept
Hierarchical escalation of unresolved or urgent tickets.

## Code
```php
<?php
class TicketEscalation {
    public static function checkEscalations() {
        $tickets = full_query("SELECT * FROM tbltickets WHERE status NOT IN (Closed) AND escalate = 0");
        
        foreach ($tickets as $ticket) {
            $hoursOpen = self::getHoursOpen($ticket["id"]);
            $escalation = self::getEscalationRule($ticket["priority"]);
            
            if ($hoursOpen >= $escalation["hours"]) {
                self::escalateTicket($ticket["id"], $escalation);
            }
        }
    }
    
    public static function escalateTicket($ticketId, $escalation) {
        update_query("tbltickets", [
            "priority" => $escalation["new_priority"],
            "assignedto" => $escalation["new_assignee"],
            "escalate" => 1
        ], ["id" => $ticketId]);
        
        send_notification("ticket_escalated", $ticketId);
    }
}

add_hook("HourlyCronJob", 1, function() { TicketEscalation::checkEscalations(); });
```
