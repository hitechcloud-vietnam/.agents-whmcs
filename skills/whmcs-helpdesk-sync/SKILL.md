# WHMCS Helpdesk Sync

## Concept
Helpdesk synchronization.

## Code
```php
<?php
class HelpdeskSync {
    public static function syncTicket($ticketId) {
        $ticket = getTicket($ticketId);
        $this->helpdesk->createTicket(["subject" => $ticket["subject"], "body" => $ticket["message"]]);
    }
    
    public static function syncReply($replyId) {
        $reply = getTicketReply($replyId);
        $this->helpdesk->addReply(["ticket_id" => $reply["ticketid"], "body" => $reply["message"]]);
    }
}
```
