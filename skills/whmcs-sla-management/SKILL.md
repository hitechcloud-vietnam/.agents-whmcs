# WHMCS SLA Management

## Concept
Service Level Agreement configuration for response/resolution times.

## Code
```php
<?php
class SLAManager {
    public static function applySLA($ticketId, $slaId) {
        $sla = getSLA($slaId);
        
        $responseTime = date("Y-m-d H:i:s", strtotime("+" . $sla["responsetime"] . " minutes"));
        $resolutionTime = date("Y-m-d H:i:s", strtotime("+" . $sla["resolutiontime"] . " hours"));
        
        update_query("tbltickets", [
            "slapriority" => $sla["priority"],
            "firstresponse" => $responseTime,
            "duetime" => $resolutionTime
        ], ["id" => $ticketId]);
    }
    
    public static function checkBreaches() {
        $breached = full_query("SELECT * FROM tbltickets WHERE status NOT IN ("Closed", "Resolved") AND duetime < NOW()");
        
        foreach ($breached as $ticket) {
            send_notification("sla_breach", $ticket);
            logActivity("SLA breach on ticket #" . $ticket["id"]);
        }
    }
}

add_hook("DailyCronJob", 1, function() { SLAManager::checkBreaches(); });
```
