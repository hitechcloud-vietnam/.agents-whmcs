# WHMCS Feedback Collection

## Concept
Customer Satisfaction (CSAT) surveys after ticket resolution.

## Code
```php
<?php
class FeedbackCollector {
    public static function sendSurvey($ticketId) {
        $ticket = getTicket($ticketId);
        
        insert_query("tblsurveys", [
            "ticket_id" => $ticketId,
            "client_id" => $ticket["userid"],
            "sent_at" => date("Y-m-d H:i:s"),
            "expires_at" => date("Y-m-d H:i:s", strtotime("+7 days"))
        ]);
        
        send_email("csat_survey", $ticket["userid"], [
            "ticket_id" => $ticketId,
            "survey_link" => "surveys.php?id=" . $ticketId
        ]);
    }
    
    public static function recordResponse($ticketId, $rating, $comments = "") {
        update_query("tbltickets", ["survey_response" => $rating], ["id" => $ticketId]);
    }
}

add_hook("TicketStatusChange", 1, function($vars) {
    if ($vars["new_status"] == "Resolved") {
        FeedbackCollector::sendSurvey($vars["ticketid"]);
    }
});
```
