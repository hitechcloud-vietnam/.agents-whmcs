# WHMCS Anniversary Emails

## Concept
Automated birthday and anniversary greetings.

## Code
```php
<?php
class AnniversaryEmails {
    public static function sendAnniversaryEmails() {
        $clients = getClientsWithAnniversaryToday();
        
        foreach ($clients as $client) {
            send_email("anniversary_greeting", $client["id"], [
                "years" => $client["years_customer"]
            ]);
        }
    }
}
add_hook("DailyCronJob", 1, function() {
    AnniversaryEmails::sendAnniversaryEmails();
});
```
