# WHMCS Email Automation

## Concept
Automated email sequences based on customer lifecycle.

## Code
```php
<?php
class EmailAutomation {
    public static function addToSequence($clientId, $sequenceId) {
        $sequence = getSequence($sequenceId);
        
        foreach ($sequence["emails"] as $step) {
            insert_query("tbl_email_queue", [
                "client_id" => $clientId,
                "template" => $step["template"],
                "send_at" => date("Y-m-d H:i:s", strtotime("+" . $step["delay"] . " days")),
                "sequence_id" => $sequenceId
            ]);
        }
    }
    
    public static function processQueue() {
        $pending = full_query("SELECT * FROM tbl_email_queue WHERE send_at <= NOW() AND sent = 0");
        
        foreach ($pending as $email) {
            send_email($email["template"], $email["client_id"]);
            update_query("tbl_email_queue", ["sent" => 1, "sent_at" => now()], ["id" => $email["id"]]);
        }
    }
}

add_hook("DailyCronJob", 1, function() { EmailAutomation::processQueue(); });
```
