# WHMCS Ticket Merge

## Concept
Consolidating multiple tickets into a single unified ticket.

## Code
```php
<?php
class TicketMerger {
    public static function mergeTickets($primaryId, $mergeIds) {
        foreach ($mergeIds as $ticketId) {
            // Move all replies to primary
            full_query("UPDATE tblticketreplies SET ticketid = " . (int)$primaryId . " WHERE ticketid = " . (int)$ticketId);
            
            // Log merge action
            insert_query("tblticketlog", [
                "ticketid" => $ticketId,
                "action" => "merged",
                "data" => "Merged into ticket #" . $primaryId
            ]);
            
            // Close merged ticket
            update_query("tbltickets", ["status" => "Closed"], ["id" => $ticketId]);
        }
        
        return true;
    }
}
```
