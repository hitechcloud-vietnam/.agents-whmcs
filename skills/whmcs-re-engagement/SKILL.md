# WHMCS Re-engagement

## Concept
Winning back inactive customers.

## Code
```php
<?php
class ReEngagement {
    public static function identifyInactiveClients($daysInactive = 90) {
        return full_query("SELECT * FROM tblclients WHERE last_login < DATE_SUB(NOW(), INTERVAL " . (int)$daysInactive . " DAY)");
    }
    
    public static function sendReEngagementCampaign($clientId) {
        $client = getClientsDetails($clientId);
        send_email("reengagement_campaign", $clientId, ["name" => $client["firstname"]]);
    }
}
```
